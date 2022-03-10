---
title: Failed-Operator-Upgrades
authors:
  - "@ankithom"
  - "@agreene"
reviewers:
  - TBD
  - "@dmesser"
  - "@bparees"
  - "@jlanford"
  - "@njhale"
approvers:
  - TBD
creation-date: 2022-04-04
last-updated: 2022-04-04
status: provisional
tags: enhancement proposal
---

# failed-operator-upgrades

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA

## Open Questions

## Summary

## Motivation

Cluster administrators rely on the [Operator Lifecycle Manager (OLM)](https://github.com/operator-framework/operator-lifecycle-manager) to install and manage the lifecycle of operators on a cluster.

An operator installation/upgrade will fail when:
- The InstallPlan enters the FAILED phase.
- The ClusterServiceVersion (CSV) enters the FAILED phase.

Today, cluster administrators can recover from the failed installation/upgrade by:
1. Updating the catalog to include a version of the operator that addresses the failure and includes a skip-range.
2. Manually deleting all traces of the failed installation and resubscribing to the operator.

The manual recovery process describe in step 2 becomes increasingly untenable as the number of clusters being administrated reaches into hundreds or thousands of instances. The goal of this enhancement is to propose a means to automatically recovery from failed installations without the need for manual intervention on the cluster.

### Goals

- Allow cluster admins to opt into "fail forward" upgrades.
- Allow OLM to automatically recover from CSVs in the FAILED phase when "fail forward" upgrades are enabled.
- Allow OLM to automatically recover from InstallPlans in the FAILED phase when "fail forward" upgrades are enabled.

### Non-Goals

- Support operator rollbacks.
- Allow OLM to automatically recover from CSVs in the PENDING phase.
- Reattempt to install a failed installation.

## Proposal

OLM should be updated to allow cluster admins to recover from failed installations and upgrades. A failed installation/upgrade typically caused by one of two scenarios:
1. **A Failed CSV:** The CSV is in the FAILED phase.
2. **A Failed InstallPlan:** Usually occurs because a resource listed in the InstallPlan fails to be created or updated. An InstallPlan may fail independently of its CSV and may fail to create the CSV.

This enhancement proposes that OLM be updated support "fail forward" upgrades by:
- Providing cluster admins a means to opting into "fail forward" upgrades.
- Allowing CSVs to move from the FAILED phase to the REPLACING phase when "fail forward" upgrades are enabled.
- Allowing OLM to calculate new installPlans for a set of installables if:
-- The InstallPlan referenced by a subscription is in the FAILED phase.
-- The CatalogSource has been updated to include a new upgrade for one or more CSVs in the namespace.

### User Stories

- As a cluster admin, I want to allow operators in a specific namespace to "fail forward".
- As a cluster admin, if an InstallPlan is in the FAILED state, I want OLM to create the next InstallPlan if the catalog has been updated and includes a new set of installables.
- As a cluster admin, if a CSV is stuck in the FAILED state and I have enabled "fail forward" upgrades, I want to OLM allow the operator to upgrade to the next version.

### Implementation Details/Notes/Constraints [optional]

#### Opting into Fail Forward Upgrades

Since resolution is namespaced, the toggle for allowing "fail forward" upgrades should be namespaced as well to avoid having to add the extra burden on the resolver surrounding understanding which dependent operators can or cannot use the forced upgrade toggle.

With this in mind, this enhancement proposes adding the `upgradeStrategy` field to an already watched namespaced resource, the OperatorGroup.

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: foo
  namespace: bar
spec:
  upgradeStrategy:
     # Possible values include "Default" or "UnsafeFailForward".
    name: UnsafeFailForward
```

With the the upgradeStrategy type is set to `UnsafeFailForward`, OLM will allow operators to "fail forward" in adherence with the principles discussed below. If the upgradeStrategy is unset or set to `Default` OLM will exhibit existing behavior. It is critical that the documentation surrounding `UnsafeFailForward` makes it clear to users that critical versions of operators may not be installed which can lead to unknown behavior and potentially data loss.

#### Supporting "Fail Forward" Upgrades

Before diving into the specific behavior OLM will need to support when "failing forward" from failed CSVs or InstallPlans, it is important to understand how OLM and the Resolver calculate what needs to be installed. When determining what to install, OLM provides the resolver with the set of CSVs in a namespace. The resolver treats these CSVs differently depending on whether or not they are claimed by a Subscription, where the Subscription lists the CSV's name in its `.status.currentCSV` field.
- If a CSV is claimed by a Subscription, the resolver will allow it to be upgraded by a bundle that replaces it.
- If a CSV is not claimed by a Subscription, it is assumed that the user installed the CSV themselves, that OLM should not upgrade the CSV, and that it must appear in the next set of installables.

OLM's Resolver is deterministic, meaning that the set of installables will not change for a given set of arguments if no new upgrades have been declared in the existing CatalogSources. When an upgrade fails due to a CSV entering the FAILED phase, the CSV being replaced still exists and is not referenced by a subscription. Today, OLM would send both `Operator v1` and `Operator v2` to the resolver which will normally be unsatisfiable because `Operator v1` is marked as required and `Operator v2` cannot be upgraded further since it provides the same APIs as `Operator v1`. In order to support "failing forward", OLM will need to omit CSVs in the REPLACING phase from the set of arguments sent to the resolver when "fail forward" upgrades are enabled.

##### Supporting "Fail Forward" Upgrades when an InstallPlan failed

Let's review what steps must be taken to recover from a failed InstallPlan today:
1. `Operator v1` is being upgraded to `Operator v2`.
2. The InstallPlan for `Operator v2` fails.
3. `Operator v3` is added to the catalog and is defined as the upgrade for `Operator v1`.
4. The user deletes the InstallPlan created for `Operator v2`.
5. A new InstallPlan is generated for `Operator v3` and the upgrade succeeds.

The enhancement proposes that when the user opts into "fail forward" upgrades that OLM allows new InstallPlans to be generated if:
- The failed installPlan is referenced by a subscription in the namespace.
- The catalogSource has been updated to include a new upgrade for the set of arguments.

In practice, the fourth step from the previous workflow would be removed, meaning that the cluster admins would simply need to update the CatalogSource and wait for the cluster to move past the failed install. We do not need to code special handling for partially applied InstallPlans as OLM will include any applied CSVs in the set of arguments sent to the resolver.

>BEST PRACTICE NOTE: If a bundle is known to fail, it should be skipped in the upgrade graph using the [skips](https://olm.operatorframework.io/docs/concepts/olm-architecture/operator-catalog/creating-an-update-graph/#skips) or [skipRange](https://olm.operatorframework.io/docs/concepts/olm-architecture/operator-catalog/creating-an-update-graph/#skiprange) feature.

##### Supporting "Fail Forward" Upgrades when one or more CSVs have failed

Let's review what steps must be taken to recover from a failed CSV today:
1. `Operator v1` is being upgraded to `Operator v2`.
2. The CSV for `Operator v2` enters the FAILED phase.
3. `Operator v3` is added to the catalog which replaces or skips `Operator v2`.
3. The resolver cannot upgrade `Operator v2` while including `Operator v1` in the solution set, upgrade is blocked.
4. User manually deletes the existing CSVs, a new InstallPlan is generated and approved.
5. `Operator v3` is installed and the upgrade succeeds.

The enhancement proposes that when the user opts into "fail forward" upgrades that OLM:
- Does not include CSVs in the REPLACING phase in the set of arguments sent to the Resolver.
- Allows CSVs to move from the FAILED phase to the REPLACING phase.

In practice, these changes would allow OLM to recover from any number of failed CSVs in an upgrade path. Cluster admins would only need to update the CatalogSource with new operators that replace the FAILED CSVs.

### Prototype Demo

[![asciicast](https://asciinema.org/a/RErOz2BGrpPQcFwPqpMieVn8T.svg)](https://asciinema.org/a/RErOz2BGrpPQcFwPqpMieVn8T)

[OLM Prototype](https://github.com/awgreene/operator-lifecycle-manager/tree/ff-hack-2)
[Demo Catalog](https://github.com/awgreene/fail-forward-demo-catalog)

### Risks and Mitigations

- Critical operator versions may be missed if "failing forward" is enabled, potentially leading to any number of unknown consequences. The documentation surrounding this API will make it clear that enabling this feature may have severe consequences and that recovery from these consequences isn't guaranteed.

## Design Details

### Test Plan

The following tests are based on the high-level workflow described below in namespaces where the OperatorGroup.spec.failForward is set to `true`:
1. `Operator v1` is installed
2. `Operator v1` is upgraded to `Operator v2`, which fails to install.
3. OLM supports "failing forward" to `Operator v3`.

- OLM can successfully upgrade to `Operator v3` if the `Operator v2` CSV is stuck in a FAILED state.
- OLM can successfully upgrade to `Operator v3` if the InstallPlan is stuck in a FAILED state. 
- Existing behavior is observed when a user has not opted into the "fail forward" feature.

### Graduation Criteria

### Version Skew Strategy

The proposed behavior is enabled by a field present in the spec of the OperatorGroup resource. Cluster admins can choose to opt into this mode as needed. As long as the operator author provided upgrade graphs are sane, there should be no issues with turning the toggle off and on in a cluster regardless of its health.

## Alternatives

### Alternative 1: Providing the CSV that initiated the failed upgrade to the resolver

Consider the following high-level workflow:
1. `Operator v1` is installed
2. `Operator v1` is upgraded to `Operator v2`, which fails to install.
3. OLM supports "failing forward" to `Operator v3` by **providing the resolver with a set of operators installed in the namespace**. `Operator v3` is identified as a valid upgrade and is installed successfully.

The workflow for catalog curators/operator authors is determined by the version of the operator sent to the resolver in Step 3. In the scenario above, the operator version could be:
- The operator that initiated the upgrade, `Operator v1`.
- The operator that failed to install, `Operator v2`.

Instead of providing `Operator v2` to the Resolver as described in this enhancement, OLM could instead provide the resolver with `Operator v1`, the operator version that initiated the upgrade. By providing the resolver with the `Operator v1`, OLM would recalculate the list of installables using the set of operators that initiated the failed upgrade. OLM would only create a new InstallPlan once the catalog was updated to include a new upgrade path for the `Operator v1`.

##### Pros

- The catalog curator/operator author must be aware of the failed installation, allowing them to address the issue directly by providing a new upgrade path.
- Involving catalog curators/operator authors helps prevent unsafe upgrades.
- Upgrade graph maintenance relies tools/concepts embraced today.

#### Cons

- The catalog curator/operator author must manually address each failed installation.
- Cluster admins cannot role forward until a new catalog is published.
- Deviates significantly from OLM's upgrade behavior today. By the time `Operator v2` has failed, most resources have been updated to include `Operator v2` as an owner. This impacts OLMs upgrade logic in multiple places and introduces additional complexity to the solution.

### Alternative 2: Address with V2 APIs

We had considered introducing the features proposed in this enhancement by way of the V2 APIs that are currently under development. Ultimately, it was decided that introducing theses features in the next generation of APIs may prohibit them from landing in a timely manner and would impose a non-trivial amount of work on users to update their existing workflows to the new APIs.

### Alternative 3: Allowing Operator Authors to opt out of fail forward upgrades

The OLM team considered introducing the "fail forward" [Bundle Property](https://olm.operatorframework.io/docs/reference/file-based-catalogs/#properties) to enable operator authors to ensure that critical versions of an operator are not skipped by the "fail forward" feature.

To prevent an operator from "failing forward" to the next version, a catalog curator/operator author would have included the following annotation on the bundle like so:

```yaml=
image: quay.io/foo/bar:baz
name: foo-operator.v1.0.1
package: bar
properties:
- type: olm.gvk
  value:
    group: foo.io
    kind: bar
    version: v1alpha1
- type: olm.package
  value:
    packageName: stable
    version: 1.0.1
- type: olm.failForward
  value:
    supported: false # This bundle does not support "failing forward" to the next version.
schema: olm.bundle
```

With this property present, OLM would not allow the CSV to transition from the FAILED phase to the REPLACING phase, effectively blocking "fail forward" upgrades in the namespace.

#### Pros

- The solution provided catalog curators / operator authors the means to opt out of "fail forward" upgrades.

#### Cons

- Cluster admins that do not control an operator's bundle properties may be blocked from "failing forward".
- Behind the scenes, this property only influences which CSVs (arguments) OLM sends to the resolver. Once OLM has installed a bundle, the property won't appear on the CSV if the catalog is updated to include it, meaning that if you release a CSV and later deem that it should not support "failing forward", customers that have already installed the CSV can still fail forward. This lead to an unintuitive user experience and was ultimately discarded in hopes of better addressing user needs with the next generation of APIs.
