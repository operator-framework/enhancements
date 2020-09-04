---
title: olm-deletes-operands
authors:
  - "@hasbro17"
reviewers:
  - "@benluddy"
  - "@ecordell"
  - "@kevinrizza"
  - "@njhale"
approvers:
  - "@kevinrizza"
  - "@dmesser"
  - "@ecordell"
  - "@shawn-hurley"  
creation-date: yyyy-mm-dd
last-updated: yyyy-mm-dd
status: provisional
see-also:
  - "N/A"
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# olm-deletes-operands

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

The purpose of this enhancement is to outline how the [Operator Lifecycle Manager (OLM)][olm] can let operator users opt-in to the clean up of their operands, or operator workloads, upon the uninstallation of their operator. Specifically when uninstalling an operator, OLM would first delete all CustomResources (CRs) managed by an operator to provide the operator time to clean up all associated resources before deleting the operator itself.

## Motivation

By design when OLM uninstalls an operator it does not remove any of the operator’s owned CustomResourceDefinitions (CRDs) and CRs in order to prevent data loss.
The operator user or cluster admin is expected to manually delete CRDs, CRs and any other secondary resources created by the operator.
While some secondary resources that have their [OwnerReference][owner-reference] set to a CR would be garbage collected, others such as off-cluster or cross-namespaced resources with no owner references would remain orphaned.

Without the operator around to process deletion events of CRs, it becomes hard to cleanly remove deployed resources, especially when they exist outside of the cluster.

### Goals

- Clean up includes removal of CRs (operands) and CRDs
- Operands and CRDs are not automatically removed
- Upon operator uninstall a user can opt-in to the clean up behavior
- The operand removal should have a timeout
- After the operand removal timeout the operator should still be uninstalled
- Exceeding the operand removal timeout should create an alert event

### Non-Goals

- OLM handles cleanup of anything other than CRs and CRDs
- Handle custom cleanup logic like filtering/ordering the deletion of CRs 

## Proposal

### User Stories

#### Story 1

As an operator user I can instruct OLM to clean up all of my operator's CRs and CRDs when I uninstall my operator.

#### Story 2

As an operator user I want to see an alert event when OLM does not succeed in operand cleanup after a configured timeout.

### Implementation Details/Notes/Constraints [optional]

#### Specify cleanup via Subscription

To provide a way to opt-in for operand cleanup, the [`Subscription`][subscription] spec can include an optional field `spec.operandCleanup` which lets a user configure the cleanup behaviour.

OLM would initiate cleanup upon the deletion of a `Subscription` instance, based on `spec.operandCleanup`:

##### None (default)

With `spec.operandCleanup.type: none` OLM will not clean up operands. This is also the default behavior if `spec.operandCleanup` is unspecified to make clean up opt-in.

```YAML
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
spec:
  ...
  operandCleanup:
    type: none
```

##### OLM managed

With `operandCleanup.type: olm-managed` OLM will delete all CRs, for the operator's owned APIs, in the target namespaces (`olm.targetNamespaces`) of that operator.
Once the CRs have their finalizers cleared by the operator and are removed from the foreground, OLM will delete the CRD and then proceed with the removal of the operator.

```YAML
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
spec:
  ...
  operandCleanup:
    type: olm-managed
    timeout: 30s
```

Additionaly a timeout configured by `spec.operandCleanup.timeout` would specify how long OLM should wait for the operator to finish cleaning up.

If the cleanup timeout is exceeded, OLM should generate an alert event indicating a cleanup failure for that operator, and then proceed to uninstall the operator.

**Note:** See [Risks and Mitigations section][risks-section] on configuring the timeout.

#### Cleanup for namespace-scoped operators

If there are two non-global operators installed of the same type (providing the same APIs), then removing the CRD during the cleanup of one operator would result in the deletion of CRs for the other operator as well.

To avoid this case OLM would have to ensure that it only removes the CRD during cleanup if:

- The operator installed is global
- The non-global operator is the only installation or `Subscription` for that operator type in the cluster

#### Operator managed cleanup (not in scope for this proposal)

There could be a use case where operator authors might require custom logic for deleting CRs e.g to order the deletion, or filter certain CRs from deletion to preserve state.

For that `spec.operandCleanup` could be extended to another type, e.g `type: operator-managed`, which could allow a user to configure a job with custom logic for the deletion of CRs.
The exact workflow and UX are unclear at this time, and are left as a future extension.

## Design Details

### Test Plan

OLM's e2e testing suite should be upgraded to handle the following use cases:

- An operator that is uninstalled with `spec.operandCleanup` unspecified or `none` preserves the CRs and CRD
- An operator that is uninstalled with `spec.operandCleanup: olm-managed` removes all CRs and CRDs
- An operator that is uninstalled with `spec.operandCleanup: olm-managed` and exceeds a timeout should generate an alert event, and not block operator removal

### Risks and Mitigations

It's worth considering if the clean up timeout `spec.operandCleanup.timeout` should have a "sane" upper bound to avoid having the operator uninstall be blocked by cleanup indefinitely.
Or if users would require setting an inifinte timeout to block operator removal, if they want to inspect operator logs in the event of a cleanup failure.

Other process, e.g cluster deletion, waiting for operators to finish clean up could end up being blocked indefinitely or for a long time in this case.

## Drawbacks

Goes against OLM's conservative approach to not remove APIs in order to prevent data loss. However this feature is opt-in so it shouldn't be a risk in the default setting.

[olm]: https://github.com/operator-framework/operator-lifecycle-manager
[owner-reference]: https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/#owners-and-dependents
[subscription]: https://olm.operatorframework.io/docs/concepts/crds/subscription/
[risks-section]: https://github.com/operator-framework/enhancements/blob/master/enhancements/olm-deletes-operands.md#risks-and-mitigations