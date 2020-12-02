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
last-updated: 2020-11-23
status: implementable
---

# Operator Conditions API Library

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

The goal of this enhancement is to define the deliverables of a library which would provide APIs for operators to access the [OperatorCondition CRD][conditions_proposal] created by OLM. Using this library, operators can specify or update conditions which could influence the behavior of OLM.

## Motivation

[Operator Lifecycle Manager][olm] intends to provide operators with a channel to communicate complex states that influence its behavior while managing the operator. To do so, OLM will provide an [`OperatorCondition`][conditions_proposal] Custom Resource Definition (CRD) for the operator. Based on the [conditions][conditions] set in the CR, OLM will change its behavior accordingly.

Operators need to be able to read the OperatorCondition CR created by OLM, and also have the ability to get/update necessary conditions. This library will provide the users with the ability to:
* Get the specific condition present in the Operator Condition CR.
* Set the status of a specific condition in the Operator Condition CR.

### Goals:

- List the APIs provided by the library.
- List the functionalities/return types of the functions present in the library.

### Non-Goals:

- Discuss on how OLM will process the conditions set in the CRD.

### User Stories

#### Story 1

As an operator author, I want my operator to be able to get/update conditions in the `OperatorCondition` CR.

## Design Details

When an operator is installed in the cluster, OLM will create an `OperatorCondition` Custom Resource for the operator. The user will be expected to provide a [client][cr_client] using which the library can access the operator condition resource present in the cluster owned by the operator. The library will provide an interface, using which users can get the required condition from the CRD, or can set the status of the specific condition.

There would be a generic Conditions interface, which would in turn have the methods to `Get` and `Set` a [conditionType][condition_type_api]. For example:

```go
type Condition interface {
  // Get fetches the condition on the operator's
  // OperatorCondition. It returns an error if there are problems getting
  // the OperatorCondition object or if the specific condition type does not
  // exist.
  Get(context.Context, metav1.ConditionType) (*metav1.Condition, error)

  // Set sets the specific condition on the operator's
  // OperatorCondition to the provided status. If the condition is not
  // present, it is added to the CR.
  // To set a new condition, the user can call this method and provide optional
  // parameters if required. It returns an error if there are problems getting or
  // updating the OperatorCondition object.
  Set(context.Context, metav1.ConditionType, metav1.ConditionStatus, ...Option) error
}
```

Optional fields like `Reason` or `Message` of the condition, can be set by passing the values to `Options`.

```go
// Option is a function that applies a change to a condition.
// This can be used to set optional condition fields, like reasons
// and messages.
type Option func(*metav1.Condition)

func WithReason(reason string) Option {
	return func(c *metav1.Condition) {
		c.Reason = reason
	}
}

func WithMessage(message string) Option {
	return func(c *metav1.Condition) {
		c.Message = message
	}
}
```

The library will provide helpers to return a new `Condition` interface when a controller-runtime client and the `conditionType` is provided.

```go

// conditionAccessor contains the controller-runtime client and the namespacedName
// required to access the objects on cluster.
type conditionAccessor struct {
  client Client.client
  namespacedName types.NamespacedName
}

// condition contains the conditionAccessor and the conditionType which
// needs to be added/modified.
type condition struct {
  conditionAccessor conditionAccessor
  conditionType metav1.ConditionType
}

// condition will implement the Set and Get methods defined in Condition interface.
var _ Condition = &condition{}

// NewCondition will return a new Condition to access the condition of the specified
// conditionType from the CR using the client provided by the user. This method will
// internally obtain the name of the condition from the environment variable set by OLM and the
// namespace from the associated service account secret.
func NewCondition(cl client.Client, conditionType metav1.ConditionType) (Condition, error) {
  // return a new conditionAccessor with the client provided by the user and the namespacedName
  // obtained from the env variables. An error is returned if the namespacedName cannot be found.
  c, err := NewConditions(cl)
  if err != nil {
    return nil, err
  }
  return &condition{conditionAccessor: c, conditionType: conditionType}
}
```

This will enable users to write their own constructor which accepts a controller-runtime client, a conditionType and returns
a Condition interface to update/add conditions. OLM currently supports the [`Upgradable`][olm_upgradable] condition, and hence
creating an interface which has methods to access `Upgradable` condition would become as simple as:

```go
func NewUpgradable(cl client.Client) (Condition, error) {
	return NewCondition(cl, "Upgradable")
}
```

Implementation for:

1. `Get`:

In order to get the specific condition from Operator Condition CRD, the library will use controller-runtimes's [`client.Get`][cr_client_get] which requires an [`ObjectKey`][cr_objectkey] of type [`types.NamespacedName`][cr_namespacedname] which is present in
`conditionAccessor`.

2. `Set`:

The library will use [`client.Update`][cr_client_update] to update the status of the specific condition. An error will occur if the conditionType is not present in the CRD.

The operator will be allowed to only modify the [`status`][status-sub-resource] sub-resource of the CR. Operators can either delete or update the `status.conditions` array to include the condition. The format and description of the fields present in the conditions can be referred [here][condition_type].

### Scenarios:

1. Operator does not set/update any conditions that affect OLM:

This is a no-op. In this case, OLM will default to its normal behavior.

2. Operator tries to get/update the `Condition` CR when it is not present in the cluster:

In this case, the specific API will return an error. The returned error can be wrapped, such that users can utilize the `errors.Is` method to compare the obtained error with specific values.

3. Operator tries to get/update condition CRD which is present in cluster, but the requested CR instance is not available.

In this case, the error returned will be in the same format as stated in the previous scenario but would specifically mention that the requested object is not found.

### Advantage of using this format:
1. Users need not import `operator-framework/api` to modify the conditions in the CR.
2. Users can create a new interface which implements `Condition` for custom condition types that are not supported by OLM.

### Disadvantages of using this format:
1. If the users want to utilize the helpers provided in [apimachinery][apimachinery_condition], they may not have access to the name and namespace of the operator conditions CR which is being used in the library.

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
[status-sub-resource]: https://book-v1.book.kubebuilder.io/basics/status_subresource.html
[condition_type_api]: https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#Condition.Type
[olm_upgradable]: https://github.com/operator-framework/enhancements/blob/master/enhancements/operator-conditions.md#upgradeable
[cr_client_get]: https://github.com/kubernetes-sigs/controller-runtime/blob/v0.6.4/pkg/client/client.go#L187
[cr_client_update]: https://github.com/kubernetes-sigs/controller-runtime/blob/v0.6.4/pkg/client/client.go#L137
