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

OPM will support marking versions as deprecated for a given package - no bundles will be removed from the index during deprecation. Deprecated versions will not be installable. (Optional) If a deprecated version is installed prior to upgrade appropriate alerts will be fired on cluster.

## Motivation

The default set of Operators provided on OCP clusters will soon (4.6+) be delivered through indexes and each OCP minor version will have a unique set of index images. Today, packages in one index will be duplicated to the next minor version index when it is available unless the team know ahead of time that a bundle is not supported on a given OCP version. This means operator owners releasing through the redhat, certified, or community pipelines will need a way to specify which versions of their operator are incompatible or unsupported in the next minor version index. All deprecated bundles still need to be present in their respective indexes (even if uninstallable) in order for the bootstrap process of new OCP minor version specific index images from old index images to succeed.

### Terminology

#### Deprecated bundle

A deprecated bundle is an upgradable but not installable bundle in the update graph of an operator.

### Goals

- A deprecated bundle in the update graph and any of its replacements will not be installed by OLM
- Operator Owners don’t need to release a new version in order to specify which bundle is deprecated
- Index maintainers have a way to un-do deprecation if needed
- The deprecation is specified after promotion (not during) and before publishing of the index image
- (Optional) Alerts are fired on cluster when a deprecated version is installed

### Non-Goals

- OLM prevents cluster upgrades based on whether versions are deprecated in the upgraded cluster’s index

## Proposal

We propose to add a command to `opm index` called `deprecate` which takes in a set of bundle images, an index, and all the same arguments as `opm index add`. The command modifies the index by marking all versions that come before the specified bundle image in the update graph (that is, until the tail of each channel) as deprecated as well. The deprecation of bundles from the update graph will not modify the update graph after the deprecated version (from the deprecated channel to the head of the channel).

A new field will be added to the operator-registry database table `operatorbundle` called `deprecated` and the bundle images specified in the command will be marked as such.

The OLM resolver will be modified with a new property type called `olm.deprecated` in order to filter out any options that have this property set and disallow installs of deprecated bundles either directly or during upgrade. This also means appropriate status reporting in the subscription and installplan need to be added to OLM, and alerts will be fired if a deprecated version is installed.

**Note**: OLM cannot know whether an upgrade from a deprecated bundle to a non-deprecated bundle is supported / possible for a given operator. For example, given 1.0.0 -> 1.0.1 -> 1.0.2 -> 1.0.3, the team may have tested and implemented only sequential upgrades. Deprecating 1.0.1 from the update graph will mean installs on 1.0.0 will update directly to 1.0.2 which may not have been tested or implemented by the operator team.

### User Stories

#### Story 1

As an index maintainer I want to be able to deprecate bundles from the update graphs in order to prevent installs of incompatible or unsupported versions of certain operators.

##### Implementation Details

A new field is added to the operator-registry database table `operatorbundle` called `deprecated` and the bundle images specified in the command will be marked as such. Versions from the deprecated version to the tail of each channel are also marked as deprecated in the database.

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
    1.4.0 -- replaces -> 1.3.0 [deprecated] -- replaces -> 1.2.0 [deprecated] -- replaces -> 1.1.0 [deprecated] 
```

Deprecated versions are upgradable but not installable.

#### Story 2

As an index maintainer I want to be able to undo the above operation.

##### Implementation Details

A flag called `--undo` is added to the deprecation command that removes the deprecated property on the bundles specified.

For example, suppose following channel:

```
    1.4.0 -- replaces -> 1.3.0 [deprecated] -- replaces -> 1.2.0 [deprecated] -- replaces -> 1.1.0 [deprecated] 
```

we then deprecate version 1.3.0 using the following command

```
    $ opm index deprecate --bundles "quay.io/my/bundle:1.3.0" --from-index "quay.io/my/index:v4.7" --tag "quay.io/my/index:v4.7 --undo"
```

Now, in `quay.io/my/index:v4.7` the above channel looks like:

```
    1.4.0 -- replaces -> 1.3.0 -- replaces -> 1.2.0 -- replaces -> 1.1.0
```

#### Story 3

Semver and semver-skipatch modes support deprecation. Adding to an index that has deprecated bundles should appropriately reject or deprecate new additions.

For example, given:

```
    1.4.0 -- replaces -> 1.3.0 [deprecated] -- replaces -> 1.2.0 [deprecated] -- replaces -> 1.1.0 [deprecated] 
```

Inserting `1.2.1` into the above channel in semver mode should either produce an error or result in `1.2.1` being also marked as deprecated.

#### Story 4 (Optional)

As a cluster admin, I want to be alerted if subscriptions exist on cluster for deprecated versions of operators. The alert should be for:

- "Subscription is to a channel that no longer exists, or the head is deprecated"
- "A Manual subscription is on a deprecated version"

Eventually, we will want alerts for "Terminated version is installed" but this is outside the scope of this enhancemnent as it currently is not possible to distinguish between a terminated version and a version installed from a semver-skipatch index using skipRange.


