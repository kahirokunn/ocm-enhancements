# Cluster-Specific Manifest Overrides for ManifestWorkReplicaSet

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

`ManifestWorkReplicaSet` (MWRS) distributes one `manifestWorkTemplate` to every cluster
selected by its `placementRefs`. The payload is copied verbatim: only the `ManifestWork`'s
namespace, labels and annotations differ per cluster. Users who need a small per-cluster
difference — a `nodeSelector` value that matches the local node pool, a pod label that lets
a service mesh distinguish copies — have to abandon MWRS and maintain one `ManifestWork`
per cluster.

This proposal introduces a new namespaced resource, `ManifestOverridePolicy`, which selects
manifests by resource identity and clusters by `ManagedCluster` labels, and applies ordered
patches (JSON Patch, JSON Merge Patch, or strategic merge patch) to the matched manifests as
they are rendered for each cluster. An MWRS lists the policies that apply to it in a new
`spec.overridePolicies` field, so the MWRS spec states exactly what shapes its payload, and
only the listed policies are applied. A platform team can define an override once and have
many workloads reference it.

A companion cluster-scoped resource, `ResourceOverrideSchema`, supplies the list-merge metadata
(`x-kubernetes-list-type` and `x-kubernetes-list-map-keys`) that strategic merge patching needs
for custom resources. A CRD schema already carries this metadata, but the hub often does not
have the CRD, because the type is installed only on the managed clusters. The registered schema
fills that gap, and also overrides parts of a hub CRD schema when needed.

## Motivation

The request originates in [ocm#1175][ocm-1175] and was carried into
[enhancements#198][issue-198]. Two distinct needs appear in those threads, and separating
them explains the shape of this proposal.

**Values that are facts about a cluster.** The application node pool is tagged
`nodepool: apps-a` on `cluster-a` and `nodepool: apps-b` on `cluster-b`. A Deployment's
`nodeSelector` must match whichever one it lands on. This is true of every workload on that
cluster, not just this one. Region-scoped settings — a higher `priorityClassName` in
`region-a` — have the same shape. These are naturally expressed as "for clusters matching
this label, patch this field", which is what this proposal provides.

**Values derived from cluster identity.** Each cluster's copy of a Deployment needs a
distinguishing pod label so Envoy or Istio can route to the copies individually. Nothing
about `cluster-a` says the label should be any particular string, only that it must differ
from `cluster-b`'s. Expressing this with static patches means one override entry per cluster,
which does not survive fleet growth. This need is **out of scope here** and is deferred to a
separate templating enhancement (see [Non-Goals](#non-goals)); the API below is shaped so
that templated patch values can be added without a breaking change.

### Current Limitations

Verified against `open-cluster-management-io/api` and `open-cluster-management-io/ocm` at
`main`:

- `ManifestWorkReplicaSetSpec` (`work/v1alpha1/types_manifestworkreplicaset.go`) has three
  fields — `manifestWorkTemplate`, `placementRefs`, `cascadeDeletionPolicy`. There is no
  customization surface.
- Delivery is a literal struct copy: `Spec: mwrSet.Spec.ManifestWorkTemplate` in
  `manifestworkreplicaset_deploy_reconcile.go`.
- The MWRS controller is wired with a `PlacementDecisionInformer` only
  (`manifestworkreplicaset_controller.go`), and `ClusterDecision` carries just `clusterName`,
  `reason` and `score`. No `ManagedCluster` attribute is reachable from the controller today.

### Goals

- Let users keep one shared `manifestWorkTemplate` in MWRS and declare per-cluster
  differences separately.
- Select target clusters by `ManagedCluster` label, and target manifests by resource identity.
- Support JSON Patch (RFC 6902), JSON Merge Patch (RFC 7386) and Kubernetes strategic merge
  patch, including for custom resources.
- Make the applied result deterministic: the same inputs always produce the same rendered
  `ManifestWork`.
- Make override failures visible per cluster, and surface them to the MWRS rollout machinery
  rather than silently stalling.
- Make it possible to tell which policy changed a given `ManifestWork`.

### Non-Goals

- **Templating or CEL expressions in patch values.** Deferred to a separate enhancement
  covering both MWRS and `ManifestOverridePolicy`. The consensus on [#198][issue-198] is that
  it should use CEL with the full `ManagedCluster` object as context rather than fixed
  placeholders such as `{{CLUSTER_NAME}}`.
- **`forEach`-style iteration** over a list to generate repeated patches. Keeping `overrides`
  a list leaves syntactic room, but iteration source, variable scope and execution order are
  genuinely undecided; see [Open Questions](#open-questions).
- **Overriding anything outside the manifest payload.** `manifestConfigs`, `deleteOption` and
  the other `ManifestWorkSpec` fields are not patchable by this proposal.
- **Changing a manifest's identity.** A patch may not change `apiVersion`, `kind`, `name` or
  `namespace`; see [Identity invariant](#identity-invariant).
- **Selecting clusters by `Placement` or `ManagedClusterSet`.** Only a label selector in the
  first iteration; see [Alternatives](#alternatives).

## Proposal

### User Stories

#### Story 1: node pool differs per cluster

As a platform engineer, I distribute one Deployment to twenty clusters. Each cluster tags its
application node pool with a `nodepool` label on its `ManagedCluster`. I write one
`ManifestOverridePolicy` per distinct node pool value. Each MWRS that lists them gets the
right `nodeSelector` on every cluster, without one copy of the Deployment per node pool.

#### Story 2: more replicas in production

As an application owner, I want `replicas: 5` on clusters labelled `env: prod` and the
template's `replicas: 2` everywhere else, without maintaining two copies of the Deployment.

#### Story 3: a shared override reused by many workloads

As a platform engineer, every workload my team distributes must carry a compliance label and
tolerate the `dedicated` taint on clusters in `region: eu`. I define that once as a policy,
and each MWRS lists it by name rather than having the patch pasted into it.

#### Story 4: different overrides for different workloads

As a batch platform owner, I run jobs whose priority depends on both the job and the cluster
(from [@everpeace on #202][pr-202-everpeace]):

| Job | `cluster-x` | `cluster-y` |
| --- | --- | --- |
| A | low (template) | low (template) |
| B | high | middle |
| C | middle | low (template) |

I write one policy per (priority, cluster) pair I need — `x-high`, `x-middle`, `y-middle` —
and each job's MWRS lists the ones it wants: B lists `x-high` and `y-middle`, C lists
`x-middle`, A lists none. Adding a new policy for job D changes nothing for A, B or C.

#### Story 5: patching a custom resource

As a platform engineer, I distribute a Knative `Service` and need to add one environment
variable to its container on clusters in `region: east`, leaving the image and the sidecar
container intact. I register a `ResourceOverrideSchema` for `serving.knative.dev/v1 Service`
once, and strategic merge patches on that type then merge the container list by name instead
of replacing it.

### API Design

#### ManifestOverridePolicy

Namespaced. Applies only to a `ManifestWorkReplicaSet` in the same namespace that lists it in
`spec.overridePolicies`.

```go
// ManifestOverridePolicy defines patches applied to manifests distributed by the
// ManifestWorkReplicaSets that reference it, for the clusters it selects.
//
// +genclient
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
type ManifestOverridePolicy struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   ManifestOverridePolicySpec   `json:"spec"`
    Status ManifestOverridePolicyStatus `json:"status,omitempty"`
}

// ManifestOverridePolicySpec selects manifests and clusters, and the patches to apply
// where both match.
type ManifestOverridePolicySpec struct {
    // Priority orders policies against each other. Policies are applied in ascending
    // priority order, so a higher priority is applied later and wins a conflict.
    // Policies with equal priority are ordered by name.
    // +optional
    Priority int32 `json:"priority,omitempty"`

    // ResourceSelectors selects the manifests this policy patches. A manifest is
    // selected if it matches any selector. Name and namespace accept "*" to match any
    // value, as elsewhere in ManifestWork.
    // +kubebuilder:validation:MinItems=1
    ResourceSelectors []workv1.ResourceIdentifier `json:"resourceSelectors"`

    // Overrides are evaluated in list order. Every override whose ClusterSelector
    // matches the cluster contributes its patches, in order.
    // +kubebuilder:validation:MinItems=1
    Overrides []Override `json:"overrides"`
}

// Override applies a list of patches on the clusters matched by ClusterSelector.
type Override struct {
    // ClusterSelector selects clusters by ManagedCluster labels. If nil or empty, it
    // matches every cluster selected by the MWRS placements.
    // +optional
    ClusterSelector *metav1.LabelSelector `json:"clusterSelector,omitempty"`

    // Patches are applied in list order to each selected manifest.
    // +kubebuilder:validation:MinItems=1
    Patches []Patch `json:"patches"`
}

// Patch is a single patch applied to a selected manifest.
type Patch struct {
    // Type is the patch format.
    Type PatchType `json:"type"`

    // Patch is the patch body, written as an object or array rather than a string so
    // it stays readable in the resource. The controller validates it against Type.
    //
    // It is apiextensionsv1.JSON rather than runtime.RawExtension because
    // controller-gen renders RawExtension as "type: object", and an RFC 6902 patch body
    // is a JSON array, so every JSONPatch would be rejected at admission.
    Patch apiextensionsv1.JSON `json:"patch"`

    // AtomicLists sets what happens when a StrategicMergePatch would replace a list
    // that is atomic, or whose merge semantics the available schema does not describe.
    // Ignored for JSONPatch and JSONMergePatch.
    // +kubebuilder:default=Reject
    // +optional
    AtomicLists AtomicListPolicy `json:"atomicLists,omitempty"`
}

// PatchType is the format of a Patch body.
// +kubebuilder:validation:Enum=JSONPatch;JSONMergePatch;StrategicMergePatch
type PatchType string

const (
    // PatchTypeJSONPatch is an RFC 6902 JSON Patch.
    PatchTypeJSONPatch PatchType = "JSONPatch"
    // PatchTypeJSONMergePatch is an RFC 7386 JSON Merge Patch.
    PatchTypeJSONMergePatch PatchType = "JSONMergePatch"
    // PatchTypeStrategicMergePatch is a Kubernetes strategic merge patch.
    PatchTypeStrategicMergePatch PatchType = "StrategicMergePatch"
)

// AtomicListPolicy sets how a strategic merge patch treats a list it cannot merge.
// +kubebuilder:validation:Enum=Reject;Replace
type AtomicListPolicy string

const (
    // AtomicListReject fails the patch rather than discarding list entries.
    AtomicListReject AtomicListPolicy = "Reject"
    // AtomicListReplace replaces the whole list, for users who intend it.
    AtomicListReplace AtomicListPolicy = "Replace"
)

// ManifestOverridePolicyList is a list of ManifestOverridePolicy.
// +kubebuilder:object:root=true
type ManifestOverridePolicyList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`

    Items []ManifestOverridePolicy `json:"items"`
}

// ManifestOverridePolicyStatus is the observed state of a ManifestOverridePolicy.
type ManifestOverridePolicyStatus struct {
    // ObservedGeneration is the generation of the policy that Conditions describe.
    // +optional
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`

    // Conditions describe the state of the policy. The known type is Rendered.
    // +listType=map
    // +listMapKey=type
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

const (
    // ManifestOverridePolicyConditionRendered is True when every render that uses the
    // policy succeeded, and False with reason RenderFailed when any failed. The message
    // gives the number of ManifestWorkReplicaSets and clusters affected; details are on
    // each ManifestWorkReplicaSet.
    ManifestOverridePolicyConditionRendered string = "Rendered"
)
```

The policy status is a summary for the policy author. The per-cluster detail lives on the
MWRS, see [Observability](#observability).

#### ResourceOverrideSchema

Cluster-scoped and administrator-managed. Schema selection has to be unambiguous, so
uniqueness is defined explicitly rather than left to "one per type".

**The uniqueness key is `(spec.target.group, spec.target.resource, spec.version)`.** Two
objects sharing that triple conflict, whatever they are named.

`spec.kind` is deliberately *not* part of the key. Within one group/version a resource maps to
exactly one kind, so admitting `kind` into the key would permit two objects that disagree
about the kind for the same resource — precisely the ambiguity the rule exists to prevent. The
reverse mapping must be unique for the same reason: two objects mapping the same
`(group, version, kind)` to different resources also conflict.

Different versions of one resource are *not* a conflict. A type served at both `v1` and
`v1beta1` may legitimately carry different list metadata per version, and the manifest's
`apiVersion` selects which schema applies.

`spec.target.name` and `spec.target.namespace` must be empty — this names a type, not an
instance. `ResourceIdentifier` carries those fields because `manifestConfigs` needs them;
here they are rejected by validation rather than silently ignored.

Conflicts are rejected deterministically rather than resolved by picking a winner:

- Both objects get a `Conflicting` condition naming the other, so the problem is visible on
  the objects themselves and not only in controller logs.
- Any render needing that schema fails for the affected cluster, with an error naming both
  objects and the duplicated key. It does **not** fall back to treating the type as
  schemaless: that would silently downgrade a merge to an atomic replacement, which is the
  exact failure `atomicLists: Reject` exists to prevent.
- Renders that do not touch the contested type are unaffected.

Last-writer-wins and lowest-name-wins were both considered and rejected: either makes the
rendered payload depend on creation order or on object naming, which contradicts the
determinism guarantee in [Ordering](#ordering). A validating webhook rejecting the second
object at admission is worthwhile hardening on top, but the controller-side rejection is
required regardless, since a webhook cannot catch objects created while it is unavailable.

```go
// ResourceOverrideSchema supplies list-merge metadata for a resource type, so that
// strategic merge patches can merge lists instead of replacing them. It affects
// override rendering only: it does not modify any CRD or API server validation.
//
// +genclient
// +genclient:nonNamespaced
// +kubebuilder:object:root=true
// +kubebuilder:resource:scope=Cluster
// +kubebuilder:subresource:status
type ResourceOverrideSchema struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   ResourceOverrideSchemaSpec   `json:"spec"`
    Status ResourceOverrideSchemaStatus `json:"status,omitempty"`
}

// ResourceOverrideSchemaSpec names a resource type and describes how its lists merge.
type ResourceOverrideSchemaSpec struct {
    // Target is the resource type this schema describes, using the same group/resource
    // identity as ManifestOverridePolicy resourceSelectors. Name and namespace must be
    // empty: this names a type, not an instance.
    Target workv1.ResourceIdentifier `json:"target"`

    // Kind is the manifest kind served for Target. With Version, it gives the hub the
    // kind-to-resource mapping for a type that is installed only on managed clusters,
    // where the hub has no RESTMapper entry.
    // +kubebuilder:validation:MinLength=1
    Kind string `json:"kind"`

    // Version is the served version this schema describes.
    // +kubebuilder:validation:MinLength=1
    Version string `json:"version"`

    // OpenAPIV3Schema describes the merge semantics of the paths a patch may touch.
    // If the hub has the CRD, it is overlaid on the CRD schema; otherwise it is the
    // whole schema. See Effective schema.
    OpenAPIV3Schema apiextensionsv1.JSON `json:"openAPIV3Schema"`
}

// ResourceOverrideSchemaStatus is the observed state of a ResourceOverrideSchema.
type ResourceOverrideSchemaStatus struct {
    // ObservedGeneration is the generation of the schema that Conditions describe.
    // +optional
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`

    // Conditions describe the state of the schema. The known type is Conflicting.
    // +listType=map
    // +listMapKey=type
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

const (
    // ResourceOverrideSchemaConditionConflicting is True when another
    // ResourceOverrideSchema has the same uniqueness key. The message names the other
    // object. While it is True, renders that need this type fail.
    ResourceOverrideSchemaConditionConflicting string = "Conflicting"
)

// ResourceOverrideSchemaList is a list of ResourceOverrideSchema.
// +kubebuilder:object:root=true
type ResourceOverrideSchemaList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`

    Items []ResourceOverrideSchema `json:"items"`
}
```

Making `ResourceOverrideSchema` carry the mapping resolves two things at once. It gives the
API a single resource-identification scheme, as @kahirokunn asked for on [#198][issue-198]
and as review feedback on [#155][pr-155] preferred. And it closes a hole that group/resource
selection would otherwise have: the hub cannot discover a type it does not have installed, so
without this the Knative example below could not be selected at all.

Example, declaring that a Knative `Service`'s containers and their environment variables
merge by `name`:

```yaml
apiVersion: work.open-cluster-management.io/v1alpha1
kind: ResourceOverrideSchema
metadata:
  name: knative-service
spec:
  target:
    group: serving.knative.dev
    resource: services
  kind: Service
  version: v1
  openAPIV3Schema:
    type: object
    properties:
      spec:
        type: object
        properties:
          template:
            type: object
            properties:
              spec:
                type: object
                properties:
                  containers:
                    type: array
                    x-kubernetes-list-type: map
                    x-kubernetes-list-map-keys: ["name"]
                    items:
                      type: object
                      properties:
                        name:
                          type: string
                        env:
                          type: array
                          x-kubernetes-list-type: map
                          x-kubernetes-list-map-keys: ["name"]
                          items:
                            type: object
                            properties:
                              name:
                                type: string
```

Knative is installed only on the managed clusters, so the hub has no `Service` schema to
inherit from and this document is the whole of the type information. It is written out in
full for that reason: every ancestor carries `type: object`, every map list declares its
`items` and the scalar property named by `x-kubernetes-list-map-keys`. A key that is named
but not declared cannot be honoured.

The markers are the structural ones, `x-kubernetes-list-type` and
`x-kubernetes-list-map-keys`, because those are what `structured-merge-diff` reads.

This is a deliberate difference from the [custom OpenAPI schema example][kustomize-openapi]
Kustomize documents, which uses the older `x-kubernetes-patch-strategy` /
`x-kubernetes-patch-merge-key` pair that `strategicpatch` reads. The difference matters
rather than being cosmetic: users will arrive carrying the Kustomize form, and an overlay
supplying only those markers leaves the list with no effective merge metadata under the
engine this proposal uses. The list would be treated as atomic, so the merge fails under the
default `atomicLists: Reject`, or discards existing entries under `Replace`. Neither is
acceptable as a silent outcome.

`ResourceOverrideSchema` therefore rejects the legacy pair at validation, with an error
giving the structural equivalent:

| Legacy marker | Structural equivalent |
| --- | --- |
| `x-kubernetes-patch-strategy: merge` + `x-kubernetes-patch-merge-key: name` | `x-kubernetes-list-type: map` + `x-kubernetes-list-map-keys: ["name"]` |
| array with no `x-kubernetes-patch-strategy` | `x-kubernetes-list-type: atomic` |

Accepting both forms and converting internally was considered and rejected: it puts two ways
of expressing one thing on a brand-new API, and commits us to keeping the conversion correct
forever, to save the author a one-line edit that the validation error spells out.

#### Effective schema

Strategic merge for a custom resource needs a schema to convert against, and in alpha the
hub may have none, so the resolution order has to be stated rather than assumed:

1. If the hub has the CRD, its structural schema for the matching version is the base, and
   the registered `ResourceOverrideSchema` is overlaid on it. Overlaid metadata wins.
2. If the hub does not have the CRD, the registered schema is the base. There is nothing to
   inherit.
3. In both cases, any path the resulting schema does not describe falls back to the deduced
   semantics `structured-merge-diff` uses for untyped data.

Step 3 is the part that matters, because the fallback is not neutral. In the vendored
`structured-merge-diff` v6, `DeducedParseableType` defines `__untyped_deduced_` with maps
`separable` and lists `atomic`. So an undescribed map still merges field by field, while an
**undescribed list is atomic**. Combined with the `atomicLists: Reject` default, a list the
author forgot to describe fails the patch with an error naming the path, rather than
silently replacing the list. That is the safe direction, and it is why the fallback is
acceptable for alpha rather than a hole.

The practical rule for authors is therefore narrow: describe every list you expect to merge,
down to the scalar properties named as its map keys. Everything else may be omitted.

This is also the sharpest argument for the agent-sourced schema cache in
[open question 1](#open-questions). Once schemas are collected from the clusters that
actually serve the type, step 2 stops being a hand-written document and the fallback stops
being reachable in normal use.

#### ManifestWorkReplicaSet references

An MWRS names the policies that apply to it. This is the only change to an existing API in
this proposal.

```go
// ManifestWorkReplicaSetSpec gains one optional field. Existing fields are unchanged.
type ManifestWorkReplicaSetSpec struct {
    // ... ManifestWorkTemplate, PlacementRefs and CascadeDeletionPolicy ...

    // OverridePolicies lists the ManifestOverridePolicy objects, in the same namespace,
    // applied to this ManifestWorkReplicaSet. List order does not affect the result;
    // see Ordering.
    // +listType=map
    // +listMapKey=name
    // +optional
    OverridePolicies []LocalOverridePolicyReference `json:"overridePolicies,omitempty"`
}

// LocalOverridePolicyReference names a ManifestOverridePolicy in the same namespace.
// It is a struct rather than a plain string so that fields can be added later without
// changing the type of spec.overridePolicies.
type LocalOverridePolicyReference struct {
    // Name is the name of the ManifestOverridePolicy.
    // +kubebuilder:validation:MinLength=1
    Name string `json:"name"`
}
```

The policies applied to an MWRS are exactly the ones listed in `spec.overridePolicies`. Nothing
is added to the list, and nothing outside it is applied. The MWRS spec then shows every policy
that shapes its payload, following the Kubernetes object model. This follows @everpeace's
proposal on [#202][pr-202-everpeace], and @kahirokunn's decision as approver on
[#202][pr-202-kahirokunn-declarative] to keep the alpha API declarative.

The validating webhook checks that names in `overridePolicies` are unique. It does **not**
reject a name that does not exist yet, so an MWRS and its policies can be applied in any order,
for example by a GitOps tool syncing a directory. A listed policy that does not exist is
reported on the MWRS (see [Failure handling](#failure-handling)), not ignored.

Adding policies to an MWRS automatically (default policies), and making some policies
mandatory, are out of scope for alpha; see [open question 8](#open-questions).

### Example

```yaml
apiVersion: work.open-cluster-management.io/v1alpha1
kind: ManifestOverridePolicy
metadata:
  name: nodepool-per-cluster
  namespace: team-a
spec:
  priority: 10
  resourceSelectors:
    - group: apps
      resource: deployments
      namespace: "*"
      name: "*"
  overrides:
    - clusterSelector:
        matchLabels:
          nodepool: apps-a
      patches:
        - type: JSONMergePatch
          patch:
            spec:
              template:
                spec:
                  nodeSelector:
                    nodepool: apps-a
    - clusterSelector:
        matchLabels:
          nodepool: apps-b
      patches:
        - type: JSONMergePatch
          patch:
            spec:
              template:
                spec:
                  nodeSelector:
                    nodepool: apps-b
```

The `ManifestWorkReplicaSet` lists the policy it uses:

```yaml
apiVersion: work.open-cluster-management.io/v1alpha1
kind: ManifestWorkReplicaSet
metadata:
  name: web-app
  namespace: team-a
spec:
  manifestWorkTemplate:
    workload:
      manifests:
        - apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: web-app
            namespace: default
          spec:
            replicas: 2
            # ...
  placementRefs:
    - name: web-app-placement
  overridePolicies:
    - name: nodepool-per-cluster
```

On a cluster labelled `nodepool: apps-a` the Deployment gets `nodeSelector: {nodepool: apps-a}`,
on `nodepool: apps-b` it gets `apps-b`, and on any other cluster it is the plain template.

### Why explicit references

Three models were discussed on [#198][issue-198] and [#202][pr-202]:

1. **Implicit application with an opt-out.** Every policy in the namespace applies to every
   MWRS, unless the MWRS sets `overrideMode: Disabled`. This was an earlier draft of this
   proposal. @kahirokunn's point behind it was that "policies will inevitably be missed in
   enumeration".
2. **Explicit references with defaults.** The MWRS lists its policies, a mutating webhook adds
   policies marked `default: true` when the MWRS is created, and an MWRS can skip one with
   `disabled: true`. This was the draft after (1).
3. **Explicit references only.** The MWRS lists its policies and nothing else applies.

This proposal takes (3).

- (1) cannot express Story 4. An all-or-nothing switch cannot give a job some policies but
  not others. A deny list could, but then every new policy reaches every existing MWRS until
  each one is updated to deny it.
- (2) was dropped on review. @everpeace asked whether a default that MWRS can skip is really a
  default, and pointed out a race between changing `default` and creating an MWRS. It also
  meant the stored MWRS differs from what the user wrote, which shows up as drift in GitOps
  tools, and it needed a new mutating webhook that fails MWRS creation when unavailable.
  @kahirokunn agreed to keep alpha declarative.

What (3) gives up: an author can forget to list a platform policy, which was the concern
behind (1). Users who need policies added or enforced automatically can use existing
Kubernetes admission mechanisms or GitOps tooling for now, as @kahirokunn noted. OCM support
for it is open question 8.

### Implementation Details

#### Feature gate

The feature is behind a new hub work feature gate, `ManifestOverridePolicy`, Alpha and off by
default. It is named after the resource it adds, like the existing `ManifestWorkReplicaSet`
gate, and it only has an effect when that gate is also enabled.

With the gate off:

- The MWRS controller starts none of the new informers, so a hub that does not use the
  feature does not pay for the `ManagedCluster` cache. It ignores `spec.overridePolicies` and
  renders the plain template.
- The validating webhook rejects a create or update that adds an entry to
  `spec.overridePolicies`, or changes one, with an error naming the feature gate. This is the
  same approach the MWRS webhook already takes for the `ManifestWorkReplicaSet` gate. An update
  that leaves the existing entries unchanged is allowed, so an MWRS created while the gate was
  on can still be edited after it is turned off.
- The validating webhooks for `ManifestOverridePolicy` and `ResourceOverrideSchema` are not
  registered, and the objects have no effect.

#### A single renderer, used by all three call sites

This is a correctness constraint, not a preference. The template is copied in three places in
the MWRS controller, and two of them are drift detection:

| Site | Purpose |
|---|---|
| `manifestworkreplicaset_deploy_reconcile.go` (candidate build) | builds a candidate and calls `workapplier.ManifestWorkEqual` |
| `manifestworkreplicaset_status_reconcile.go` | the same comparison, for status accounting |
| `manifestworkreplicaset_deploy_reconcile.go` (create) | creates the `ManifestWork` |

If overrides were applied only at creation, both comparison sites would compare an
un-rendered template against a rendered live object. They would never agree. The cluster
would stay `ToApply`, `Progressive` rollout would keep selecting the same clusters,
`PlacementRolledOut` would never reach `Complete`, and the status counts would be wrong.

All three sites must therefore route through one function:

```go
// Render returns the ManifestWorkSpec for one cluster: template with the patches of
// every matching policy applied. It does not modify its arguments, and the same inputs
// always produce the same output.
func Render(template *workv1.ManifestWorkSpec, cluster *clusterv1.ManagedCluster,
    policies []*workv1alpha1.ManifestOverridePolicy, types TypeResolver) (*Result, error)

// TypeResolver maps a manifest's kind to its resource and its merge schema. It is
// backed by hub discovery and by ResourceOverrideSchema objects.
type TypeResolver interface {
    ResourceFor(gvk schema.GroupVersionKind) (schema.GroupVersionResource, error)
    // MergeSchema returns nil for a built-in type, which uses strategicpatch.
    MergeSchema(gvk schema.GroupVersionKind) (*typed.ParseableType, error)
}

// Result is a rendered ManifestWorkSpec and the policies that changed it.
type Result struct {
    Spec *workv1.ManifestWorkSpec
    // AppliedPolicies are the names of the policies that patched at least one
    // manifest, in the order they were applied.
    AppliedPolicies []string
}
```

`Render` takes the whole `ManagedCluster` rather than its labels so that the templating
enhancement, which will evaluate CEL against the cluster object, does not need a new
signature. It returns an error of type `*RenderError`, which carries the policy, manifest,
override and patch index of the failure.

A prototype with unit tests for the scenarios in [#198][issue-198] is in
[ocm#1792][ocm-1792]. It still uses the earlier signature and will be updated to match.

#### Ordering

1. The policies listed in `spec.overridePolicies` are sorted by `(priority, name)`, ascending.
   Their order in the list is ignored, so reordering the list never changes the result.
2. Within a policy, `overrides` are evaluated in list order; an override contributes only if
   its `clusterSelector` matches the cluster.
3. Within an override, `patches` are applied in list order.

Later patches see the result of earlier ones. Because the sort is total and every list is
ordered, two renders of the same inputs always produce the same bytes.

#### Resource selection

`resourceSelectors` reuses `ManifestWork`'s existing `ResourceIdentifier`
(group / resource / namespace / name), the same type `manifestConfigs` uses, with the same
`"*"` wildcard support for name and namespace. This keeps one resource-addressing concept in
the work API and lets the implementation reuse `helper.ResourceMatch`.

This requires mapping each manifest's `apiVersion`/`kind` to its resource, which the work hub
does not do today — it needs a `RESTMapper`. A `RESTMapper` built from hub discovery covers
built-in types and any CRD installed on the hub. For a type installed only on managed
clusters, the mapping comes from the `kind`/`version`/`target` triple on its
`ResourceOverrideSchema`, as described above. A manifest whose type can be resolved by
neither route and which no selector matches is simply left unpatched; if a selector does
name an unresolvable group/resource, the render fails for that cluster with an error naming
the type, rather than silently skipping the manifest.

#### Patch application

- **JSONPatch** — RFC 6902, via `github.com/evanphx/json-patch/v5`. v5 is used deliberately:
  v4 lets `replace` on a missing field succeed silently, v5 returns an error. `sdk-go`
  already uses v5.
- **JSONMergePatch** — RFC 7386.
- **StrategicMergePatch** — for built-in types the merge keys come from the Go types in
  `client-go`'s scheme. For custom resources they come from the type's CRD schema, overlaid
  with any `ResourceOverrideSchema` registered for that type.

For custom resources the implementation uses typed conversion plus merge from
`sigs.k8s.io/structured-merge-diff`, which is already vendored in OCM, rather than
`strategicpatch`. This is the approach Kubernetes' own `MutatingAdmissionPolicy`
`ApplyConfiguration` takes. It is necessary because a raw CRD schema declares list semantics
with `x-kubernetes-list-type` and `x-kubernetes-list-map-keys`, which `strategicpatch` does
not read; handing it a CRD schema is not sufficient on its own. A composite
`x-kubernetes-list-map-keys` must be honoured in full and must never be silently reduced to
its first field.

When the available schema cannot support the requested merge — an atomic list, or a map list
whose keys cannot be honoured — the patch fails rather than silently replacing the list,
unless the patch sets `atomicLists: Replace`.

**Deletion semantics must not diverge between built-ins and custom resources.** This is a
trap in the approach above. A strategic merge patch deletes a field by setting it to `null`,
and removes a list entry with the `$patch: delete` directive; `strategicpatch` implements
both. `structured-merge-diff`'s `TypedValue.Merge` does not — it merges, and expresses
removal through field-set subtraction in the apply path, not through nulls in the merge
input. Using `strategicpatch` for built-ins and typed merge for custom resources would
therefore give the same patch two different meanings depending on the target type, and the
difference would be silent.

The implementation must close this explicitly. Before the typed merge, the patch is scanned
for `null` values and `$patch` directives; those are extracted and applied as removals
against the merged result, so that `spec: {field: null}` and `$patch: delete` mean the same
thing for a Knative `Service` as they do for a `Deployment`. Any directive that cannot be
honoured for a custom resource is an error naming the directive and the path, never a
silent no-op. The test plan covers deletion on both a built-in and a custom resource with
the same patch body, asserting identical results.

#### Identity invariant

After all patches are applied, the rendered manifest's `apiVersion`, `kind`, `name` and
`namespace` must equal the original's. A patch that changes any of them is an error. Without
this, a patch could retarget a manifest at a different object, orphan the previously applied
resource, and make `ManifestWork` status accounting meaningless.

#### Controller wiring

The MWRS controller gains four informers:

- `ManifestOverridePolicy`, to re-render when a policy changes. The informer has an index
  from policy to the MWRS that list it in `spec.overridePolicies`, so a policy change
  enqueues only those MWRS instead of every MWRS in the namespace.
- `ManagedCluster`, because label selectors need cluster labels, which `PlacementDecision`
  does not carry.
- `ResourceOverrideSchema`, on create, update **and delete**. Merge metadata is an input to
  the rendered output, so editing a schema changes what the manifest should be. Without this
  watch, existing `ManifestWork`s keep the payload rendered under the old metadata until some
  unrelated event happens to reconcile their MWRS, which is both wrong and hard to diagnose.
  Deletion matters as much as update: removing a schema can turn a previously valid merge
  into one that must now fail under `atomicLists: Reject`.
- `CustomResourceDefinition` on the hub, because a hub CRD is an input in two ways:
  - **Resource mapping, for every patch type.** The hub `RESTMapper` resolves
    `resourceSelectors` through hub CRDs (see [Resource selection](#resource-selection)), so
    creating or deleting a CRD, or changing the versions it serves, changes which manifests a
    selector matches. Without this, a render that failed on an unresolvable selector is never
    retried once the CRD is installed, and a deleted CRD leaves the old render in place.
  - **Effective schema, for strategic merge.** The hub CRD is the base of the
    [effective schema](#effective-schema), so a schema change such as a list moving from `map`
    to `atomic` changes the merge result.

  An index from group and resource to the MWRS whose listed policies select that type keeps
  unrelated CRD churn from enqueuing anything. On create, delete, or a change to served
  versions, every MWRS in the index is enqueued and the `RESTMapper` is reset. An update that
  changes only the schema enqueues just the MWRS whose policies carry a strategic merge patch
  for that type. Other updates are dropped.

The `ManagedCluster` informer is the significant cost. It caches every `ManagedCluster` on the
hub and must re-render affected MWRS when labels change. Mitigations to measure before
committing: an informer `transform` that drops `managedFields` and `status`, and event
filtering so that only label changes enqueue work. See
[Risks and Mitigation](#risks-and-mitigation).

#### Failure handling

A render failure affects one cluster. The implementation:

- does not create or update that cluster's `ManifestWork`;
- reports the cluster to the rollout machinery as `Failed`, not as `ToApply`. This matters:
  a cluster stuck in `ToApply` never counts toward `maxFailures` and a `Progressive` rollout
  with a `progressDeadline` would stall instead of failing;
- sets the MWRS `OverridesRendered` condition to `False` with reason `RenderFailed` (see
  [Observability](#observability)), so a failure anywhere is visible without inspecting each
  cluster;
- includes the offending policy, manifest index, override index and patch index in the
  message. The prototype's `RenderError` already carries exactly these.

Clusters whose render succeeded are unaffected.

A listed policy that does not exist is different, because it affects every cluster of the
MWRS rather than one. The MWRS `OverridesRendered` condition is set to `False` with reason
`OverridePolicyNotFound` naming the policy, and its existing `ManifestWork`s are left as they are rather than rewritten to
the template without that policy. Rendering resumes when the policy is created or the
reference is removed. Leaving the works alone is the safer direction: if a policy is deleted
by mistake, the workload keeps running as it was.

#### Observability

Two mechanisms, in the spirit of `ManagedClusterAddOn.status.configReferences`: the object
that was changed says what changed it, and the object the user owns says whether it worked.

**On each `ManifestWork`: an annotation.**

```go
const (
    // ManifestWorkOverridePoliciesAnnotationKey is set by the MWRS controller on each
    // ManifestWork it renders. The value is a comma-separated list of the names of the
    // ManifestOverridePolicy objects that patched at least one manifest, in the order
    // they were applied. It is not set when no policy patched the ManifestWork.
    ManifestWorkOverridePoliciesAnnotationKey = "work.open-cluster-management.io/override-policies"
)
```

For example, `work.open-cluster-management.io/override-policies: base,prod,prod-hotfix`. Names
are enough because policies are in the MWRS namespace.

An annotation rather than a status field, because `ManifestWork` status is written by the work
agent, while the hub controller owns the object's spec and metadata. It already sets the
`work.open-cluster-management.io/manifestworkreplicaset-owner` annotation the same way. The
annotation is part of the rendered object, so `ManifestWorkEqual` compares it like any other
annotation: when the set of applied policies changes, the `ManifestWork` is updated, and when
it becomes empty the annotation is removed.

**On the MWRS: one condition.**

```go
const (
    // ManifestWorkReplicaSetConditionOverridesRendered reports whether the listed
    // override policies rendered for every selected cluster. It is set only when
    // spec.overridePolicies is not empty.
    ManifestWorkReplicaSetConditionOverridesRendered string = "OverridesRendered"

    // ReasonRenderFailed means the render failed for one or more clusters.
    ReasonRenderFailed = "RenderFailed"

    // ReasonOverridePolicyNotFound means a listed policy does not exist.
    ReasonOverridePolicyNotFound = "OverridePolicyNotFound"
)
```

| Status | Reason | When |
| --- | --- | --- |
| `True` | `AsExpected` (existing reason) | Every selected cluster rendered. |
| `False` | `OverridePolicyNotFound` | A listed policy does not exist. Takes precedence, since it affects every cluster. |
| `False` | `RenderFailed` | One or more clusters failed to render. |

For `RenderFailed`, the message gives the failed count and the total, then the first five
failed clusters sorted by name, each with its `RenderError`:

```yaml
- type: OverridesRendered
  status: "False"
  reason: RenderFailed
  message: '1 of 3 clusters failed: cluster-c (policy gpu-pool, manifest 0, override 0, patch 0: add operation does not apply: doc is missing path "/spec/template/spec/nodeSelector/pool")'
```

This answers "is anything failing?" from the MWRS alone, as asked on [#198][issue-198]. The
condition is not set on an MWRS that lists no policies, so the status of existing MWRS does
not change on upgrade.

### Risks and Mitigation

- **`ManagedCluster` informer cost.** Memory grows with fleet size and label churn triggers
  re-render. Needs measurement at realistic fleet scale before this graduates. Mitigated by an
  informer `transform` and label-only event filtering.
- **One policy edit changes many workloads.** A shared policy is listed by many MWRS, so
  editing it re-renders all of them. This is the point of sharing, but the blast radius is
  real. Mitigated by policies being namespaced, by provenance on each `ManifestWork`, and by
  MWRS rollout strategies applying to the change like any other.
- **A platform policy is not listed.** Policies apply only where they are listed, so an MWRS
  author can forget one. This is the cost of a declarative spec. Teams that need policies
  added or enforced automatically can use Kubernetes admission (for example a
  `MutatingAdmissionPolicy`) or their GitOps tooling until OCM support is decided in open
  question 8.
- **Silent data loss through list replacement.** The reason `atomicLists` defaults to
  `Reject`. Replacing a container list and losing a sidecar is the kind of failure that is
  discovered in production.
- **Two overlapping customisation mechanisms.** If templating lands later, users will have
  patches and templates to choose between, with a permanent documentation cost. Keeping
  templating out of this proposal limits the combined surface for now, but the interaction
  must be designed when templating is proposed.
- **A new CRD is a permanent commitment.** Install, RBAC, bundle packaging downstream,
  upgrade handling, QE matrix and support burden. This is the main argument for the
  MWRS-embedded alternative below.
- **Divergence from the `ManagedCluster` as source of truth.** A fleet-wide value such as a
  node pool name is a fact about the cluster and is already recorded on the `ManagedCluster`.
  Restating the cluster-to-value mapping inside policies duplicates it, and the duplicate can
  go stale. Templating would remove this duplication; until then, label selectors group
  clusters so that the mapping is written per distinct value, not per cluster.

## Design Details

### Open Questions

1. ~~**Where do CRD schemas come from?**~~ **Resolved.** Reading a CRD schema requires the
   CRD on the hub, but a type such as Knative may only be installed on managed clusters.
   Options discussed on [#198][issue-198]:
   a. require a `ResourceOverrideSchema` and error otherwise — no new machinery, but the
      administrator must supply the schema;
   b. add discovery on the hub — only works for types installed on the hub;
   c. collect schemas from managed clusters and keep a content-hash-deduplicated cache on the
      hub, with small per-cluster references, as proposed by @kahirokunn. This handles clusters
      whose schemas genuinely differ for the same type, but requires an agent-to-hub schema
      reporting channel that does not exist in `work` today, plus a refresh and
      garbage-collection story. That is arguably its own enhancement.

   Settled on [#198][issue-198] (2026-11-06): **alpha ships (a)** — built-in types plus
   custom resources with an explicitly registered `ResourceOverrideSchema` — and **(c) gates
   beta graduation rather than existence**.

   The disagreement was about sequencing, not destination. @kahirokunn's objection was that
   manual registration is a poor first experience and does not handle clusters whose schemas
   genuinely differ for the same type, which is correct, and (c) remains the intended normal
   path. The counter-position was that (c) is a new hub/agent contract carrying its own skew,
   refresh and garbage-collection design, so making it a prerequisite couples this proposal's
   delivery to a second and larger one. Gating graduation keeps both true: no user meets
   manual registration as the permanent answer, and neither design blocks the other. See
   [Graduation Criteria](#graduation-criteria).

2. ~~**One identifier scheme or two?**~~ **Resolved.** `ResourceIdentifier` (group/resource)
   is used throughout — `resourceSelectors` and `ResourceOverrideSchema.spec.target` — per
   @kahirokunn on [#198][issue-198] and review feedback on [#155][pr-155]. The objection
   that group/resource needs hub discovery for types the hub does not have installed is
   answered by `ResourceOverrideSchema` carrying `kind` and `version`, so registering a
   schema also registers the mapping. See [Resource selection](#resource-selection).

3. **How is `forEach` added later, if at all?** Keeping `overrides` a list leaves room, but
   the loop boundary determines variable scope and execution order. Given
   `forEach: [{region: [east, west]}]` with two patches appending `--start=$region` and
   `--end=$region`, an override-level loop yields
   `["--start=east", "--end=east", "--start=west", "--end=west"]` while a patch-level loop
   yields `["--start=east", "--start=west", "--end=east", "--end=west"]`. Also undecided:
   what may drive iteration (literal lists, `ManagedCluster` data, fields in the matched
   manifest), and whether a loop variable may be used in `clusterSelector` — which in turn
   decides whether expansion happens before or after cluster matching. Recorded here so the
   templating enhancement inherits the questions rather than rediscovering them.

4. ~~**Scope of the first implementation.**~~ **Resolved.** Built-in types and
   `ResourceOverrideSchema`-backed custom resources ship together in alpha, per the same
   [#198][issue-198] agreement. This follows from question 1: with (a) as the alpha answer
   there is no reason to hold custom resources back, and splitting them would have put one
   behaviour through two rounds of review. Delivery is still expected to be several PRs.

5. ~~**Implicit application versus explicit `overridePolicies` references.**~~
   **Resolved.** Explicit references only, per @everpeace and @kahirokunn on
   [#202][pr-202-kahirokunn-declarative]. Default policies and a per-entry `disabled` field
   were in an earlier draft and were removed. See
   [Why explicit references](#why-explicit-references).

6. **Should policies be namespaced or cluster-scoped?** Proposed: **namespaced**, as
   suggested on [#198][issue-198] with no objection raised. Waiting for approver
   confirmation before this is marked resolved.
   - Authorisation follows the namespace that already governs the MWRS: anyone who can
     create an MWRS there can see and reference the policies that may change it, and nobody
     outside the namespace can change its payload.
   - It matches `placementRefs`, which also name objects in the MWRS namespace.
   - The cost: a platform team wanting a fleet-wide override creates one policy per
     namespace. A cluster-scoped kind can be added later without changing this one, if field
     experience shows that cost matters.

7. ~~**How does `clusterSelector` combine with other targeting added later?**~~
   **Resolved.** Alpha selects clusters only by `clusterSelector`. If ClusterSet or Placement
   targeting is added to `Override` later, the conditions are combined with AND: a cluster
   must match every condition set on the same override, per @kahirokunn on
   [#202][pr-202-kahirokunn].

8. **Should OCM add policies to an MWRS automatically, or enforce them?** Out of scope for
   alpha, per @kahirokunn and @everpeace on [#202][pr-202-kahirokunn-declarative]. Two
   separate needs:
   - *Default policies*: added to new MWRS so authors don't have to remember them. An
     earlier draft did this with `default: true` and a mutating webhook, and it was dropped:
     a default that MWRS can skip is not really a default, changing `default` races with
     MWRS creation, and webhook-added entries show up as GitOps drift.
   - *Required policies*: applied whether or not the MWRS lists them, for compliance. This
     is enforcement rather than convenience, and needs its own design.

   Until then, Kubernetes admission or GitOps tooling can cover both.

### Test Plan

**Unit tests** on the renderer, which is pure and the natural place for most coverage:
each patch type; ordering across priority, overrides and patches; wildcard and non-matching
resource selectors; cluster selectors that match, do not match, and are empty; the identity
invariant; atomic-list reject and the explicit `Replace` opt-in; composite
`x-kubernetes-list-map-keys`; `ResourceOverrideSchema` overlay precedence over the CRD schema;
two schemas sharing a uniqueness key, asserting that the render fails naming both rather than
picking one, and that a second schema differing only in `spec.version` is accepted; an overlay
carrying the legacy `x-kubernetes-patch-strategy` pair, asserting a validation error that
names the structural equivalent rather than a silently atomic list; a managed-cluster-only
type whose registered schema omits a list the patch touches, asserting that the deduced
fallback makes it atomic and that `atomicLists: Reject` therefore fails with the path named,
rather than replacing the list; determinism, by rendering
twice and comparing bytes. [ocm#1792][ocm-1792] already carries this suite for the agreed
subset.

Two cases deserve naming because they guard against silent divergence rather than an
outright bug. **Deletion parity**: the same strategic merge patch using `null` and
`$patch: delete` is applied to a built-in and to a custom resource, and the two results must
agree — this is the gap between `strategicpatch` and typed merge described above. **JSON
Patch body shape**: a policy whose `patch` is a JSON array round-trips through the CRD
schema, which is what `apiextensionsv1.JSON` buys over `runtime.RawExtension`.

**Integration tests** on the controller: that a policy change re-renders existing
`ManifestWork`s; that the two drift-detection sites agree with the rendered object, so no
churn occurs — a regression test for the failure mode described above; that a render failure
marks one cluster `Failed` and leaves others untouched; that a `Progressive` rollout with
`maxFailures` and `progressDeadline` reacts to a failing cluster instead of stalling; that
`ManagedCluster` label changes re-render; that creating, editing and deleting a
`ResourceOverrideSchema` re-renders the `ManifestWork`s whose merge result it changes; that
a hub CRD whose list changes from `map` to `atomic` re-renders the affected MWRS and fails
under `atomicLists: Reject`, while a change to an unrelated CRD enqueues nothing; that
installing a hub CRD retries a JSON Patch render that failed on an unresolvable selector, and
deleting it re-renders the MWRS; that
an MWRS is patched only by the policies it lists, while a sibling MWRS in the same
namespace that does not list the policy renders the plain template; that removing a reference
re-renders without that policy; that the `override-policies` annotation lists the applied
policies in order and is removed when none apply; that `OverridesRendered` is `False` with
`RenderFailed` when one cluster fails and `True` again once it renders; that a missing referenced policy sets
`OverridePolicyNotFound` and leaves the existing `ManifestWork`s unchanged.

**Webhook tests**: duplicate names in `overridePolicies` are rejected, and a name that does
not exist yet is accepted. With the `ManifestOverridePolicy` gate off, the validating webhook
rejects adding or changing an `overridePolicies` entry but allows other edits to an
MWRS that already has entries.

**E2E tests**: a `ManifestWorkReplicaSet` plus a `ManifestOverridePolicy` producing different
payloads on two managed clusters, including a custom resource patched with a registered
`ResourceOverrideSchema`.

**Scale**: measure hub memory and reconcile latency with the `ManagedCluster` informer at
realistic fleet size, with and without the informer `transform`, before graduation.

### Graduation Criteria

#### Alpha

- `ManifestOverridePolicy` served at `v1alpha1` behind the `ManifestOverridePolicy` feature
  gate, off by default (see [Feature gate](#feature-gate)).
- Renderer used by all three MWRS call sites; unit and integration tests above passing.
- Per-cluster failure reporting and provenance implemented.
- `ResourceOverrideSchema` and custom-resource strategic merge implemented, backed by
  explicitly registered schemas (open question 1, option a).

#### Beta

- Schema collection from managed clusters, with the content-hash-deduplicated hub cache of
  open question 1 option c, implemented and serving as the normal path. Explicit
  `ResourceOverrideSchema` registration remains available as an override.
- E2E coverage on a real multi-cluster setup.
- Scale measurements published for the `ManagedCluster` informer.
- Feature gate on by default.

#### GA

- No API changes for one release.
- User-facing documentation on the OCM website.
- A decision on open question 8, based on field experience with explicit references.

### Upgrade / Downgrade Strategy

Both new resources are optional, and the change to an existing API — one optional field on
`ManifestWorkReplicaSetSpec` — keeps today's behaviour when it is unset: an MWRS with no
`overridePolicies` renders the plain template. Nothing changes on upgrade until a user lists
a policy on an MWRS.

On downgrade, or if the feature gate is turned off, rendering reverts to the plain template;
the next reconcile rewrites each `ManifestWork` back to the un-overridden payload, which is a
real workload change and must be documented. Removing the references before downgrading
produces the same result through the normal reconcile path, with rollout strategies applied,
and is the recommended procedure.

Downgrade also drops `overridePolicies` from stored objects, so
re-enabling the feature after a downgrade does not bring the references back. Users must
re-apply them, which is one more reason to remove them deliberately first.

### Version Skew Strategy

Rendering happens entirely on the hub, and the agent receives an ordinary `ManifestWork`. An
`n-2` klusterlet that knows nothing about this feature is unaffected. If option (c) in open
question 1 is adopted later, that will introduce a genuine hub/agent contract and will need
its own skew analysis.

## Implementation History

- 2026-10-07: [enhancements#198][issue-198] opened by @kahirokunn, from
  [ocm#1175][ocm-1175].
- 2026-10-13 to 2026-10-15: discussion converges on a separate override resource rather than
  fields on MWRS; @mikeshng rejects attaching overrides to `Placement` on the grounds that
  `Placement` must stay workload-agnostic.
- 2026-10-22 to 2026-10-23: implicit application, priority ordering, per-cluster failure
  isolation and provenance agreed; templating and `forEach` moved out of scope.
- 2026-10-30: renderer prototype opened as [ocm#1792][ocm-1792].
- 2026-11-02: atomic-list rejection agreed as the default; `ResourceOverrideSchema` and the
  cross-cluster schema cache proposed; this proposal opened.
- 2026-11-03: identifier question resolved on `ResourceIdentifier` throughout, with
  `ResourceOverrideSchema` carrying the kind/version mapping so the hub can resolve types it
  does not have installed. Patch body changed from `runtime.RawExtension` to
  `apiextensionsv1.JSON` so JSON Patch arrays pass CRD validation. Deletion-semantics
  divergence between `strategicpatch` and typed merge identified and specified away. A
  `ResourceOverrideSchema` watch added to the controller wiring.
- 2026-11-11: `ResourceOverrideSchema` uniqueness defined; structural list markers required;
  effective-schema resolution defined for types the hub does not have; `overrideMode` opt-out
  added.
- 2026-11-13 to 2026-11-16: @everpeace shows a per-workload case that an all-or-nothing
  opt-out cannot express; with @kahirokunn, the proposal moves to an explicit
  `spec.overridePolicies` list with `default: true` policies added at creation, replacing
  implicit application and `overrideMode`. @kahirokunn assigned as approver.
- 2026-11-20: hub `CustomResourceDefinition` watch added to the controller wiring, since hub
  CRDs drive both resource mapping and the effective schema. Open question 7 added.
- 2026-11-20: @kahirokunn confirms the separate policy resource and strategic merge in alpha,
  resolves open question 7 with AND, and suggests a per-entry `disabled` field in place of
  `defaultOverridePolicies: Add | Skip`, so an author can skip one default and keep the rest.
- 2026-11-23: status types defined for both new resources; the `OverridesRendered` MWRS
  condition and the `override-policies` ManifestWork annotation specified; the
  `ManifestOverridePolicy` feature gate named, with webhook and controller behaviour when it
  is off; a proposed answer recorded for open question 6.
- 2026-11-25: @everpeace asks whether a default that MWRS can skip is really a default;
  @kahirokunn agrees and decides to keep alpha declarative. `default`, `disabled` and the
  mutating webhook are removed; MWRS apply exactly the policies they list. Default and
  required policies become open question 8.

## Drawbacks

The strongest argument against this proposal is cost against benefit. It adds two CRDs, a
second cluster-selection mechanism alongside `Placement`, and a `ManagedCluster` informer in
the work hub — and it still does not solve the per-copy-uniqueness need that started
[ocm#1175][ocm-1175], which needs the templating enhancement. A reviewer could reasonably
hold that the narrower MWRS-embedded alternative below delivers most of the value for a
fraction of the permanent surface, and that users who need more should use a GitOps tool.

The counter-argument is that overrides are genuinely shared across workloads — the point
@everpeace raised — and that a field inside MWRS cannot express sharing without being
restated in every MWRS. A separate resource that each MWRS references by name gives sharing
while keeping each MWRS's spec complete.

## Alternatives

### Overrides embedded in MWRS, keyed by clusterSet or placement

This is the direction of [#155][pr-155] and of review feedback on it, which suggested an
extensible `overrides` field with an `overrideStrategy` selecting between `clustersets` and
`placements`:

```yaml
overrides:
  overrideStrategy: placements
  placements:
    - name: aws-placement
      patch: ...
    - name: gke-placement
      patch: ...
```

*For:* near-zero cost — the reconciler already iterates `placementRefs` and holds the
`Placement` per iteration. No new CRD, no second selection mechanism, no new informer if
keyed by placement. Fields on an API that is still `v1alpha1`.

*Against:* overrides cannot be shared across `ManifestWorkReplicaSet`s; the same patch has
to be copied into each one.

This alternative was not chosen because of the sharing and governance arguments, even though
it is materially cheaper. @kahirokunn, as approver, confirmed the separate resource on
[#202][pr-202-kahirokunn].

### Implicit application with an `overrideMode` opt-out

The earlier draft of this proposal: every policy in the namespace applies to every MWRS,
and `overrideMode: Disabled` on an MWRS turns all of them off. Rejected because
all-or-nothing cannot express Story 4, and a per-policy deny list to fix that would make every
new policy reach every existing MWRS until each one denies it. It also hides from the MWRS
spec which policies apply to it.

### Overrides referenced from Placement

@everpeace's original sketch added an `overridePolicies` list to `Placement`, with MWRS
continuing to reference only the `Placement`. Rejected because `Placement` is used for things
other than workload distribution — selecting clusters to audit or observe, for example — and
coupling it to workload concerns narrows it.

### Templating instead of patches

Make patch values reference cluster data: `{{CLUSTER_NAME}}`, or CEL over the `ManagedCluster`
object. This addresses both needs in the motivation and removes the need to enumerate
clusters or distinct values anywhere, which is the property that fails at fleet scale. OCM has
precedent: `pkg/addon/templateagent` substitutes `{{CLUSTER_NAME}}` with `fasttemplate`, and
the policy framework offers Go templating with `lookup` and `fromClusterClaim`.

Not an alternative so much as a complement, and the thread agreed to sequence it separately.
It is listed here because it weakens the case for sharing: much of what teams would want to
share is precisely a cluster-to-value mapping, and the architecturally correct home for that
is the `ManagedCluster` object rather than a list that restates it.

### Do nothing; direct users to GitOps

`ApplicationSet` over `PlacementDecision`s already solves per-cluster customisation for users
on Argo CD. Zero cost, and it genuinely serves a large population. It does not help anyone
using MWRS directly, which is the population MWRS exists for — more relevant now that the MWRS
feature gate is enabled by default.

## References

- [enhancements#198 — Feature request: Support cluster-specific resource overrides][issue-198]
- [enhancements#155 — Add proposal for cluster set level override in MWRS][pr-155]
- [ocm#1175 — the originating request][ocm-1175]
- [ocm#1792 — renderer prototype][ocm-1792]
- [Karmada OverridePolicy](https://karmada.io/docs/userguide/scheduling/override-policy/)
- [Kustomize custom OpenAPI schema][kustomize-openapi]

[issue-198]: https://github.com/open-cluster-management-io/enhancements/issues/198
[pr-202]: https://github.com/open-cluster-management-io/enhancements/pull/202
[pr-202-everpeace]: https://github.com/open-cluster-management-io/enhancements/pull/202#issuecomment-6863619311
[pr-202-kahirokunn]: https://github.com/open-cluster-management-io/enhancements/pull/202#issuecomment-7042250849
[pr-202-kahirokunn-declarative]: https://github.com/open-cluster-management-io/enhancements/pull/202#issuecomment-7165932619
[pr-155]: https://github.com/open-cluster-management-io/enhancements/pull/155
[ocm-1175]: https://github.com/open-cluster-management-io/ocm/issues/1175
[ocm-1792]: https://github.com/open-cluster-management-io/ocm/pull/1792
[kustomize-openapi]: https://github.com/kubernetes-sigs/kustomize/blob/master/examples/customOpenAPIschema.md
