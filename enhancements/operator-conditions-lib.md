---
title: operator-conditions-lib
authors:
  - "@varshaprasad96"
reviewers:
  - "@joelanford"
  - "@estroz"
  - "@awgreene"
approvers:
  - "@joelanford"
  - "@estroz"
  - "@awgreene"
creation-date: 2020-09-28
last-updated: 2020-09-29
status: implementable
---

# Operator Conditions API Library

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

The goal of this enhancement is to define the deliverables of a library which would provide APIs for operators to access the [Status CRD][conditions_proposal] created by OLM. Using this library, operators can specify or update conditions which could influence the behavior of OLM.

## Motivation

[Operator Lifecycle Manager][olm] intends to provide operators with a channel to communicate complex states that influence its behavior while managing the operator. To do so, OLM will provide a [`Status`][conditions_proposal] Custom Resource Definition (CRD) for the operator. Based on the [conditions][conditions] set in the CR, OLM will change its behavior accordingly.

Operators need to be able to read the Status CR created by OLM, and also have the ability to add/update necessary conditions. This library will provide APIs that can:
* Get the [`NamespacedName`][cr_namespacedname] of the resource which can be used to obtain the singleton Status CR managed by OLM for their operator.
* Update the CR's `.status.conditions` array.

### Goals:

- List the APIs provided by the library.
- List the functionalities/return types of the functions present in the library.

### Non-Goals:

- Discuss on how OLM will process the conditions set in the CRD.

### User Stories

#### Story 1

As an operator author, I want my operator to be able to add/modify conditions in the `Status` CR.

#### Story 2

As an operator author, I would like to know the `NamespacedName` of the `Status` CR, which can be used to obtain the resource from cluster.

## Design Details

When an operator is installed in the cluster, OLM will create a `Status` Custom Resource for the operator. The operator will be able to perform the following actions using the APIs provided in the library:

1. Get the `NamespacedName` of the CRD:

In order to get the `Status` CRD from cluster using controller-runtimes's [client][cr_client], users will need [`ObjectKey`][cr_objectkey] of type [`types.NamespacedName`][cr_namespacedname]. OLM will set the name of Status CR as an [environment variable][conditions_followup_pr] while creating the resource. Namespace of the operator can be found from the associated [service account secret][accessing_api_from_pod] available in the operators' pod. The library will utilize both of these to obtain the `NamespacedName` for the CRD.

2. Specify conditions in the CRD:

The operators can introduce OLM-supported conditions which influence OLM's behavior or can also specify any custom ([non-OLM supported][non-olm-supported-condition]) condition for communicating the status of the operator to users. An example Status CR is given below:

```yaml
apiVersion: operators.coreos.com/v1
kind: Status
metadata:
  name: foo-operator-zdgen-asdf # RandomGen Name
  namespace: operators
status:
  conditions:
    type: Upgradeable # The name of the `foo` OLM Supported Condition.
    status: "False"   # The operator is not ready to be upgraded.
    reason: "migration"
    message: "The operator is performing a migration."
    lastTransitionTime: "2020-08-24T23:15:55Z"
```

The operator will be allowed to only modify the [`status`][status-sub-resource] sub-resource of the CR. Operators can either add, delete or update the `status.conditions` array to include OLM-supported or custom conditions. The format and description of the fields present in the conditions can be referred [here][condition_type].

### Scenarios:

1. Operator does not set/update any conditions that affect OLM:

This is a no-op. In this case, OLM will default to its normal behavior.

2. Operator tries to get/update the `Condition` CR when it is not present in the cluster:

In this case, the specific API will return an error. The returned error can be wrapped, such that users can utilize the `errors.Is` method to compare the obtained error with specific values.

3. Operator tries to get/update condition CRD which is present in cluster, but the requested CR instance is not available.

In this case, the error returned will be in the same format as stated in the previous scenario but would specifically mention that the requested object is not found.

To summarize, these are the overall helpers present in the library. Here,
- `*apiv1alpha1.Status` refers to the CR definition provided by OLM and
- [`metav1.Condition`][metav1-Condition] refers to specific Condition type present in `status.conditions` array.

```go
// GetNamespacedName - returns the NamespacedName of the CR.
// Will return error when the namespace or name of the associated
// Status CR cannot be found.
GetNamespacedName() (types.NamespacedName, error)

// SetConditionStatus - adds the specific condition to Status CR or updates the condition.Status
// if the particular condition is already present.
// Errors will be retured as specified in the `Scenarios` section.
SetConditionStatus(status *apiv1alpha1.Status, condition metav1.Condition) error

// RemoveConditionStatus - remove the specific condition present in Status CR.
// Will return an error if the specified condition was not found in Status.
RemoveConditionStatus(status *apiv1alpha1.Status, condition metav1.Condition) error

// FindConditionStatus - finds the ConditionType in `status.conditions` array of Status CR.
// Return nil if not present.
FindConditionStatus(status *apiv1alpha1.Status, conditionType string) *metav1.Condition

// IsConditionStatusTrue - returns true when the conditionType is present in `status.conditions`
// and is set to `metav1.ConditionTrue`
IsConditionStatusTrue(status *apiv1alpha1.Status, conditionType string) bool

// IsStatusConditionPresentAndEqual - returns true if conditionType is present in
// `status.conditions` and status is equal to as provided.
IsConditionStatusPresentAndEqual(status *apiv1alpha1.Status, conditionType string, conditionStatus metav1.ConditionStatus) bool
```

### Location of the library:

This library will live in [`operator-lib`][operator_lib] repository and the definitions for Status CRD will be present in [`operator-framework/api`][api_repo]. OLM will utilize the `Condition` type defined in [apimachinery][apimachinery_condition]. The existing [`status`][operator_lib_status] package present in operator-lib will be removed in favor of the available helpers to work with conditions provided in [apimachinery][apimachinery_repo].

**NOTE:**
Operators cannot CREATE/DELETE the `Status` CR. OLM will scope the operators' RBAC to only GET, UPDATE, PATCH the Status CR associated with their operator.

### Risks and Mitigations

- In order to restrict modifications on entire Status CR, OLM will restrict operators' changes to only [`status`][status-sub-resource] sub-resource, though the entire CRD can be read.

[olm]: https://github.com/operator-framework/operator-lifecycle-manager
[conditions_proposal]: https://github.com/operator-framework/enhancements/blob/master/enhancements/operator-conditions.md
[conditions]: https://github.com/kubernetes/enhancements/blob/3aa92e20bdfa2e60e3cb1a2b92cdca61847d7ad2/keps/sig-api-machinery/1623-standardize-conditions/README.md#kep-1623-standardize-conditions
[cr_client]: https://github.com/kubernetes-sigs/controller-runtime/tree/master/pkg/client
[cr_get]: https://github.com/kubernetes-sigs/controller-runtime/blob/dba75e5c7c5d39c6926307c01e402c0747bdc7ee/pkg/client/client.go#L174
[cr_objectkey]: https://github.com/kubernetes-sigs/controller-runtime/blob/dba75e5c7c5d39c6926307c01e402c0747bdc7ee/pkg/client/interfaces.go#L30
[cr_namespacedname]: https://pkg.go.dev/k8s.io/apimachinery/pkg/types#NamespacedName
[conditions_followup_pr]: https://github.com/operator-framework/enhancements/compare/master...awgreene:update-operator-status-enhancement?expand=1
[condition_type]: https://godoc.org/k8s.io/apimachinery/pkg/apis/meta/v1#Condition
[operator_lib]: https://github.com/operator-framework/operator-lib
[api_repo]: https://github.com/operator-framework/api
[operator_lib_status]: https://github.com/operator-framework/operator-lib/blob/aac7eed1886158f2b42c11ade64b9e0673a5d3d6/status/conditions.go#L53
[accessing_api_from_pod]: https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#accessing-the-api-from-a-pod
[apimachinery_condition]: https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#Condition
[apimachinery_repo]: https://github.com/kubernetes/apimachinery/tree/master/pkg/api/meta
[non-olm-supported-condition]: https://github.com/operator-framework/enhancements/blob/master/enhancements/operator-conditions.md#non-olm-supported-conditions
[status-sub-resource]: https://book-v1.book.kubebuilder.io/basics/status_subresource.html
[metav1-Condition]: https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L1367
