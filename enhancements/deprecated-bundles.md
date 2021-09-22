---
title: Deprecating-Bundles
authors:
  - "@gallettilance"
reviewers:
  - "@ecordell"
  - "@shawn-hurley"
  - "@kevinrizza"
approvers:
  - "@ecordell"
  - "@shawn-hurley"
  - "@kevinrizza"
creation-date: 2020-07-15
last-updated: 2020-07-15
status: implementable
---

# Deprecating Bundles

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined

## Summary

OPM will support marking versions as deprecated and truncating the update graph of a given package. Deprecated versions will not be installable. If a deprecated version is installed prior to upgrade appropriate alerts will be fired on cluster. 

## Motivation

The default set of Operators provided on OCP clusters will soon (4.6+) be delivered through indexes and each OCP minor version will have a unique set of index images. Today, packages in one index will be duplicated to the next minor version index when it is available. This means operator owners releasing through the redhat, certified, or community pipelines will now need a way to specify which versions of their operator are incompatible or unsupported in the next minor version index.

### Terminology

#### Deprecated bundle

A deprecated bundle is an upgradable but not installable bundle in the update graph of an operator.

#### Terminated bundle

A terminated bundle is neither upgradable nor installable.

### Goals

- A deprecated bundle in the update graph and any of its replacements (terminated bundles) will not be installed by OLM
- Operator Owners don’t need to release a new version in order to specify which bundle is deprecated
- Index maintainers have a way to un-do deprecation if needed
- The deprecation is specified after promotion (not during) and before publishing of the index image
- Alerts are fired on cluster when a deprecated version is installed

### Non-Goals

- OLM supports upgrading from a terminated bundle
- Alerts are fired if terminated bundles exist on cluster
- OLM prevents cluster upgrades based on whether versions are terminated in the upgraded cluster’s index

## Proposal

We propose to add a command to `opm index` called `deprecate` which takes in a set of bundle images, an index, and all the same arguments as `opm index add`. The command modifies the index by removing all versions that come before the specified bundle image in the update graph (that is, until the tail of each channel). If this operation removes the head of a channel, then the channel is removed. The removal of bundles from the update graph will not modify the update graph after the deprecated version (from the deprecated channel to the head of the channel).

Versions that were removed from the index will not receive updates if subscriptions still exist for them on cluster. Ideally, appropriate communication from the operator maintainer will mean that cluster admins will upgrade to the latest supported version prior to cluster upgrade. In the event that the cluster is upgraded and `Subscriptions` to terminated bundles exist on cluster, the cluster admin can manually trigger an upgrade by either:

- editing the `Subscription`'s `channel` field to start receiving updates from a different channel
- creating a custom `CatalogSource` pointing to the previous OCP index, editing the `Subscription`'s `source` field to point to the new `CatalogSource` in order to receive updates.

With the new work on skip range being steppable, the above scenario may be less likely to happen.

A new field will be added to the operator-registry database table `operatorbundle` called `deprecated` and the bundle images specified in the command will be marked as such.

The OLM resolver will be modified with a new property type called `olm.deprecated` in order to filter out any options that have this property set and disallow installs of deprecated bundles. This also means appropriate status reporting in the subscription and installplan need to be added to OLM, and alerts will be fired if a deprecated version is installed.

### User Stories

#### Story 1

As an index maintainer I want to be able to truncate update graphs in order to remove or prevent installs of incompatible or unsupported versions of certain operators.

##### Implementation Details

A new field is added to the operator-registry database table `operatorbundle` called `deprecated` and the bundle images specified in the command will be marked as such. Versions from the deprecated version to the tail of each channel are removed from the database.

For example, suppose following channel:

```
    1.4.0 -- replaces -> 1.3.0 -- replaces -> 1.2.0 -- replaces -> 1.1.0
```

we then deprecate version 1.3.0 using the following command

```
    $ opm index deprecate --bundles "quay.io/my/bundle:1.3.0" --from-index "quay.io/my/index:v4.6" --tag "quay.io/my/index:v4.7"
```

Now, in `quay.io/my/index:v4.7` the above channel looks like:

```
    1.4.0 -- replaces -> 1.3.0 [deprecated]
```

Deprecated versions are upgradable but not installable. To achieve this, we will modify the operator-registry apis used by OLM to determine what can be installed.

#### Story 2

As an index maintainer I want to be able to undo the above operation.

##### Implementation Details

Index maintainers should revert to the index image used in the original command. If the tag is overwritten, they must keep track of the shas of the index images they modify in order to guarantee recoverability. If the image is, for some reason, not available, the index maintainer can remove the entire package using `opm index rm` and re-add all the bundles for that package.

Eventually, the sha of the index being modified could be stored in the database of the modified index for quicker recovery. Additionally this field would allow stepping backward with a step size equivalent to the operations / transactions provided by `opm` guaranteeing stepping back to a valid index image.

#### Story 3

As a cluster admin, I want to be alerted if subscriptions exist on cluster for deprecated versions of operators. The alert should be for:

- "Subscription is to a channel that no longer exists, or the head is deprecated"
- "A Manual subscription is on a deprecated version"

Eventually, we will want alerts for "Terminated version is installed" but this is outside the scope of this enhancemnent as it currently is not possible to distinguish between a terminated version and a version installed from a semver-skipatch index using skipRange.

## Open Questions

- Can this be done by inserting the tail of graphs in semver mode and the rest in replaces mode?
- When a bundle is added to the tail of the graph in replaces mode, is it required that the added bundle be replaced by the tail of the graph?

