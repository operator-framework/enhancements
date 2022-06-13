---
title: feature-gate
authors:
  - "@fgiloux"
reviewers:
  - "@awgreene"
  - "@joelanford"
  - "@perdasilva"
approvers:
  - "@perdasilva"
creation-date: 2022-04-30
last-updated: 2022-04-30
status: implementable
see-also:
  - ""
replaces:
  - ""
superseded-by:
  - ""
---

# Feature Gate

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [olm-docs](https://github.com/operator-framework/olm-docs/)

## Summary

Feature gates are a set of key/value pairs designed for enabling/disabling a specific OLM feature.
This enhancement proposal suggests to follow the [feature stages defined in Kubernetes](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#feature-stages) and to make feature gates configurable in [OLMConfig](https://olm.operatorframework.io/docs/advanced-tasks/configuring-olm/) through a generic API.

## Motivation

[Feature Toggles](https://martinfowler.com/articles/feature-toggles.html) are a powerful technique allowing the modification of the behaviour of a system by activating new capabilities without modification of the code. Kubernetes feature gates are an implementation of this pattern.

This enhancement proposal aims at bringing the following benefits to OLM:

- A more phased approach of releasing features.
- An API allowing to make features at different phases available with transparence on the maturity level of the features, not jeopardizing the stability of production environments.
- The possibility to make users aware and experimenting with new development early.
- Getting earlier feedback on new features from the large community.
- The possibility for users to easily and quickly revert the activation of a feature.

### Goals

- To have a simple and generic API for feature gate .
- Putting new features behind a gate
  - does not require any API anymore
  - requires minimal code modifications: adding the feature to a list and testing its activation in the feature implementation
- Having a mechanism that does not make the complete cluster unsupported.
- Having a clear lifecycle for feature gates.

### Non-Goals

- Putting existing features behind a gate.

## Proposal

This enhancement proposal suggests to extend the OLMConfig API to have a generic mechanism for activating and deactivating optional features through a feature gate. This is inspired by the feature gate mechanism available in Kubernetes and will follow the [feature stages defined in Kubernetes](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#feature-stages) for the lifecycle. Taken directly from the Kubernetes documentation:

A feature can be in Alpha, Beta or GA stage.

An Alpha feature means:

- Disabled by default.
- Might be buggy. Enabling the feature may expose bugs.
- Support for feature may be dropped at any time without notice.
- The API may change in incompatible ways in a later software release without notice.
- Recommended for use only in short-lived testing clusters, due to increased risk of bugs and lack of long-term support.

A Beta feature means:

- Enabled by default.
- The feature is well tested. Enabling the feature is considered safe.
- Support for the overall feature will not be dropped, though details may change.
- The schema and/or semantics of objects may change in incompatible ways in a subsequent beta or stable release. When this happens, we will provide instructions for migrating to the next version. This may require deleting, editing, and re-creating API objects. The editing process may require some thought. This may require downtime for applications that rely on the feature.
- Recommended for only non-business-critical uses because of potential for incompatible changes in subsequent releases. If you have multiple clusters that can be upgraded independently, you may be able to relax this restriction.

A General Availability (GA) feature is also referred to as a stable feature. It means:

- The feature is always enabled; you cannot disable it.
- The corresponding feature gate is no longer needed.
- Stable versions of features will appear in released software for many subsequent versions.

[Kubernetes deprecation rules](https://kubernetes.io/docs/reference/using-api/deprecation-policy/#deprecating-a-feature-or-behavior) will also be followed: Feature gates are intended to cover the development life cycle of a feature - they are not intended to be long-term APIs. As such, they are expected to be deprecated and removed after a feature becomes GA or is dropped. Features can be removed at any point in the life cycle prior to GA. When features are removed prior to GA, their associated feature gates are also deprecated.

Feature gates must be deprecated when the corresponding feature they control transitions a lifecycle stage as follows. Feature gates must function for no less than:

- Beta feature to GA: 6 months or 2 releases (whichever is longer).
- Beta feature to EOL: 3 months or 1 release (whichever is longer).
- Alpha feature to EOL: 0 releases.

Deprecated feature gates must respond with a warning when used. When a feature gate is deprecated it must be documented in the release notes.

The proposed OLMConfig API extension is as follows:

~~~
spec:
  type: object
    properties:
      features:
        description: Features contains the list of configurable OLM features.
        type: object
        properties:
          disableCopiedCSVs:
            description: DisableCopiedCSVs is used to disable OLM's "Copied CSV" feature for operators installed at the cluster scope, where a cluster scoped operator is one that has been installed in an OperatorGroup that targets all namespaces. When reenabled, OLM will recreate the "Copied CSVs" for each cluster scoped operator.
            type: boolean
          gates:
            description: gates are used to activate/deactivate features.
            type: array
            items:
              description: A gate is a name/value pair that contains whether a specific feature is enabled or not.
              type: object
              required:
                - name
                - value
              properties:
                name:
                  description: name identifies a feature.
                  type: string
                value:
                  description: value is a boolean that specifies whether the feature is activated or not.
                  type: boolean
~~~


> **Note**
> disableCopiedCSVs is an existing feature and not part of this enhancement proposal. It predates this enhancement proposal and gets further discussed under [Implementation Details](#DisableCopiedCSVs).

Here is a configuration example:

~~~
spec:
  features:
    disableCopiedCSVs: true
    activation/gates
      - name: disableCopiedCSVs
        value: true
      - name: enableOptionalManifest
        value: true
~~~

### User Stories

#### Stories: Authoring

- As an author I want to be able to set a feature behind a gate without requiring any API change and minimal code changes.

- As an author I want to get feedback on my feature implementation before it has fully matured.

- As an author I want to get feedback on my feature implementation before I commit to its stability in API and behaviour.

- As an author I want to clearly set the expecations in terms of feature maturity and don't want features that are not mature to be activated by default.

#### Stories: Consuming

- As a user I want to understand the product development direction and its impact on my use of it hands-on.

- As a cluster operator I want to avoid features that could lower the stability or consistency of my production clusters.

- As a cluster operator I don't want to rely on features that are not yet mature.

- As a cluster operator I want to be able to quickly deactivate a feature when it has been identified that it lowers the stability or consistency of my clusters.

### Implementation Details

#### OLMConfig

A new version v2alpha1 of the OLMConfig API will be created and supported along v1. This version will introduce a list named `gates` for activation and deactivation of features:

~~~
spec:
  type: object
    properties:
      features:
        description: Features contains the list of configurable OLM features.
        type: object
        properties:
          disableCopiedCSVs:
            description: DisableCopiedCSVs is used to disable OLM's "Copied CSV" feature for operators installed at the cluster scope, where a cluster scoped operator is one that has been installed in an OperatorGroup that targets all namespaces. When reenabled, OLM will recreate the "Copied CSVs" for each cluster scoped operator.
            type: boolean
          gates:
            description: gates are used to activate/deactivate features.
            type: array
            items:
              description: A gate is a name/value pair that contains whether a specific feature is enabled or not.
              type: object
              required:
                - name
                - value
              properties:
                name:
                  description: name identifies a feature.
                  type: string
                value:
                  description: value is a boolean that specifies whether the feature is activated or not.
                  type: boolean
~~~

The internal structure will look like this:
~~~
// Features contains the list of configurable OLM features.
type Features struct {
	DisableCopiedCSVs *bool `json:"disableCopiedCSVs,omitempty"`
  // +optional
	Activations []FeatureActivation `json:"activations,omitempty" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,2,rep,name=activations"`
}

type FeatureActivation struct { or type FeatureGate struct {
  // +patchMergeKey=name
  // +patchStrategy=merge
  Name string `json:"name" protobuf:"bytes,1,opt,name=name"`
  Value string `json:"value" protobuf:"bytes,2,opt,name=value"`
}
~~~

#### Feature gate

The feature gate implementation will leverage existing [Kubernetes libraries](https://pkg.go.dev/k8s.io/component-base/featuregate).

#### DisableCopiedCSVs

As DisableCopiedCSVs already exists. As a first class field its support needs to be guaranteed. As such the API field will be kept and deprecated but the implementation will convert it to the map key/value pair and will handle it in a way consistent with the other feature gates.

The implementation will also move away from a logic where [OLMConfig is queried](https://github.com/operator-framework/operator-lifecycle-manager/pull/2466/files#diff-1c45679686ec7da9a4e3c02d009d84f465130fec2278b9512d27d53e23224f3aR1455-R1468) to a logic where OLMConfig changes are reconciled and reflected in a [featureGate structure](https://github.com/kubernetes/component-base/blob/master/featuregate/feature_gate.go)

There are now two ways of setting disableCopiedCSVs. Although the former field will be deprecated (and a deprecation warning raised) backward and forward compatibility need to be ensured. Therefore [Kubernetes recommendations for changing APIs](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md#making-a-singular-field-plural), making a singular field plural, will be followed:

Upon any read operation
- If only the former field is set the controller will set the newer one to the same value.

Upon any create operation
- if only the former field is specified (e.g. an older client), API logic must populate the new element as a one-element list, which contains the value of the former field. Rationale: It's an old client and they get compatible behavior.
- if both the old and new fields are specified, API logic must validate that they match.
- Any other case is an error and must be rejected. This includes the case of the `gates` field being specified and `disableCopiedCSVs` not. Rationale: In an update, it's impossible to tell the difference between an old client clearing the former field via patch and a new client setting the new field. For compatibility, we must assume the former, and we don't want update semantics to differ from create.

Upon any update operation (including patch):
- If old field is cleared and new field is not changed, API logic must clear new field. Rationale: It's an old client clearing the field it knows about.
- If new field is cleared and old field is not changed, API logic must populate the new field with the same value as the old one. Rationale: It's an old client which can't send fields it doesn't know about.
- If the old field is changed (but not cleared) and the new field is not changed, API logic must populate the new field, with the value of the old field. Rationale: It's an old client changing the field they know about.

A function like the one described in Kubernetes API changes will be implemented:

~~~
func normalizeFeatures(after, before *api.Features)
~~~

### Risks and Mitigations

The change is very localised so that the risks are low. 

The logic in any future solution for bundle distribution, like RukPak, would need to provide a similar functionality to avoid regressing from what OLM would provide after this enhancement has been implemented.

## Design Details

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:
- Maturity levels - `Dev Preview`, `Tech Preview`, `GA`
- Deprecation

Clearly define what graduation means.

#### Examples

These are generalized examples to consider, in addition to the aforementioned
[maturity levels][maturity-levels].

##### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers

##### Tech Preview -> GA

- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

##### Removing a deprecated feature

- Announce deprecation and support policy of the existing feature.
- Deprecate the feature.

### Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to make use of the enhancement?

### Version Skew Strategy

How will the component handle version skew with other components?
What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- During an upgrade, we will always have skew among components, how will this impact your work?
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI, CRI
  or CNI may require updating that component before the kubelet.

## Implementation History

2022.05.27 First draft

## Drawbacks

As OLMConfig is not an Alpha or Beta but a GA API changes require careful handling and [the deprecated API must not be removed within a major version of Kubernetes](https://kubernetes.io/docs/reference/using-api/deprecation-policy/).

## Alternatives

### Using a Map

The proposed OLMConfig API extension would be as follows:

~~~
spec:
  type: object
    properties:
      features:
        description: Features is a map of {key,value} containing the list of OLM feature gates. The "key" field identifies a feature. The "value" fields specifies whether it is activated or not.
        type: object
        additionalProperties:
          type: string
~~~

Here is a configuration example:

~~~
spec:
  features:
    disableCopiedCSVs: true
    enableOptionalManifest: true
~~~

The implementation side will still require changes. The `features` field would then translate to a Map:

~~~
Features map[string]bool `json:"features,omitempty"`
~~~

#### Pros

- From a client point of view there is "almost" no change to the existing API. As today OLMConfig is only used for activating disableCopiedCSVs the same yaml would still work, which is GitOps friendly. It would however be an incompatible change for strongly typed clients doing schema validation.
- Excluding the caveat mentioned above there is no need for an API extension, hence a new version.

#### Cons

- Incompatible change for strongly typed clients doing schema validation.
- The `features` field gets then fully used for feature gates. It is not the case with `disableCopiedCSVs` but it is conceivable that future features may support configuration parameters. In that respect `features` is not a good name and `featureGates` would be more appropriate for the specifics of feature activation/deactivation.
- [Using maps is actually discouraged in Kubernetes](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#lists-of-named-subobjects-preferred-over-maps) for anything other than labels, annotations and like. Kubernetes favours strong typing and [disambiguity on field names versus configurable values](https://github.com/kubernetes/kubernetes/issues/2004#issuecomment-60641437).

