---
title: operator-conditions
authors:
  - "@awgreene"
reviewers:
  - "@ecordell"
approvers:
  - "@kevinrizza"
creation-date: 2020-08-24
last-updated: 2020-08-24
status: implementable
see-also:
  - "N/A"  
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# operator-conditions

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

The goal of this enhancement is to allow operators managed by the [Operator Lifecycle Manager (OLM)](https://github.com/operator-framework/operator-lifecycle-manager) operator to communicate conditions that OLM would otherwise be unable to infer. In particular, these conditions will be used by OLM to prevent irreversible actions (such as upgrades) when the operator is in a state that would be problematic.

## Motivation

As part of its role in managing the lifecycle of an operator, OLM infers the state of an operator from the state of Kubernetes resources that define the operator. While this sanity-check provides some level of assurance that an operator is in a given state, there are many complex states specific to each operator that OLM is unable to infer on its own. OLM could better handle the lifecycle of operators by providing them the means to communicate complex states. In particular, there are a number of benefits if an operator managed by OLM could communicate when it should not be upgraded. Consider the following scenario:

>An operator installed with automatic updates is currently performing a migration that could result in data loss if OLM upgrades the operator. While this migration process is running, a new version of the operator becomes available. Since the operator was installed with automatic updates, OLM upgrades the operator without allowing the migration process to finish, resulting in unrecoverable data loss. Had OLM given the operator the ability to communicate that it should not be upgraded, no data loss would have occurred.

As shown in the example above, by allowing operators to report complex condition and influence OLM's management, the overall user experience could be improved for both operator authors and OLM users.

### Goals

- Provide operators with the means to explicitly communicate conditions that influence OLM behavior.
- Allow the list of "OLM Supported Conditions" to grow in the future.
- Update OLM to support the "Upgrade Readiness" condition defined above.
- Allow operators to communicate additional information to admins without influencing OLM behavior.
- Preserve existing behavior if the operator does not take advantage of this feature.

### Non-Goals

- Provide operators with the means to cancel an upgrade that is in process.

## Proposal

### User Stories

#### Story 1

As an operator author, I want operators managed by OLM to be able to communicate complex states that OLM cannot infer on its own.

#### Story 2

As an operator author, I want operators managed by OLM to be able to communicate whether or not they are ready to be upgraded.

#### Story 3

As an operator author, I want operators managed by OLM to be able to communicate whether or not they are currently upgrading or if they successfully completed the upgrade process.

## Design Details

### Condition CRD

OLM will create a channel for operators to communicate complex conditions by introducing the `Condition` [CustomResourceDefinition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/).

When an operator is installed, OLM will create a `Condition` CustomResource (CR) for the operator. The operator should only be concerned with updating the status of the `Condition` that it owns and will have its RBAC scoped so it may only GET, LIST, or UPDATE the `Condition` CR it is associated with.

The `Condition` CR will be created with the [aggregation label](https://github.com/openshift/enhancements/blob/master/enhancements/olm/simplify-apis.md#component-selection) introduced for the `Operator` CRD that will associate it with a specific operator. When OLM deploys the operator, an additional environment variable will be appended to the deployment. Operators will then be able to use this label along with a library provided by the [operator-sdk](https://github.com/operator-framework/operator-sdk) team to set [conditions](https://github.com/kubernetes/enhancements/blob/3aa92e20bdfa2e60e3cb1a2b92cdca61847d7ad2/keps/sig-api-machinery/1623-standardize-conditions/README.md#kep-1623-standardize-conditions) on the CR to communicate complex states to OLM. For those unfamiliar with conditions, they are used by many Kubernetes Resources to provide users with additional insight into the state of the resource and typically adhere to the following format:

| Field | Description |
| ----- | ----------- |
| Type | Type of condition in CamelCase or in foo.example.com/CamelCase. |
| Status | Status of the condition, one of True, False, Unknown. |
| ObservedGeneration | If set, this represents the .metadata.generation that the condition was set based upon. |
| LastTransitionTime | Last time the condition transitioned from one status to another. |
| Reason | The reason for the condition's last transition in CamelCase. |
| Message | A human readable message indicating details about the transition. |

OLM will define a set of "OLM Supported Conditions" which will influence how OLM manages the Lifecycle of the operator when present.

### OLM Supported Conditions

All "OLM Supported Conditions" will be communicated by setting the conditions in the `status.conditions` array of the `Condition` CR associated with the operator. The name of each "OLM Supported Condition" will correspond to the `Type` field of the condition added to the array. Let's look at a fake "OLM Supported Condition" named `foo` that has been added to an `Condition` CR:

```yaml
apiVersion: operators.coreos.com/v1
kind: Condition
metadata:
  name: foo-operator-zdgen-asdf # RandomGen Name
  namespace: operators
status:
   conditions:
    type: Foo # The name of the `Foo` OLM Supported Condition.
    status: "True" # The "status" of the `foo` OLM Supported Condition.
    reason: "foo" # A camelCase reason that the operator is in this state
    message: "The operator is in the "foo" state." # A human readable message.
    lastTransitionTime: "2020-08-24T23:15:55Z" # The last time the condition transitioned from one status to another.
```

#### Upgradeable

The `Upgradeable` "OLM Supported Condition" allows an operator to communicate when OLM should or should not upgrade the operator.

When the `Upgradeable` condition is set to `False`, OLM will not:

- Perform automatic or manual updates to the operator.
- Update the operator as a part of [dependency resolution](./operator-dependency-resolution.md).

When the `Upgradeable` condition is set to `True`, OLM will not prevent upgrades to the operator.

The `Upgradeable` condition might be useful when:

- An operator is about to start a critical process and should not be upgraded until after the process is completed.
- The operator is performing a migration of CRs that must be completed before the operator is ready to be upgraded.

##### Example Condition with the Upgradeable Condition

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

Given that the `Upgradable Condition`'s status is set to `False`, OLM will understand that it should not upgrade the operator.

#### Upgrading

The `Upgrading` "OLM Supported Condition" allows an operator to communicate if it's now performing upgrade to a newer version.

When the `Upgrading` condition exists, OLM will use it to find out if the operator completed the upgrade process, instead of the readiness probe.

If the `Upgrading` condition is not exists in the CR, OLM will keep the current behaviour of detecting the upgrade status.

##### Open Issues
- **Issue**: How can OLM distinct between an upgrade process that was not started yet, and one that was already completed.

  **Suggestion**: Add a version field to the operator configuration in the CSV. Add a version field to the condition CR. 
  The operator will update this new version filed only in two cases: on a clean deployment, or upon completion of successful 
  upgrade.
  
  OLM will check the CSV version with the CR version field in addition to the `Upgrading=false` in order to make sure  the upgrade was completed.
  
  For example: the CR represents an operator that is now upgrading from version v1.2.3 to version v2.3.4:
  ```yaml
  apiVersion: operators.coreos.com/v1
  kind: Condition
  metadata:
    name: foo-operator-zdgen-asdf # RandomGen Name
    namespace: operators
  status:
    conditions:
    - type: "Upgradeable" # The name of the `foo` OLM Supported Condition.
      ...
    - type: "Upgrading" # The name of the `foo` OLM Supported Condition.
      status: "True"   # The operator is now performing upgrade.
      reason: "migration"
      message: "The operator is performing a migration to version v2.3.4"
      lastTransitionTime: "2020-08-24T23:15:55Z"
    version: "v1.2.3" 
  ```
 
##### Example Condition with the Upgrading Condition

```yaml
apiVersion: operators.coreos.com/v1
kind: Condition
metadata:
  name: foo-operator-zdgen-asdf # RandomGen Name
  namespace: operators
status:
  conditions:
  - type: "Upgradeable" # The name of the `foo` OLM Supported Condition.
    ...
  - type: "Upgrading" # The name of the `foo` OLM Supported Condition.
    status: "True"   # The operator is now performing upgrade.
    reason: "migration"
    message: "The operator is performing a migration to version v2.3.4"
    lastTransitionTime: "2020-08-24T23:15:55Z"
```

```yaml
apiVersion: operators.coreos.com/v1
kind: Condition
metadata:
  name: foo-operator-zdgen-asdf # RandomGen Name
  namespace: operators
status:
  conditions:
  - type: "Upgradeable" # The name of the `foo` OLM Supported Condition.
    ...
  - type: "Upgrading" # The name of the `foo` OLM Supported Condition.
    status: "False"   # The operator is not performing upgrade right now
    reason: "Upgrade Done"
    message: "Sucessfully upgraded to version v2.3.4"
    lastTransitionTime: "2020-08-24T23:15:55Z"
```

### Non-OLM Supported Conditions

There may be instances where an Operator may want to communicate a condition to users that isn't known by OLM or expected to change OLM behavior. Operators may communicate these "Non-OLM Supported Conditions" by appending the conditions to the `status.conditions` array of the `Condition` CR provided that the `Type` of the condition does not intersect with any "OLM Supported Condition" `Types`.

### Overriding an OLM Supported Condition

There are times as a Cluster Admin that you may want to ignore an "OLM Supported Condition" reported by an Operator. For example, imagine that a known version of an operator always communicates that it is not upgradeable. In this instance, you may want to upgrade the operator despite the operator communicating that it is not upgradeable. This could be accomplished by overriding the `OLM Supported Condition` by adding the condition's name to the `spec.overrides` array in the `Condition` CR:

```yaml
apiVersion: operators.coreos.com/v1
kind: Condition
metadata:
  name: foo-operator-zdgen-asdf # RandomGen Name
  namespace: operators
spec:
  overrides:
    - Upgradeable # Informs OLM to ignore the Upgradeable "OLM Supported Condition".
    - Foo # Informs OLM to ignore the foo "OLM Supported Condition".
status:
   conditions:
    type: Upgradeable
    status: "False"
    reason: "migration"
    message: "The operator is performing a migration."
    lastTransitionTime: "2020-08-24T23:15:55Z"
```

Since the `Upgradeable` condition is being overridden, OLM will allow the operator to be upgraded despite the `Upgradeable Condition`'s status being set to `False`.

### Managing the Condition CR

As outlined earlier in this enhancement, the Operator SDK team will provide a library that operators can use to manage conditions on their `Condition` CR. The functions provided within the library will detect if the `Condition` CRD is present on the cluster using the Discovery API. If the API is not present, the functions will perform no-ops.

At the time of this enhancement, the operator will only need to manage its `Upgradeable` condition.

### Test Plan

OLM's e2e testing suite will be upgraded to handle the following use cases:

- If an operator does not have an associated `Condition` CR, OLM reverts to existing behavior.
- If an operator has an associated `Condition` CR:
  - Upgrades are not blocked if the `Upgradeable` "OLM Supported Condition" is not present in the `status.conditions` array.
  - Upgrades are not blocked if the `Upgradeable` "OLM Supported Condition" is present and the condition's status is `True`.
  - Upgrades are blocked if the `Upgradeable` "OLM Supported Condition" is present and the condition's status is `False`.

### Risks and Mitigations

- Operators may not opt-in to this feature, so existing behavior must be preserved in this case.
- Operators should only be concerned with updating the status of the `Condition` CR that is associated with the Operator. OLM will scope the operators RBAC so it may only identify the `Condition` CR associated with the operator and will not provide the operator will CREATE or DELETE verbs.
