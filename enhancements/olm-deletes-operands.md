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

The purpose of this enhancement is to outline how the [Operator Lifecycle Manager (OLM)][olm] can let operator users opt-in to the cleanup of their operator workloads, which would be cleaned up upon the uninstallation of their operator. Specifically when uninstalling an operator, OLM would delete all CustomResource (CR) objects (operands) managed by an operator so that the operator can be triggered to clean up all the resources associated with its CRs via [Finalizers][finalizers].

## Motivation

### Resource model and cleanup

Per the Kubernetes [controller guidelines][controller-guidelines] the typical resource heirarchy for an operator should be a primary resource (usually a CR) that is reconciled, and secondary resources (Pods, Deployments, other CRs) that the operator creates and manages for the primary resource. The secondary resources should have their [OwnerReference][owner-reference] set to the primary resource so that when the root CR is deleted, the cleanup of all its owned resources is automatically handled by the Kubernetes garbage collector.

If there are secondary resources that can't be owned by the primary CR, such as off-cluster or cross-namespaced resources, then the operator needs to handle cleanup of those resources by processing deletion events of the primary CR via Finalizers.

### OLM's role in assisting cleanup

By design when OLM uninstalls an operator it does not remove any of the operator’s owned CustomResourceDefinitions (CRDs) and CRs in order to prevent data loss.
The operator user is expected to manually delete the CRs that they've created.

However operators that do have cleanup logic implemented via Finalizers can benefit from an automatic cleanup of all their managed resources when the operator is uninstalled. This cleanup of CRs and resources would have to happen before the operator is removed so it can process the deletion events of its CRs.  

To facilitate an operator's cleanup logic, OLM can delete any CRs provided by an operator before removing it.

## Goals

- Cleanup includes removal of CRs and CRDs for provided APIs
- Cleanup of CRs and CRDs is not enabled by default
- A user can opt-in to the clean up behavior before operator uninstall time

## Non-Goals

- OLM handles cleanup of anything other than CRs
- Handle custom cleanup logic like filtering/ordering the deletion of CRs

## Possible Future Goals (Not in scope)

- OLM should fire an alert event if the operator is stalled or taking too long on operand cleanup
- The timeout for firing an alert event should be user configurable
- Once [operator conditions][operator-conditions] are supported, there could be an OLM supported condition that the operator can use to indicate operand cleanup failure
- The `operator-sdk` and `kubectl operator` can use OLM's operand cleanup feature instead of [deleting the CRDs][kb-operator-crd-delete] manually.
- OLM handles cleanup of [Owned APIServices][owned-apiservices]

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

From OLM's perspective these would be all CRs in the `OperatorGroup`'s [target namespaces][target-namespaces] that are of the same type as the [owned CRDs][owned-crds] listed in the CSV. This represents the set of all CRs that the operator reconciles and is responsible for cleaning up after.

OLM should not delete CRs of the type [required CRDs][required-crds] as those are likely created by the operator, and whose cleanup should be handled by the operator's finalizer.

The CRs must directly be deleted instead of only deleting the CRD and relying on garbage collection of CRs because of the complication that causes for namespace scoped operators. See the [CRD removal section][crd-removal-section] below for more details.  

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
  - operatorframework.io/cleanup-apis
```

In the event of operand cleanup being stalled due to some operator error, OLM will not clear the CSV finalizer so that the operator stays alive indefinitely. This way the operator can be observed to figure out the required action to unblock cleanup.

#### Operand cleanup API

The CSV spec can have an optional field `spec.cleanup` which lets a user configure the cleanup behavior before the CSV is deleted.

```YAML
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
...
spec:
  ...
  cleanup:
    enabled: false
```

With `spec.cleanup.enabled: true` OLM will first add the `operatorframework.io/cleanup-apis` finalizer on the CSV. Now when the CSV is deleted by the user, OLM will delete all CRs for the operator's owned CRDs, in the target namespaces of that operator. Once the CRs are successfully removed (after the operator clears its finalizers), OLM will clear the finalizer from the CSV to let the CSV and operator be uninstalled.

With `spec.cleanup.enabled: false` OLM will remove the `operatorframework.io/cleanup-apis` finalizer if it is already present on the CSV. This allows users to opt-out of cleanup and not have OLM delete any CRs.

This is also the default behavior if `spec.cleanup` is unspecified. This ensures operator's that haven't opted-in are not subjected to the finalizer and have their CSV lifecycle remain unchanged.

While toggling `spec.cleanup.enabled` does not result in a phase change, OLM could generate events to provide a history of when the cleanup behavior was toggled, and also events for when OLM begins and ends the cleanup process.

#### Cleanup off by default

OLM should not allow the cleanup behavior to be enabled by default when the operator is installed (and the CSV is first created). This is to prevent the case where the CSV author has enabled cleanup and the operator user is unaware of this feature (and expects OLM's current behavior) and suffers unexpected data loss on uninstalling the operator. The operator user will have to explicitly opt-in to cleanup by updating the CSV after operator installation.

For this OLM should always overwrite and disable the CSV cleanup option to `spec.cleanup.enabled: false` regardless of what was configured in CSV initially packaged in the operator bundle.
This can be done by overwriting the CSV during its initial `None` phase when it's first created, or alternatively when the CSV is first written out to the `InstallPlan` by the catalog operator.

#### Opt-in only if cleanup is supported

Since operator users can switch on the cleanup behavior for operators that don't actually implement Finalizers, OLM can end up deleting CRs for which there remain leftover secondary resources (off-cluster and cross-namespaced) that are now potentially untraceable because the root CR is deleted.

To prevent this there should be a second field `spec.cleanup.supported` (default `false`).

This field should be set to `true` by the operator/CSV author to convey if the operator has the capability to clean up all of its created resources via Finalizers.
This field can also be `true` even if the operator has no finalizers but it is sufficient to rely on the garbage collector to cleanup on-cluster secondary resources on CR deletion.

```YAML
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
...
spec:
  ...
  cleanup:
    supported: false
    enabled: true
```

If cleanup is not supported by the operator `spec.cleanup.supported: false`, OLM will ignore the field `spec.cleanup.enabled` and default to no cleanup and not delete CRs.
Only if `spec.cleanup.supported: true` will OLM act on `spec.cleanup.enabled` to enable/disable cleanup.

This allows the operator author to signal cleanup capabilities while still letting the operator user be the one to explicitly opt-in.

Ideally `spec.cleanup.supported` should be set by the operator author but if the user is aware that an operator requires no Finalizers and wants to leverage automatic cleanup then they can set both `spec.cleanup.supported: true` and `spec.cleanup.enabled: true` to force cleanup on uninstall.

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

**NOTE:** The number of CRs that OLM has to list in the status is unbounded and could potentially cause the CSV size to exceed the [max request size][max-request-size] configured in etcd. For this reason there would have to be an upper limit on the number of CRs displayed in `status.cleanup.pendingDeletion`.

The CSV can also be updated with a status condition to indicate that cleanup is in progress when the CSV is blocked on deletion.

```YAML
status:
  conditions:
  - lastTransitionTime: "..."
    lastUpdateTime: "..."
    message: |
      waiting for operator to finish cleanup for 4 CRs
    phase: Cleanup
    reason: CleanupWaiting
```

This CSV will transition to a new phase `Cleanup` with the reason `CleanupWaiting` when it is pending deletion with the finalizer `operatorframework.io/cleanup-apis` present.

#### Handling CSV deletion due to replacement

A CSV can also be [deleted by OLM][csv-deletion-on-upgrade] during it's lifecycle when OLM detects a newer CSV that replaces it as part of an upgrade. For an operator that has already opted-in to operand cleanup, this would result in OLM cleaning up all the CRs before removing the old CSV.

To prevent the cleanup of CRs on an upgrade, OLM should ensure that a CSV which has `spec.cleanup.enabled: true` and is marked for replacement (`phase: replacing`), should first be cleared of the `operatorframework.io/cleanup-apis` finalizer and opted-out of cleanup before it is marked for deletion or deleted.

#### CRD removal

While not strictly necessary for the goal of cleaning up operator-created resources, the cleanup process can also include removing the CRDs after the CRs and operator have been removed. The CRD can't always be removed however.

If there are two non-global operators installed of the same type (providing the same APIs), then removing the CRD during the cleanup of one operator would result in the deletion of CRs for the other operator as well.

To avoid this case OLM would have to ensure that it only removes the CRD during cleanup if:

- The operator installed is global
- The non-global operator is the only installation for that operator type in the cluster, i.e no other CSVs provide the same owned APIs.

## Design Details

### Test Plan

OLM's e2e testing suite should be upgraded to handle the following use cases:

- An operator that is uninstalled with `spec.cleanup` unspecified or `spec.cleanup.enabled: false` preserves the CRs
- An operator that is uninstalled with `spec.cleanup.enabled: true` removes all CRs
- An operator that is uninstalled with `spec.cleanup.enabled: true` should display all CRs that are pending deletion in the CSV status.
- An CSV with `spec.cleanup.enabled: true` when replaced/upgraded by a new CSV should trigger cleanup when the old CSV is deleted.

## Drawbacks

Goes against OLM's conservative approach to not remove APIs in order to prevent data loss. However this feature is opt-in so it shouldn't be a risk in the default setting.

[olm]: https://github.com/operator-framework/operator-lifecycle-manager
[owner-reference]: https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/#owners-and-dependents
[csv]: https://olm.operatorframework.io/docs/concepts/crds/clusterserviceversion/
[subscription]: https://olm.operatorframework.io/docs/concepts/crds/subscription/
[finalizers]: https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#finalizers
[operator-conditions]: https://github.com/operator-framework/enhancements/blob/master/enhancements/operator-conditions.md
[subscription-config]: https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/subscription-config.md
[target-namespaces]: https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/operatorgroups.md#target-namespace-selection
[owned-apiservices]: https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/building-your-csv.md#owned-apiservices
[finalizers-on-csvs]: https://github.com/operator-framework/enhancements/blob/master/enhancements/olm-deletes-operands.md#finalizers-on-csvs
[max-request-size]: https://github.com/etcd-io/etcd/blob/master/Documentation/dev-guide/limit.md#request-size-limit
[csv-deletion-on-upgrade]: https://github.com/operator-framework/operator-lifecycle-manager/blob/721fd858637579b894cfbe259a9e92f25b2e3f25/pkg/controller/operators/olm/operator.go#L1696-L1717
[kb-operator-crd-delete]: https://github.com/operator-framework/kubectl-operator/blob/ff147ab7b4560c32e9881bd0676dc728f54ea2fa/internal/pkg/action/operator_uninstall.go#L91-L97
[controller-guidelines]: https://github.com/kubernetes/community/blob/master/contributors/devel/sig-api-machinery/controllers.md#guidelines
[owned-crds]: https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/building-your-csv.md#owned-crds
[required-crds]: https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/building-your-csv.md#required-crds
[crd-removal-section]: https://github.com/operator-framework/enhancements/blob/master/enhancements/olm-deletes-operands.md#crd-removal
