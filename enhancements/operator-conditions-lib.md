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

The goal of this enhancement is to define the deliverables of a library which would provide APIs for operators to access the [Condition CRD][conditions_proposal] created by OLM. Using this library, operators can specify or update conditions which could influence the behavior of OLM.

## Motivation

[Operator Lifecycle Manager][olm] intends to provide operators with a channel to communicate complex states that influence its behavior while managing the operator. To do so, OLM will provide a [`Condition`][conditions_proposal] Custom Resource Definition (CRD) for the operator. Based on the [conditions][conditions] set in the CR, OLM will change its behavior accordingly.

Operators need to be able to read the Condition CR created by OLM, and also have the ability to add/update necessary conditions. This library will provide APIs, using which users can read the Condition CRD present in the cluster for their operator and modify the `.status.conditions` array to add or update any specific condition.

### Goals:

- List the deliverables of the library.

### Non-Goals:

- Discuss on how OLM will process the conditions set in the CRD.
- List the function signatures/interface types which will be present in the library.

### User Stories

#### Story 1

As an operator author, I want my operator to be able to add/modify conditions in the `Condition CR`.

#### Story 2

As an operator author, I would like to have the flexibility of being able to read the `Condition CR` and decide to not set any conditions, thereby opting for default OLM behavior.

## Design Details

When an operator is installed in the cluster, OLM will create a `Condition` Custom Resource for the operator. The operator will be able to perform the following actions using the APIs provided in the library:

1. Read the `Condition` CRD:

The library will use controller-runtime's [client][cr_client] to access the resources present in the cluster. To fetch the resource, we will use the [`Get`][cr_get] method of the client, which requires the [`ObjectKey`][cr_objectkey] of type [`types.NamespacedName`][cr_namespacedname]. OLM will set the name of Condition CR as an [environment variable][conditions_followup_pr] while creating the resource. Namespace of the operator can be found from the associated [service account secret][accessing_api_from_pod] available in the operators' pod. The library will utilize both of these to obtain the CRD from cluster.

2. Specify conditions in the CRD:

The operators can introduce OLM-supported conditions which influence OLM's behavior or can also specify any custom (non-OLM supported) condition for communicating the status of the operator to users. An example Condition CR is given below:

```yaml
apiVersion: operators.coreos.com/v1
kind: Condition
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

The operator will be allowed to only modify the `status` field of condition CR. Operators can either add, delete or update the `status.conditions` array to include OLM-supported or custom conditions. The format and description of the fields present in the conditions can be referred [here][condition_type].

The library will live in [`operator-lib`][operator_lib] repository and the definitions for Condition CRD will be present in [`operator-framwork/api`][api_repo]. The helper methods to work with [`conditions`][operator-lib-condition], which are present in operator-lib will be utilized to build APIs that will operate on the `Condition` CRD provided by OLM and perform necessary actions.

#### Scenarios:

1. Operator does not set/update any conditions that affect OLM:

This is a no-op. In this case, OLM will default to its normal behavior.

2. Operator tries to get/update condition CRD when it is not present in the cluster:

In this case, the specific operation will return an error. The returned error can be wrapped, such that users can utilize the `errors.Is` method to compare the obtained error with specific values.

**NOTE:**
Operators cannot CREATE/DELETE the `Condition CR`. OLM will scope the operators' RBAC to only GET, LIST, UPDATE the Condition CR associated with their operator.

### Risks and Mitigations

- In order to restrict modifications on entire Condition CR, OLM will restrict operators' changes to only `status.conditions` field, though the entire CRD can be read.

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
[operator-lib-condition]: https://github.com/operator-framework/operator-lib/blob/aac7eed1886158f2b42c11ade64b9e0673a5d3d6/status/conditions.go#L53
[accessing_api_from_pod]: https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#accessing-the-api-from-a-pod
