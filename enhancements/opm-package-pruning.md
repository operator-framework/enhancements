---
title: opm-package-pruning
authors:
  - "@djzager"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-04-03
last-updated: 2020-04-03
status: implementable
---

# OPM Package Pruning

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

Add support to the `opm` command line utility to prune operators from an
operator index (or registry). This enables consumers of operator indexes to take
an existing, and presumably large, index and generate a new index with only the
operators they wish to make available in the catalog. In short, this is the
inverse of `opm (index|registry) rm` where you select which operators to remove,
`opm (index|registry) prune` allows you to select the operators to keep.

## Motivation

As Operator adoption expands and OLM usage increases the index images used to
make operator catalogs available to OLM will grow. Cluster administrators
__should__ have a way to curate operator catalogs. This feature provides a basic
mechanism for catalog curation.

### Goals

- Allow `opm` user to select operators to keep from an existing registry or
  index image.

### Non-Goals

- Manipulate the available channels of any operator in the registry or index
  image.

## Proposal

### Prune Subcommand

Adding the `prune` subcommand to the `index` and `registry` command provides a
simple mechanism for selecting operators to keep from an index image or a
registry database (depending on the root command).

### User Story

#### Pruning an Existing Operator Catalog

As a cluster administrator, Billy knows what operators from an existing operator
index he wants to make available to his cluster's users. Billy can build a new
operator index by selecting operators from the previous index to keep.

### Implementation Details

#### New Commands

A `prune` subcommand will be added to both the `index` and `registry` command
mimmicking the behavior of `rm` in all but one significant way. The `packages`
flag will represent the packages to be kept in the index or registry.

## Design Details

### Test Plan

e2e tests will be added alongside existing tests for the `opm` tool verifying
the ability to prune an index.
