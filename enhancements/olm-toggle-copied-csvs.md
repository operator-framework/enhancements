---
title: toggle-copied-csvs
authors:
  - "@njhale"
reviewers:
  - "@TheRealJon"
  - "@spadgett"
  - "@bparees"
  - "@joelanford"
  - "@kevinrizza"
approvers:
  - "@bparees"
  - "@joelanford"
  - "@kevinrizza"
creation-date: 2021-09-27
last-updated: 2021-11-15
status: implementable
see-also:
  - "/enhancements/olm-toggle-copied-csvs.md"
---

# Toggle Copied CSVs 

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Operational readiness criteria is defined
- [x] Graduation criteria for dev preview, tech preview, GA

## Summary

This proposal introduces a novel config [CRD](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) along with a feature toggle that disables/enables the creation and management of [Copied CSVs](https://olm.operatorframework.io/docs/concepts/crds/operatorgroup/#copied-csvs), so that cluster admins can curb OLM's resource usage on constrained clusters.

## Motivation

When an Operator is installed by OLM, a stripped down copy -- e.g. a stub -- of its [CSV](https://olm.operatorframework.io/docs/concepts/crds/clusterserviceversion/) is created in every namespace the Operator is configured to watch (see [OperatorGroup](https://olm.operatorframework.io/docs/concepts/crds/operatorgroup/); collectively, these are known as _Copied CSVs_. This means that the number of Copied CSVs on a given cluster grows proportionally with both the number of namespaces and installed Operators. In the worst case, a cluster can have a Copied CSV for every Operator in every namespace, that is `|copied| = |operators|*|namespaces|`. For especially large clusters, with namespaces and installed operators tending in the hundreds or thousands, Copied CSVs consume an untenable amount of resources; e.g. OLM's memory usage, cluster Etcd limits, networking, etc. For these reasons, it's imperative that Copied CSVs be replaced with a more resource friendly successor. In the meantime, while no such successor exists, OLM should have a knob that helps curtail its high resource utilization.

### Goals

- Alleviate the resource utilization incurred by Copied CSVs
- Don't block upgrades for customers of OpenShift

### Non-Goals

- Provide a replacement for Copied CSVs

## Proposal

Provide a CRD for OLM that allows users to enable and disable Copied CSVs for CSVs installed in `AllNamespace` mode, then for each non-Copied CSV, a single [Operator](https://github.com/operator-framework/enhancements/blob/master/enhancements/simplify-olm-apis.md#proposal) resource will be generated. This cluster-scoped resource will take the place of any related Copied CSVs.

### User Stories

#### Toggle Copied CSVs Off

As a Cluster Admin, I want a way to disable Copied CSVs so that I can reduce OLM's resource consumption on my cluster.

#### Toggle Copied CSVs On

As a Cluster Admin, I want a way to enable Copied CSVs if I've reconfigured the cluster in ways that mitigate the problem; e.g. uninstalled operators from `AllNamespace` `OperatorGroups`.

### Implementation Details/Notes/Constraints

#### Operator Cardinality

Today, Operator resources are generated on a per-Subscription basis. Since CSVs -- and thus Copied CSVs -- can exist without matching Subscriptions, OLM's controllers will be extended to generate additional Operator resources for these "free CSVs" such that:

- there exists at least one `Operator` resource for every CSV
- there exists at least one `Operator` resource for every `Subscription`

That is, all CSVs installed by a given `Subscription` share an `Operator` resource __and__ any CSVs that weren't installed by a `Subscription` are related to their own `Operator` resource.

#### Orphaned CSVs

Orphaned CSVs are CSVs that were installed via a `Subscription` that no longer exists. Since the lifetimes -- i.e. the period of existence on a cluster --  of `Operator` resources aren't directly tied to the lifetimes of the resources that triggered their generation, the `Operator` resource(s) associated with Orphaned CSVs will continue to exist _after_ a `Subscription` is deleted. If that `Operator` resource is deleted explicitly, OLM will generate another as per the rules [outlined above](#operator-cardinality).

#### Copied RBAC 

In addition to operator discoverability, Copied CSVs also serve as convienient vehicles for the garbage collection of Copied RBAC. These are resources that are copied along with the CSV to provide the operator access to a namespace. To support the loss of Copied CSVs, the RBAC copying logic in OLM will be extended to write additional owner references from copied resources to the related Operator resource. 

#### The Toggle

A feature toggle will be added to control OLM's use of Copied CSVs. When enabled (toggled on), OLM will delete all existing Copied CSVs and will not generate any more. When disabled (toggled off), OLM will generate Copied CSVs (much like it does today).

When toggled (on then off), OLM will recreate any missing Copied CSVs and reassociate the owner references of related copied resources with them (i.e. the inverse operation of the one described in the [Copied RBAC](#copied-rbac) section).

The toggle will be disabled by default.

The toggle can be set from a command-line option on the olm-operator binary.

#### Config CRD w/ Toggle

A novel cluster-scoped `OLMConfig` CRD will be added to OLM. The resource type it defines will allow users to apply a limited set of configurations to the OLM instance on their cluster by creating or modifying a singlton (of that type) on the cluster.

The name of that singleton will be `cluster` and OLM will ignore all other `OLMConfig` resources on a cluster.

Its spec will contain a Copied CSV feature toggle field.

e.g.

```yaml
apiVersion: operators.coreos.com/v1
kind: OLMConfig
metadata:
  name: cluster
spec:
  features:
    'disable-copied-csvs': true
```

When both the [command-line option](#the-toggle) _and_ `cluster` resource toggle field are present, the latter takes precedence.

When the `cluster` resource isn't present, OLM uses its default settings (in this case, Copied CSVs would be **on** by default).

### Risks and Mitigations

#### Loss of Operator Discovery

**Risks:** When Copied CSVs are disabled, users won't be able to discover what operators are configured to watch a particular namespace unless they have access to their install namespaces.

**Mitigations:** 

- Reduce number of impacted users by limiting toggle to operators installed in `AllNamespace` mode only; that is, `Single` and `MultiNamespace` mode Copied CSVs will not be affected by the toggle.
- OLM fires an alert in all affected namespaces on startup when Copied CSVs are disabled
- Docs suggest admins notify their cluster tenants of installed operators directly; i.e. email, slack, etc

#### OpenShift Console Integration

**Risk:** The OpenShift Console uses Copied CSVs in the rendering of certain pages, so disabling Copied CSVs will negatively impact UX.

For the **admin console**, Copied CSVs are used to "drill down" on the details of operators watching a particular namespace:

"Installed Operators" page (i.e. a list of Copied CSVs) 
└── "Operator Details" page (e.g. Copied CSV of Strimzi)
            └── "Operand List" page (i.e. instance list of provided APIs through Copied CSV)
                                    └── "Operand Details" page (e.g. kafka instance details through Copied CSV)

For the **dev console**, Copied CSVs drive the creation and tracking of CRs.

**Mitigations:**

- Impact to console UX when Copied CSVs are disabled is well documented.

**Potential Further Mitigations:**

> **Note:** The mitigations below are **suggestions** for mitigations that could be implemented in the Console if deemed valuable by its maintainers. They are not a prerequisite of any other solutions in this proposal. As such, they are **not** guaranteed implementation.

- Console detects when Copied CSVs are diabled and displays a warning to admins and affected users
- A toggle component is added to Console to make re-enabling Copied CSVs easier
- A link to revelant docs is displayed to admins and affected users


## Drawbacks

- [Loss of Operator Discovery](#loss-of-operator-discovery)

## Alternatives

### Command-line Option Only

Update the olm-operator `Deployment` to toggle the Copied CSV feature.

#### Cons

On an OpenShift cluster, [a CVO override](https://docs.openshift.com/container-platform/4.8/logging/config/cluster-logging-maintenance-support.html#unmanaged-operators_cluster-logging-unsupported) would be required to prevent the updated command-line option in the olm-operator `Deployment` from reverting, but this also puts clusters into an unsupported state and prevents cluster upgrades.

### OpenShift FeatureGates

Define a new OpenShift [FeatureGate](https://github.com/openshift/api/blob/master/config/v1/types_feature.go) feature set that, when disabled, disables Copied CSVs. This would be preferred over any command-line option and would require no CVO override.

#### Cons

- Requires OLM to be OpenShift-aware
  - Could be addressed with a downstream-only controller that deploys OLM with the correct options
- Requires changes OpenShift (a new feature set)

### Config ConfigMap

Instead of using a novel CRD to configure OLM, use a ConfigMap.

#### Cons

- ConfigMap fields are loosely typed; i.e. lacking validation at the API level

### New Field (or Annotation)

Add a new field or annotation to existing types such as `CSVs` or `OperatorGroups` to denote disabling/enabling Copied CSVs for related resources.

#### Cons

- CVO would revert changes to resources included with the payload; e.g. the `openshift-operators` `Namespace` or `OperatorGroup`
- Changing existing schemas is questionably backportable (for OpenShift releases)
- Giving the ability to toggle the feature for some portions of a cluster but not others opens the door to an inconsistent user experience; e.g. as a user, I must use two separate methods to be sure of operator availablity.

### Virtual "InstalledOperator" API

A "virtual resource" served by an aggregated API -- much like `PackageManifest` -- that provides a namespaced, CSV-like, view of the operators configured to interact with a given namespace. Such a resource would be a direct drop-in-place for a Copied CSV.

#### Cons

- Aggregated APIs generally lead to cluster-instability; e.g. breaking cluster API discovery
- Aggregated APIs require serving cert management and rotation; i.e. increased installation and lifecycle burden
- Virtual resources may impact garbage collection in unforeseen ways
