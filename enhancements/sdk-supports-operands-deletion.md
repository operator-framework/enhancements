---
title: sdk-supports-operands-deletion
authors:
 - "@rashmigottipati"
reviewers:
  - "@ecordell"
  - "@estroz"
  - "@jmrordi"
  - "@joelanford"
approvers:
  - "@ecordell"
  - "@estroz"
  - "@jmrordi"
  - "@joelanford"
creation-date: 2021-01-29
last-updated: 2021-01-29
status: provisional
---

# sdk-supports-operands-deletion

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

The purpose of this enhancement is to outline how the [Operator SDK (OSDK)][sdk] can support operator developers targeting [Operator Lifecycle Manager (OLM)][olm] and need to clean up operands prior to an uninstall via custom resource (CR) deletion. OSDK will provide mechanisms to enable and facilitate CR deletion for operators that are installed using OLM and operator developers can explicitly opt-in for the cleanup of their operator workloads.

## Motivation

An operand is the program underlying a CR that an operator controls.
From this proposal’s perspective a CR controlled by the operator is the only resource object _directly_ cleaned up by OLM. Other resources such as pods, volumes, and off-cluster resources created by the operator need to be cleaned up by the operator.

In Kubernetes garbage collection model, an object can be an owner of another object within the same namespace. These "dependent" objects have an [OwnerReference][owner-reference] field that references the "parent" object. This relationship can be specified manually by the user, or automatically for certain resources on creation. If user deletes the parent object, Kubernetes garbage collector will delete all of its dependent objects (sometimes automatically if it is specified).

Cross-namespace owner references are not allowed by design; instead, a [finalizer][finalizers] should be used to perform cleanup. Finalizers are more powerful than owner references because they permit a controller to define how an object is cleaned up.

In OLM's [operand cleanup proposal][operand-cleanup-proposal] to handle operand deletion, OLM would configure the cleanup API in the [ClusterServiceVersion (CSV)][csv] and deleting the CSV will be the trigger to initiate operand cleanup. The CSV spec would have an optional field `spec.cleanup` which lets the user configure the intent to cleanup operands before the CSV is deleted.

```YAML
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
...
spec:
  ...
  cleanup:
    enabled: false
```

Upon setting `spec.cleanup.enabled: true`, OLM will add `operatorframework.io/cleanup-apis` finalizer on the CSV to prevent operator removal before cleanup. See [operand cleanup proposal][operand-cleanup-proposal] for more details.

## Goals

- Provide default scaffolding for an operand cleanup
- Cleanup is not opted in by default to preserve current behavior
- `operator-sdk cleanup` should optionally delete CRDs, and use OLM's operand cleanup feature

## Possible Future Goals (Not in scope)

- SDK should have a mechanism to handle events/alerts received by OLM in case cleanup is blocked
- The timeout for operand deletion should be user configurable by the SDK
- Currently, the cleanup command in OSDK is only used for testing purposes and not intended for production environments. In the future, when OLM supports cleanup for global operators, SDK cleanup mechanism would have the desired behavior with changes described in [existing cleanup command in SDK][existing-cleanup-command-in-sdk] section.

## Proposal

### User Stories

#### Story 1

As an operator author, I want an upstream finalizer handler scaffolded by default that lets me easily fill in operand-specific cleanup code.

#### Story 2

As an operator author, I want `spec.cleanup: false` added to my CSV by default when I call `operator-sdk generate kustomize manifests` so I can easily turn on the cleanup feature if I want to.

#### Story 3

As an operator author, I would like `operator-sdk cleanup` to handle deletion of CRDs appropriately based on the flags provided.

### Implementation Details/Notes/Constraints

#### Upstream finalizer no-op handler scaffolded by default and/or library to help write Finalizers

Any operator that deletes or expects delete events for CRs that create resources can use finalizers or owner references, although the latter can be used when those CRs only create resources in the same namespace. Additionally finalizers permit customizable deletion steps, ex. update a CR's status during deletion with a particular message. Because finalizers are broadly applicable and give operator authors control over how CR deletion proceeds, operator projects would benefit from some initial finalizer code scaffolded by default.

To enable this, it would be ideal to have a finalizer "no-op" handler, i.e. some code that does not clean up a CR but has a `TODO(user)` comment, scaffolded by default in the kubebuilder project. The idea is to always scaffold out a finalizer, a delete event handler and a Reconciler that checks for Finalizers and has a no-op in cases where it's not required to use a Finalizer.

The Reconciler checks for presence of a Finalizer in a CR object. If the CR object is marked for deletion, indicated by `metadata.deletionTimestamp` being set, the Reconciler runs the finalizer logic and removes the finalizer. If the finalization logic fails for some reason, then the Reconciler keeps the finalizer to retry during the next reconcilation. Typically, the finalizer logic would entail necessary cleanup steps that the operator needs to perform before the CR can be deleted. See [finalizer example][finalizer-example] for more details.

#### Configuration of the cleanup API in SDK

A [ClusterServiceVersion (CSV)][csv] represents a particular version of a running operator on a cluster and is an actual installation of the operator. The CSV will support the `spec.cleanup` field once the [operand cleanup proposal][operand-cleanup-proposal] is implemented

To enable configuration of operand deletion on the CSV, the `operator-sdk generate kustomize manifests` command will generate CSVs with the `spec.cleanup` field set to `false` by default. The user can then explicitly set this field to true to opt-in to automatic deletion of CRs using OLM.

Alternatively, a use-case where the user can automatically opt-in from the beginning will be supported in the future. The command should have an optional cleanup flag.  
* `operator-sdk generate kustomize manifests [--cleanup]`
  * If cleanup flag is not set by the user, SDK will apply the default and set `spec.cleanup: false` and leave it to the user to manually patch the CSV if the user intends to opt-in later.
  * If cleanup flag is set to true, `spec.cleanup: true` will be scaffolded in the CSV.


#### Existing cleanup command in SDK

The existing `operator-sdk cleanup` command currently deletes the CRDs and all other resources owned by the operator, without an option to disable this behavior. Ideally, the cleanup command should have certain flags that prevent deletion of CRDs, for feature parity with the [kubectl operator plugin](https://github.com/operator-framework/kubectl-operator/).

Existing cleanup command in OSDK:

```
operator-sdk cleanup <operatorPackageName>
```

Proposed cleanup command with flags:

```
operator-sdk cleanup <operatorPackageName> [--delete-crds]
```

* `--delete-crds` boolean flag should be added to the `cleanup` command, and defaulted to `true` to preserve current behavior.
  * When `--delete-crds=true`, the command will clean up all CRDs.
  * When `--delete-crds=false`, the command will not cleanup any CRDs.

This flag can be set accordingly to prevent deletion of CRDs unless explicitly asked for by the user.

### Risks and Mitigations

* What if Finalizers cause to hang? How do we handle this before implementing timeouts, and how do we mitigate this risk.
  * There are ways to remove the finalizer to unstick a namespace deletion.
* Implementing timeouts would be one of the future goals. When we implement timeouts, each operator could have its own custom cleanup of resources which would mean one operand cleanup could take longer than another.
  * If it is user configurable, the operator author needs to be aware of how long the cleanup process would take.
  * If SDK enforces a timeout that's not user configurable, it's important to come up with a default timeout for all operators or make timeout customizable per operator. For now, we don't have timeout implementation planned for the proposal and the risk is operator removal will be indefinitely stuck if a cleanup failure occurs.

[sdk]: https://github.com/operator-framework/operator-sdk
[olm]: https://github.com/operator-framework/operator-lifecycle-manager
[owner-reference]: https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/#owners-and-dependents
[finalizers]: https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#finalizers
[csv]: https://olm.operatorframework.io/docs/concepts/crds/clusterserviceversion/
[subscription]: https://olm.operatorframework.io/docs/concepts/crds/subscription/
[existing-cleanup-command-in-sdk]: https://github.com/operator-framework/enhancements/blob/master/enhancements/sdk-supports-operands-deletion.md#existing-cleanup-command-in-sdk
[operand-cleanup-proposal]: https://github.com/operator-framework/enhancements/blob/master/enhancements/olm-deletes-operands.md
[finalizer-example]: https://v1-3-x.sdk.operatorframework.io/docs/building-operators/golang/advanced-topics/#handle-cleanup-on-deletion