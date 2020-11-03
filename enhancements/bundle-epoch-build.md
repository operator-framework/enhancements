---
title: Bundle-Versioning-with-Epoch-and-Release
authors:
  - "@ecordell"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-11-03
last-updated: 2020-11-03
status: provisional
---

# Bundle-Versioning-with-Epoch-and-Release

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

1. Do we want to store older `release`s of bundles in an index?

## Summary

Add two new fields for controlling bundle version selection: `epoch` and `release`.

## Motivation

### `release`

Allows providing a newer version of the same exact release. This is particularly useful for in-place updates due to CVE respins, but may also be useful in a continuous build environment, like CI or a staging repo.

Examples from other packaging systems include the [rpm `release` field](https://rpm-packaging-guide.github.io/#what-is-a-spec-file) and [deb's `debian_revision` field](https://www.debian.org/doc/debian-policy/ch-controlfields.html#version)

## `epoch`

Allows renumbering / reversioning of a package. Though it should be used rarely, it will be useful if a package is tracking an upstream project's versioning that changes (for example: `docker` switching from semver to time-based versions).

Examples from other packaging systems include the [rpm epoch](https://rpm-packaging-guide.github.io/#epoch) and the [dpkg epoch](https://wiki.debian.org/Teams/Dpkg/FAQ#Q:_What_are_version_epochs_and_why_and_when_are_they_needed.3F).


### Goals

- Provide a mechanism to provide newer versions of the same release (semantically)
- Provide an escape hatch for when versioning scheme needs to change completely

## Proposal

### `release`

A new field is added to the operator metadata:

`annotations.yaml`:
```yaml
annotations:
  operators.operatorframework.io.bundle.release.v1: 1
```

It is assumed that this value will also be available in a `LABEL` on the bundle image, and that the value is an integer that increases monotonically.

For the purposes of the resolver, the `release` is considered a `property`. Bundles without a release field or value are assumed to have the release value of `0`.

`ListBundles` will be augmented to include the `release` value. The resolver MUST ignore packages with the same properties, but that have `release` values that are not the latest. 

The registry MAY remove older releases entirely to reduce index size and resolver load. 

## `epoch`

A new field is added to the operator metadata:

`annotations.yaml`:
```yaml
annotations:
  operators.operatorframework.io.bundle.epoch.v1: 1
```

It is assumed that this value will also be available in a `LABEL` on the bundle image, and that the value is an integer that increases monotonically.

For the purposes of the resolver, the `epoch` is considered a `property`. Bundles without an epoch field or value are assumed to have the epoch value of `0`.

`ListBundles` will be augmented to include the `epoch` value. Given two bundles with different epoch values, the resolver MUST treat the bundle with the higher epoch value as greater (in version order) than any bundle with a lower epoch value.
