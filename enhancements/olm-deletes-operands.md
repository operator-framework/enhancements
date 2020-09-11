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

The purpose of this enhancement is to outline how the [Operator Lifecycle Manager (OLM)][olm] can let operator users opt-in to the cleanup of their operator workloads, which would be cleaned up upon the uninstallation of their operator. Specifically when uninstalling an operator, OLM would delete all CustomResource (CR) objects managed by an operator so that the operator can be triggered to clean up all the resources associated with its CRs via [Finalizers][finalizers].

## Motivation

By design when OLM uninstalls an operator it does not remove any of the operator’s owned CustomResourceDefinitions (CRDs) and CRs in order to prevent data loss.
The operator user or cluster admin is expected to manually delete CRs and any other secondary resources created by the operator.

This cleanup can be incomplete when the operator is no longer available to process deletion events of CRs and handle cleanup via finalizers.
Secondary resources that have their [OwnerReference][owner-reference] set to a CR would be garbage collected, but others such as off-cluster or cross-namespaced resources with no owner references would not be removed if the operator is already removed.

So to facilitate automatically removing operator managed resources OLM can trigger cleanup via CR removal before the operator is removed.

### Goals

- Clean up includes removal of CRs (operands) and CRDs
- CRs and CRDs are not automatically removed
- A user can opt-in to the clean up behavior at operator uninstall time

### Non-Goals

- OLM handles cleanup of anything other than CRs
- Handle custom cleanup logic like filtering/ordering the deletion of CRs

### Possible Future Goals (Not in scope)

- OLM should fire an alert event if the operator is stalled or taking too long on operand cleanup
- The timeout for firing an alert event should be user configurable
- Once [operator conditions][operator-conditions] are supported, there could be an OLM supported condition that the operator can use to indicate operand cleanup failure

## Proposal

### User Stories

#### Story 1

As an operator user I want my operator uninstall to also clean up operator-created resources so that I don't have to manually clean up any leftover resources.

### Implementation Details/Notes/Constraints

#### Where to configure cleanup: CSV and/or Subscription

The API to configure operand cleanup could be specified on either the Subscription or ClusterServiceVersion (CSV).

If the cleanup API is only present on the Subscription then operators installed without Subscriptions (by manually creating the CSV and fulfilling requirements) can not opt-in to operand cleanup.

The CSV is the more appropriate point to configure the cleanup option for an operator as the CSV represents the actual installation of the operator, whereas the Subscription only represents the intent to subscribe to updates from the catalog. Moreover, since deleting the CSV causes the operator to be removed, deleting the CSV should be the trigger to initiate operand cleanup. See the [section on CSV finalizers][finalizers-on-csvs] for more details.

#### Deleting CRs

The operand cleanup process, simply put, is the removal of all CRs being reconciled by an operator so it can run finalizers for each CR and clean up any associated operator-created resources.

From OLM's perspective these would be all CRs in the `OperatorGroup`'s [target namespaces][target-namespaces] that are of the same type as the [owned APIservices][owned-apiservices] listed in the CSV. This represents the set of all CRs that the operator reconciles and is responsible for cleaning up after.

OLM should not delete CRs of the type [required APIService][required-apiservices] as those are likely created by the operator, and whose cleanup should be handled by the operator's finalizer.

#### Finalizers on CSVs

OLM can initiate the process of operand clean up when the operator is uninstalled, which is when the CSV is deleted. However, deleting a CSV causes the operator deployment (owned by the CSV) to be garbage collected.

So in order to keep the operator around to process CR deletion events, OLM can add a finalizer to the CSV to prevent the CSV and the operator from being removed. Once OLM has completed operand cleanup and can verify the removal of all CRs, the CSV finalizer can be cleared so that the operator can be safely removed.

```YAML
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
...
metadata:
  ...
  finalizers:
  - olm/operand-cleanup
```

In the event of operand cleanup being stalled due to some operator error, OLM will not clear the CSV finalizer so that the operator stays alive indefinitely. This way the operator can be observed to figure out the required action to unblock cleanup.

#### Operand cleanup API

The CSV spec can have an optional field `spec.cleanup` which lets a user configure the cleanup behaviour before the CSV is deleted.

```YAML
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
...
spec:
  ...
  cleanup:
    enabled: false
```

With `spec.cleanup.enabled: false` OLM will not perform cleanup or delete any CRs. The finalizer on the CSV will be cleared so the CSV and operator can be removed without any cleanup. This is also the default behavior if `spec.cleanup` is unspecified to make cleanup opt-in.

With `cleanup.enabled: true` OLM will delete all CRs, for the operator's owned APIs, in the target namespaces of that operator. Once the CRs are successfully removed (after the operator clears its finalizers), OLM will clear the finalizer from the CSV to let the operator be uninstalled.

#### Operand cleanup status

To provide visibility on the status of operand cleanup, OLM can list out the CRs that are still pending deletion in the CSV status e.g `status.cleanup.pendingDeletion`.

```YAML
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
...
status:
  ...
  cleanup:
    pendingDeletion:
    - resource: etcdclusters.etcd.database.coreos.com
      kind: EtcdCluster
      version: v1beta2
      name: Foo
      namespace: Bar
    - kind: EtcdCluster
      ...
```

This information can be useful when debugging a stalled operator uninstall process to see the CRs whose removal is blocking operand cleanup.

#### CRD removal and caveats

While not strictly necessary for the goal of cleaning up operator-created resources, the cleanup process can also include removing the CRDs after the CRs and operator have been removed. There are a few edge cases consider before removing the CRD as well.

**Namespaced scoped operators:**

If there are two non-global operators installed of the same type (providing the same APIs), then removing the CRD during the cleanup of one operator would result in the deletion of CRs for the other operator as well.

To avoid this case OLM would have to ensure that it only removes the CRD during cleanup if:

- The operator installed is global
- The non-global operator is the only installation for that operator type in the cluster, i.e no other CSVs provide the same owned APIs.

**Resolver cache:**

OLM caches all the on-cluster subscriptions, catalogs, and CSVs to determine what the provided APIs are for dependency resolution (TODO: citation/link needed). It is possible that OLM could install an unusable operator if another operator that provides the required API is uninstalled from the cluster and the resolver cache is stale. The cache should be invalidated in the event of a CRD cleanup on operator uninstall.

## Design Details

### Test Plan

OLM's e2e testing suite should be upgraded to handle the following use cases:

- An operator that is uninstalled with `spec.cleanup` unspecified or `spec.cleanup.enabled: false` preserves the CRs
- An operator that is uninstalled with `spec.cleanup.enabled: true` removes all CRs
- An operator that is uninstalled with `spec.cleanup.enabled: true` should display all CRs that are pending deletion in the CSV status.

## Drawbacks

Goes against OLM's conservative approach to not remove APIs in order to prevent data loss. However this feature is opt-in so it shouldn't be a risk in the default setting.

[olm]: https://github.com/operator-framework/operator-lifecycle-manager
[owner-reference]: https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/#owners-and-dependents
[csv]: https://olm.operatorframework.io/docs/concepts/crds/clusterserviceversion/
[subscription]: https://olm.operatorframework.io/docs/concepts/crds/subscription/
[risks-section]: https://github.com/operator-framework/enhancements/blob/master/enhancements/olm-deletes-operands.md#risks-and-mitigations
[finalizers]: https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#finalizers
[operator-conditions]: https://github.com/operator-framework/enhancements/blob/master/enhancements/operator-conditions.md
[subscription-config]: https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/subscription-config.md
[target-namespaces]: https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/operatorgroups.md#target-namespace-selection
[owned-apiservices]: https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/building-your-csv.md#owned-apiservices
[required-apiservices]: https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/building-your-csv.md#required-apiservices
[finalizers-on-csvs]: https://github.com/operator-framework/enhancements/blob/master/enhancements/olm-deletes-operands.md#finalizers-on-csvs
