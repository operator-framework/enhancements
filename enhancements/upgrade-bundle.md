---
title: bundle-upgrade
authors:
  - "@rashmigottipati"
reviewers:
  - "@joelanford"
  - "@estroz"
  - "@shawn-hurley"
  - "@jmrodri"
  - "@njhale"
  - "@matskiv"
approvers:
  - "@shawn-hurley"
  - "@estroz"
  - "@jmrodri"
creation-date: 2020-06-11
last-updated: 2020-06-11

---

# Bundle Upgrade

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [X] Design details are appropriately documented from clear requirements
- [X] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions
 
1. How do we handle `--mode` flag on registry add?
   * Based on the discussions, making `--mode` a hidden optional flag and using the default `replaces`. 
2. Can we provide configurability to user to set the approval to `manual/automatic` based on their use case? In this case, we will have to patch the subscription. 
  * In this enhancement, use the existing subscription approval field that is already set.
3. If subscription name is on the CSV, can we infer it from the CSV and not make it a required flag? 
  * For now, keep it as required. And if `run bundle` has executed before, it should be printing the subscription name.
4. Is it possible to infer the index image reference from `subscription` --> `catalog source spec` and make index image optional? If optional, what would the default be?  

## Summary

Enhance the Operator SDK to add a new subcommand `run bundle-upgrade` that performs an operator upgrade using OLM.

## Motivation

Testing operator upgrades remains difficult today due to the number of steps to prepare for an upgrade and the lack of tooling that automates those steps. Having high quality Operators is crucial to the success of the Operator Framework and thereby operator developers should have a tool to use to perform an operator upgrade.

### Goals

Provide a command line that upgrades an operator version using OLM.

### Non-Goals

This enhancement doesn't include the capability for SDK to use labels to detect the DB path. For the sake of simplicity, we will assume that the existing registry DB is located at `/database/index.db`.

## Proposal

### Implementation Details/Notes/Constraints [optional]

The Operator SDK currently supports running an operator with OLM using the `run packagemanifests` subcommand. This enhancement will add a new subcommand, `bundle-upgrade`, to the `run` command. 
This new command, `run bundle-upgrade`, will perform the upgrade of an operator from a previous version to a new specified version.

Currently, the run command has different subcommands: 

 * `run local`
 * `run packagemanifests`
 * `run bundle` - currently being implemented, design document is covered by [run bundle enhancement](https://github.com/operator-framework/enhancements/blob/master/enhancements/run-bundle.md)

For doing an `bundle-upgrade`, there could be two scenarios: 

* Case 1 - A previous version of the operator is already running on the cluster, and the index image contains this operator version. Now, we want to upgrade to a development version that the user is currently working on. 

* Case 2 - No previous version of an operator is running on the cluster, so we provide the new bundle image which is accessible and pullable within the cluster, which contains the version that the user is working on. 

In this proposal, we will design bundle upgrade for case 1. One of the crucial assumptions we are making in this proposal is that an operator is already running in the cluster which leads to the assumption that an index image already exists that has the from version of the operator bundle. Therefore, we don't create a new index image but use the existing index image (which is a required flag) for the operator and provide the new bundle image for the upgrade. So the previously installed operator will have a subscription (which should be provided by the user) using which we will implement the logic to get the catalog source the existing operator was installed in and dynamically inject the new bundle image into the index image for OLM to detect this change and upgrade to the new version we provide. 

Let's consider a few scenarios: 

  * Verifying operator upgrades
    * To verify whether an operator upgrade to a desired version has been successful, the ClusterServiceVersion (CSV) CustomResource is a useful resource for providing such context to the end user. A given subscription has its status field and corresponding InstallPlan. If a ClusterServiceVersion (CSV) is created for that InstallPlan, then the CSV Status indicates that the operator has been successfully upgraded. We will ensure that the CSV status is `Successful` and that the previous CSV has been deleted.

  * Do we cleanup resources after failures? 
    * We keep the registry pod and not clean up any resources as it could be useful for further debugging. Alternative, we could offer flexibility to the operator developers to use a flag --skip-cleanup to indicate whether or not cleanup of resources is desired. This could enable the operator developers to capture all the logs and inspect the pod failures.

  * Behavior of `run bundle-upgrade` in the presence of an OperatorGroup
    * If an operator is installed, it should have a Subscription and an OperatorGroup already created. This OperatorGroup exists with the correct namespacing and it satifies the subscription. So `run bundle-upgrade` will use the same OperatorGroup that was created during install. However, if the upgrade of the operator doesn't work with the existing OperatorGroup, then error out. 
    In `run bundle`, there is support for 3 modes: 
        * AllNamespaces
        * OwnNamespace
        * SingleNamespace=ns1

Example UX Scenarios are discussed below. 

#### Command Line Interface
The new command line will look as follows:

```
operator-sdk run bundle-upgrade (alias: bu) <bundle-image> --index-image=<> --subscription-name=<> --namespace=<> [--mode=<>]
```

* Positional arguments:
    * `bundle-image`
* Required flags:
    * `--index-image` 
    * `--subscription-name`
* Optional flags:
    * `--namespace`
    * `--mode` 

* `<bundle-image>` is a positional argument that specifies the bundle image that we want to upgrade to.
* `--index-image=<>` is a required flag, which contains the operator we’re upgrading from.
* `--subscription-name=<>` is a required flag that specifies the name of a subscription. 
* `[--namespace=]` is an optional flag that specifies the namespace of a subscription.
* `[--mode=]` is a hidden optional flag that specifies the graph update mode that defines how channel graphs are updated. Mode can be one of: [replaces, semver, semver-skippatch] (default "replaces"). It will default to the `replaces`.


#### Assumptions
* OLM is already installed
* A previous version of the operator is already running in the cluster
* An index image exists that has the from version of the operator bundle
* The provided images (i.e. bundle and index) are accessible and pullable by kubelet 

#### Example UX Scenarios

Here are some example scenarios of using `run bundle-upgrade`:

* `operator-sdk run bundle-upgrade <bundle-image> --index-image quay.io/operator-framework/upstream-opm-builder --subscription-name sub`

* `operator-sdk run bundle-upgrade <bundle-image> --index-image quay.io/operator-framework/upstream-opm-builder --subscription-name --namespace memcached-system --mode replaces` 

#### Simple Example

Here's a simple example with all the required flags.

```
operator-sdk run bundle-upgrade <bundle-image> --index-image quay.io/operator-framework/upstream-opm-builder --subscription-name sub
```

Below is the workflow when the above command is invoked:

 1. Create pod using `--index-image`, set container command (entrypoint) to:
    ```
    /bin/bash -c ‘ \
      /bin/mkdir -p /database && \
      /bin/opm registry add   -d /database/index.db -b {.BundleImage} && \
      /bin/opm registry serve -d /database/index.db’
    ```
 1. If pod fails, error out with message of unsupported index image. We don't clean up the pod but we capture the pod's logs for debugging.
 1. From the Subscription, find the existing GRPC CatalogSource pointing to `<pod-name>.<namespace>:50051`.
 1. Update the image reference in the existing CatalogSource, pointing it to the new bundle image provided.
 1. Based on the approval set in the existing Subscription, OLM will perform the upgrade (manual/automatic).
 1. As `--namespace` i.e. subscription namespace was not provided, look for operator being upgraded in the namespace defined by KUBECONFIG.
 1. Verify operator is upgraded (by looking at the CSV version, status, etc.).
 1. Log what happened and final state (e.g. operator is running in namespace X, watching namespace(s) Y).
 1. Log command to execute to cleanup operator.
Exit with success (leaving operator running)

#### Complex Example

Let's take an example providing all the required and optional flags in the `bundle-upgrade` command.

```
operator-sdk run bundle-upgrade <bundle-image> --index-image quay.io/operator-framework/upstream-opm-builder --subscription-name sub --mode replaces --namespace memcached-system 
```

1. Create pod using `--index-image`, set container command (entrypoint) to:
    ```
    /bin/bash -c ‘ \
      /bin/mkdir -p /database && \
      /bin/opm registry add -d /database/index.db -b {.BundleImage} mode={.Mode} && \
      /bin/opm registry serve -d /database/index.db’
    ```
 1. If pod fails, error out with message of unsupported index image. We don't clean up the pod but we capture the pod's logs for debugging.
 1. From the Subscription, find the existing GRPC CatalogSource pointing to `<pod-name>.<namespace>:50051`.
 1. Update the image reference in the existing CatalogSource, pointing it to the new bundle image provided.
 1. Based on the approval set in the existing Subscription, OLM will perform the upgrade (manual/automatic).
 1. As `--namespace` i.e. subscription namespace was provided, look for the operator being upgraded in the namespace `memcached-system`
 1. Verify operator is upgraded (by looking at the CSV version, status, etc.)
 1. Log what happened and final state (e.g. operator is running in namespace
    `memcached-system`.
 1. Log command to execute to clean up operator.
Exit with success (leaving operator running)

#### Prototype Links

 * Prototype for run bundle may be able to be adapted to prototype run bundle-upgrade:    https://github.com/joelanford/bundle-runner
 * Bundle format:
   https://github.com/operator-framework/operator-registry/blob/master/docs/design/operator-bundle.md

### Test Plan

  * manual testing with different scenarios 
  * add unit tests for `internal/olm/operator/operator.go` (with coverage requirements)
  * add e2e tests [e2e tests](https://github.com/operator-framework/operator-sdk/blob/master/test/integration/operator_olm_test.go)

### Graduation Criteria

The `run bundle-upgrade` command will come in as non-alpha and be ready to use once merged.

#### Future Improvements

* Enable automated testing of operator upgrades using scorecard

### Alternatives

An alternative to handling failed pods is to have a flag, --skip-cleanup, that could be added to skip cleaning up failed pods. This will enable the developers to inspect the failed pods by capturing all the logs and debug the issues. Irrespective of this flag, the current implementation will support capturing pod logs. 

## Infrastructure Needed [optional]

N/A
