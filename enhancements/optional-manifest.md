---
title: optional-manifest
authors:
  - "@fgiloux"
reviewers:
  - "@njhale"
  - "@kevinrizza"
approvers:
  - "@kevinrizza"
creation-date: 2021-10-06
last-updated: 2021-10-06
status: implementable
see-also:
  - ""
replaces:
  - ""
superseded-by:
  - ""
---

# Optional Manifests

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [olm-docs](https://github.com/operator-framework/olm-docs/)

## Summary

Operator authors can ship additional resources with their operators by adding their manifest to the manifest directory inside the operator bundle. Today the creation of these resources is however mandatory for the InstallPlan, hence the installation of the operator, to succeed.
This enhancement proposal is to allow for optional resources. When a resource is marked as optional the InstallPlan can carry on with the operator installation and eventually succeed even if the creation of the aforementioned resource failed.

## Motivation

With the current implementation when the API for the resource in the manifest directory does not exist in the cluster where the operator gets installed the InstallPlan fails.
ServiceMonitor and VerticalPodAutoscaler are among the supported resources. They allow the operator authors to provide more advanced features (observability, autoscaling). These features are however not part of the core functionalities of most operators. As such the operator author may want to leverage them but not enforce them so that the operator bundle can be installed on clusters where they are not available. As these resources are not shipped with Kubernetes it is a significant drawback. It either holds operator authors of leveraging them or forces them to have additional logic directly in the operator for managing their creation.

### Goals

- Allow the specification of optional resources
- Avoid InstallPlan failure when optional resources cannot be created due to the absence of their API
- Keep the current behaviour for resources that are not specified as optional

### Non-Goals

- Although there is a relation between this enhancement proposal and optional dependencies it does not intend to cover the full scope of the later. It can however be seen as an enabler for it and aim at extending OLM in a way that is consumable by such an enhancement.

## Proposal

A resource can be marked as optional by leveraging [Explicit Bundle Properties](https://github.com/operator-framework/enhancements/blob/master/enhancements/properties.md). Therefore a new file can be added to the bundle metadata directory. The file itself can have any name, but it has to be in the top-level metadata directory of the bundle and to consist of YAML with a top-level `properties` element containing an array of type/value tuples. There should be a single file containing a top-level `properties` element.

The property type for optional manifest is `olm.manifests.optional`. Its value field is a `manifests` structure, which contains a list of optional manifests. These manifests are identified by group, kind, namespace and name. The namespace value is optional. For cluster scoped resources or resources where no namespace is specified in the manifest file the field has to be empty.

Example:

~~~
properties:
- type: 'olm.manifests.optional'
  value:
    manifests:
      - group: monitoring.coreos.com
        kind: PrometheusRule
        namespace: my-namespace
        name: my-rule
      - group: autoscaling.k8s.io/v1
        kind: VerticalPodAutoscaler
        name: my-vpa
~~~

If a manifest listed in the property file does not exist in the bundle manifest directory it will be ignored without further hints.

It does not make sense to list a resource core to OLM (ClusterServiceVersion) or resources shipped with Kubernetes under `olm.manifests.optional` as the availability of the API is a prerequisite for OLM to work. This means that no extra check will be implemented for the following types:

- ClusterServiceVersions
- Secrets
- ServiceAccounts
- Roles
- RoleBindings
- ClusterRoles
- ClusterRoleBindings
- Services
- ConfigMaps

This has the advantage of limiting the scope of the change. There is no specific logic with manifests of unknown types (types not in the above list). Should the list of allowed resources get extended possibly to any resource type no further amendment should be required to this enhancement.

### User Stories

#### Story 1: Authoring

As an operator author I want to be able to flag some of the manifests provided as optional.

#### Story 2: Consuming

As a cluster administrator and operator consumer I don't want optional resources to prevent the successful installation of an operator when they cannot get created. I don't want to have to install additional and "optional" operators that may consume resources or conflict with existing operators or policies.

### Implementation Details

#### InstallPlan

The InstallPlan needs to convey the information. Therefore, this enhancement proposal specifies the following changes to it:

+ Adding a new field "optional" at the same level as "resource"
+ Adding a new value `NotCreated` for [the status field](https://github.com/operator-framework/operator-lifecycle-manager/blob/master/vendor/github.com/operator-framework/api/pkg/operators/v1alpha1/installplan_types.go#L65-L76)

Example:

~~~
- resolving: memcached.4.8.0-202109210857
      resource:
        group: autoscaling.k8s.io
        kind: VerticalPodAutoscaler
        manifest: '{"kind":"ConfigMap","name":"cc67c20a4b985ad4e9dd0530c5c801a4a024e27c6d6feac762bbc68c7e6a83e","namespace":"openshift-marketplace","catalogSourceName":"redhat-operators"...}'
        name: memcached-operator-memcached
        sourceName: redhat-operators
        sourceNamespace: openshift-marketplace
        version: v1
      optional: true
      status: NotCreated
~~~

#### Step

The [Step structure](https://pkg.go.dev/github.com/operator-framework/api/pkg/operators/v1alpha1#Step) is what convey the InstallPlan information and therefore a field will be added to it:

~~~
// Step represents the status of an individual step in an InstallPlan.
type Step struct {
	Resolving string        `json:"resolving"`
	Resource  StepResource  `json:"resource"`
	Optional  bool          `json:"optional,omitempty"`
	Status    StepStatus    `json:"status"`
}
~~~

A new value `NotCreated` is added to the StepStatus enumeration

~~~
// StepStatus is the current status of a particular resource an in
// InstallPlan
type StepStatus string

const (
	StepStatusUnknown             StepStatus = "Unknown"
	StepStatusNotPresent          StepStatus = "NotPresent"
	StepStatusPresent             StepStatus = "Present"
	StepStatusCreated             StepStatus = "Created"
	StepStatusNotCreated          StepStatus = "NotCreated"
	StepStatusWaitingForAPI       StepStatus = "WaitingForApi"
	StepStatusUnsupportedResource StepStatus = "UnsupportedResource"
)
~~~

[NewStepsFromBundle](https://pkg.go.dev/github.com/operator-framework/operator-lifecycle-manager/pkg/controller/registry/resolver#NewStepsFromBundle) needs to be extended to set the value for the optional field of the Step structure according to the presence of the file under the `olm.manifests.optional` property.

[EnsureUnstructuredObject](https://pkg.go.dev/github.com/operator-framework/operator-lifecycle-manager/pkg/controller/operators/catalog#StepEnsurer.EnsureUnstructuredObject) which is called by [ExecutePlan](https://pkg.go.dev/github.com/operator-framework/operator-lifecycle-manager/pkg/controller/operators/catalog#Operator.ExecutePlan) in the default case will need to be amended so that it does not return an error when trying to create a manifest if the step is flagged as optional. It will instead populate the status with the new value: `StepStatusNotCreated`

#### Error handling

The steps in InstallPLans are ordered and include the resources of the dependencies:

1. ClusterServiceVersion
2. CustomResourceDefinition
3. Remaining resources in any order

Logic is built in the InstallPlan so that the creation of the CustomResourceDefinitions is completed before further steps are applied. This guarantees that all the APIs shipped with the bundle and its dependencies are available at the time when an optional manifest gets created. Based on that there is no race condition.

When OLM requests the API server to create an optional manifest there are 3 potential outcomes:

- The optional manifest is successfully created: no change to the earlier behaviour
- The API server returns an error related to the cluster specificities[1]: the step status is set to NotCreated and the InstallPlan continues. This is not seen as an error condition. The information is however surfaced through the step status: NotCreated and a warning in the logs.
- The API server does not respond, respond with an internal error or the query times out: no change to the earlier behaviour. An error is returned and the reconciliation loop tries to resync till it either succeeds or the operation times out.

[1] The logic used in OLM to differentiate when the API is not available from other errors is as follows:

~~~
if apierrors.IsNotFound(err) {
	notFoundErr := discoveryQuerier.WithStepResource(step.Resource).QueryForGVK()
	if notFoundErr != nil {
		return notFoundErr
	}
}
~~~

For optional manifests we will differentiate between errors that are specific to a cluster configuration and should not prevent the InstallPlan to succeed from errors that are either recoverable or independent of the cluster configuration. This gives for [the errors returned by the API](https://github.com/kubernetes/apimachinery/blob/release-1.22/pkg/apis/meta/v1/types.go#L729-L873):

**NotCreated**

- StatusReasonUnauthorized 401
- StatusReasonForbidden 403
- StatusReasonNotFound 404
- StatusReasonInvalid 442
- StatusReasonNotAcceptable 406
- StatusReasonUnsupportedMediaType 415
- StatusReasonConflict 409

**Failure**

- StatusReasonGone 410
- StatusReasonServerTimeout 500
- StatusReasonTimeout 504
- StatusReasonTooManyRequests 429
- StatusReasonBadRequest 400
- StatusReasonMethodNotAllowed 405
- StatusReasonRequestEntityTooLarge 413
- StatusReasonInternalError 500
- StatusReasonExpired 410
- StatusReasonServiceUnavailable 503

StatusReasonAlreadyExists 409: it is a special case as the error is [caught during object creation](https://github.com/operator-framework/operator-lifecycle-manager/blob/d6346a1c0507c765145a0b6bb18e9f06bc4058c4/pkg/controller/operators/catalog/step_ensurer.go#L307-L325) and an update is performed instead.

### Risks and Mitigations

The change is very localised so that the risks are low. The interpretation of the property `olm.manifests.optional` is also 100% backward compatible: Resources not listed under it continue to be created and mandatory for the InstallPlan to succeed. Having a file in the optional list and the operator deployed on a cluster with an OLM version not having this enhancement means that the property is just ignored and the previous behaviour stays unchanged.

The addition of a new value to the status of a step is not fully backward compatible. Clients may assume they know the complete set of values and not be able to deal with the new one. It is however important to note that InstallPlan.status.plan[].status is primarily intended for logic internal to OLM and that external clients would usually rely on the higher level InstallPlan.status.phase for getting information on the progress of an InstallPlan.

The logic in any future solution for bundle distribution, like RukPak, would need to provide a similar functionality to avoid regressing from what OLM would provide after this enhancement has been implemented.

## Design Details

### Test Plan

Code coverage for this enhancement is ensured by two set of unit tests.

- Validation that a resource is properly identified as optional based on the Properties that have been configured.
- Step processing:
	- Validation that a step flagged as optional does not cause the failure of the InstallPlan if it cannot be successfully processed due to one of the API errors listed above as NotCreated.
	- Validation that a step flagged as optional still fails the InstallPlan due to one of the API errors listed above as Failure.
	- Validation that the logic has not changed for non optional steps

### Feature gate

The optional manifest feature is currently scheduled to be available in the next release behind a feature gate, which means it is disabled by default. It will first be provided with no guarantee of API (the configuration through explicite bundle properties) stability and begins to gather users' feedbacks.

As the feature is being used and evaluated API changes can be made to improve its overall usefulness and user experience. Such changes may not be backward compatible with the first verion of its API.

After enough time has been given the feature gate may get removed and the feature available by default. Decision criteria:
- Negative feedback could be addressed
- No resilience issue introduced by the feature
- API has been stable for 3 months

If the use of the feature presents major hurdles it will be deprecated and get eventually removed.

## Implementation History

- 2021.10.06 First draft
- 2022.03.30 Second draft

## Drawbacks

This enhancement proposal is based on OLM current state where there are both explicit (dependencies.yaml) and implicit dependencies (manifest directory).
If the OLM authors decide to move away from implicit dependencies the logic may need to be reviewed.

## Alternatives

### Resource annotation

This was the original proposal. A resource provided in the manifest directory would be marked as optional by setting the following annotation:
~~~
olm.optional
~~~

**Note:**

This does not include the use of a prefix as recommended in [Kubernetes documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/#syntax-and-character-set) but is aligned with what is done elsewhere in OLM. An annotation more aligned with Kubernetes recommendations would be: `operators.coreos.com/olm.optional`

Valid values for the annotations are `true` and `false`. If the annotation is not set or set to a different value this will be interpreted as `olm.optional: false` and the resource will be mandatory.

This annotation will first be interpreted for resources that are currently allowed but non core to OLM, currently excluding:

- ClusterServiceVersions
- Secrets
- ServiceAccounts
- Roles
- RoleBindings
- ClusterRoles
- ClusterRoleBindings
- Services
- ConfigMaps

The reasoning behind the exclusion of the above list is:

- Limiting the scope of the change. There is no specific logic with manifests of unknown types. Should the list of allowed resources get extended possibly to any resource type no further amendment should be required to this enhancement.
- At the exception of ClusterServiceVersion, which is core to OLM the other resource types are shipped with Kubernetes, hence should be available on any cluster, so that there is no expected gain on making them optional as part of this enhancement request.

**Pros**:

- No additional list needs to be managed by the operator authors. An additional list can diverge from the files in the manifest directory.
- The change surface in the code is very narrow.
- It is more aligned with the workflow of operator authors that can set the property in the same file as the resource definition.
- When the resource definition is removed the property is removed as well

**Cons**:

- The list of mandatory or optional resources cannot can be seen in one place.
- The "Explicit Bundle Properties" mechanism already exists. Using a manifest annotation introduces an additional control mechanism.

### Identfication by filenames

The approach is similar to the current enhancement proposal with the difference that manifests are identified by filenames instead of /group/kind/namespace/name. This was challenged by the fact that filenames may not be available when the gRPC API is used to process bundles.

### Operator dependencies

Implementing the support of full blown operator dependencies and forcing implicit dependencies to become explicit. In that case the optionality is defined at the API level rather than at the resource level.

**Pros**:

- This provides more advanced features: the required operator and API may get conditionally installed based on some predicates.

**Cons**:

- The size of the change is way more significant.

**Note**:

Even with the optionality defined at the API level there is a need for not failing the InstallPlan when resources of the optional API can't be created due to the lack of the API. This enhancement proposal can be seen as a first step to support the more ambitious optional dependency.

### Allowing cluster administrator to skip Steps

Another approach would be to allow skipping optional Steps in the InstallPlan. Cluster administrators that have selected "manual" as approval strategy for an operator would be able to edit the InstallPlan and to mark unwanted optional Steps so that they get skipped.
Today any change made to an InstallPlan gets reverted.

**Pros**:

- This gives more control to the cluster administrators. When they review the InstallPlan they can either decide to install the missing API or to remove the optional Step

**Cons**:

- Few cluster administrators know about the possibility to set "manual" as approval strategy to inspect InstallPlans at installation time. Approval strategies are only documented in the context of operator updates.
- This is not taking the burden off the users' shoulders
