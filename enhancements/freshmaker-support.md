---
title: Freshmaker-support
authors:
  - "@gallettilance"
reviewers:
  - "@ecordell"
  - "@bparees"
  - "@kevinrizza"
approvers:
  - "@ecordell"
  - "@bparees"
  - "@kevinrizza"
creation-date: 2020-11-11
last-updated: 2020-11-11
status: implementable
---

# OLM Support for CVE patches

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift/docs]

## Summary

Freshmaker is an internal service that rebuilds container images based on CVEs. When an operator/operand image that Freshmaker built is shipped, Freshmaker must rebuild all bundle images that refer to the old operator/operand image. This enhancement is concerned with how these bundle images should be rebuilt such that they are compatible with existing and future update

### Goals

The bundle rebuilt by Freshmaker should:

- create an update graph that never allows for an update from a CVE-patched version to a CVE-affected version
- allow for CVE-affected versions to **update to** the CVE-patched version
- allow for ealier versions to **update to** the CVE-patched version instead of the CVE-affected version
- allow for newer versions to **update from** the CVE-patched version

### Requirements

1. Must make freshmaker usable by all portfolio teams
2. Must have minimal impact on portfolio teams’ workflow for releasing
3. Must be compatible with 4.5, 4.6, and 4.7
4. Must be compatible with the refresher job (i.e. **batch add**ing multiple bundles received from the ListBundles grpc API and **deprecating** specific bundles by sha reference)
5. Must provide proper semver ordering to not block a transition to semver in the future

## Proposal

A new field is added to the operator metadata:

`annotations.yaml`:
```yaml
annotations:
  operators.operatorframework.io.bundle.release.v1: 1
```

It is assumed that this value will also be available in a `LABEL` on the bundle image, and that the value is an integer that increases monotonically.

For the purposes of the resolver, the `release` is considered a `property`. Bundles without a release field or value are assumed to have the release value of `0`.

In 4.7, the `ListBundles` API (the operator-registry API that serves the list of bundles to the OLM resolver) will be augmented to include the `release` value. The resolver MUST ignore packages with the same properties, but that have `release` values that are not the latest. 

The registry MAY remove older releases entirely to reduce index size and resolver load. 

In 4.6, the `ListBundles` API will be augmented to filter out non-latest releases. No changes will be needed to the 4.6 OLM resolver.

To work on 4.5, the bundles exported from the 4.5 index need to be loadable with the 4.5 initializer. The opm export command needs to be modified to mutate CSVs with equal versions but different releases such that they reference each other in a proper replaces / skips semantic.

#### Example Freshmaker rebuild flow

Given the following index graphs:

```
4.5
release-1.0:
    1.0.0 -> 1.0.1 -> 1.0.2 -> 1.0.3
```

```
4.6
release-1.0:
    1.0.0 -> 1.0.1 -> 1.0.2 -> 1.0.3
release-1.1:
    1.1.0 -> 1.1.1
```

```
4.7
release-1.0:
    1.0.2 [depr] -> 1.0.3
release-1.1:
    1.1.0 -> 1.1.1
```

If a CVE is detected on `1.0.2`, Freshmaker rebuilds the operator image of `1.0.2` and replaces the image pull spec in the `1.0.2` bundle with the pull spec of the rebuilt operator image.

If a `release` field isn't present in the annotations.yaml file of the `1.0.2` bundle, then Freshmaker appends

```
    operators.operatorframework.io.bundle.release.v1: 1
```

to the annotations.yaml. If a `release` field is present, then Freshmaker bumps that field to its next incremental integer value. Freshmaker duplicates that process to add or update the `release` field in the container image labels.

The new release of the `1.0.2` bundle can then be built and added to the same set of indexes that the previous `1.0.2` release was added to. It should create the following update graphs:

```
4.5
release-1.0:
    1.0.0 -> 1.0.1 --1.0.2-> 1.0.2 (release 1) -> 1.0.3
```

```
4.6
release-1.0:
    1.0.0 -> 1.0.1 --1.0.2-> 1.0.2 (release 1) -> 1.0.3
release-1.1:
    1.1.0 -> 1.1.1
```

```
4.7
release-1.0:
    1.0.2 [depr] -> 1.0.2 (release 1) [depr] -> 1.0.3
release-1.1:
    1.1.0 -> 1.1.1
```


### Impact of Requirements

Requirement 3 essentially constrains the solution to operator-registry and the opm binary since these can be easily used and switched out in the pipeline. Relaxing this requirement changes the scope of the problem.

#### First class support on 4.6.z

Backporting the resolver change introduced in 4.7 to 4.6.z has the following impact:

- alignment with the philosophy behind the OLM resolver: it's the OLM resolver's responsibility to determine which set of bundles are installable - not the operator-registry's.
- first class implementation of `releases` means easier debugging / better support on 4.6 (important especially since 4.6 is LTS).
- If dependencies cannot be resolved by non-CVE-affected versions, relaying that to the user is more  challenging if the OLM resolver does not have access to all the information.
 

#### Certain updates unsupported in 4.5

Since no changes to the csv's replaces / skips were made during rebuild, all releases would fit at the exact same spot in a 4.5 catalog built from app-registry. Supporting 4.5 means we need to mutate the CSVs during export such that they can reference each other in a replaces / skips way so that they can be loaded into a catalog built from app-registry.

If we decide updating from a CVE-affected version to a CVE-patched version is not supported in 4.5, we can simply modify the export command to only export the latest releases for a given version. Note that updates:

- from previous versions to CVE-patched versions
- from CVE-patched versions to next versions
- from CVE-affected versions to next versions

are all still possible. To get updates from CVE-affected to CVE-patched, you would need to be using the index image on cluster instead of the catalog created from app-registry.

---

## User Stories

### User Story 1

`operators.operatorframework.io.bundle.release.v1` field should be added to the annotations.yaml spec / type.

Impact:
- changes to the api repository (?)
    - if so, then changes to operator-sdk to bump the dep are needed
    - if so, then changes to CVP are needed to bump to new operator-sdk
- changes to the operator-registry repository

### User Story 2

Index add should handle adding a new release of a bundle in replaces mode.

### User Story 3

Index batch add should order equal versions by their release number.

### User Story 4

OLM resolver should understand the `release` field and handle reconciliation / update when CSV names are equal.

### User Story 5

New releases of versions marked as deprecated should also be marked as deprecated at index add time.

### User Story 6

Exporting from an index should either (depends on requirement 3):

- only export the latest releases of each version
- generate replaces / skips between all releases of a given version such that it can be loaded using the 4.5 initializer

