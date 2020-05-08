---
title: index-init
authors:
  - "@kevinrizza"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-05-08
last-updated: 2020-05-08
status: implementable
---

# index-init

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

The purpose of this enhancement is to define a new function for `opm index` which would generate initial index images with empty databases. Today, this can technically be accomplished by using `opm index add -b ""`, but semantically that is very unintuitive. The intention of this enhancement is to define a separate command, `opm index init`, to bootstrap empty index images . 

## Motivation

The primary motivating driver for this is to give CI pipelines utilizing `opm` a way to differentiate between initializing an index image and actually adding bundles to that initial image. From a CI tool's perspective, validation and rules would reasonably expect that `--from-index` should always be set, which means that the initial bootstrapping process becomes necessary. However, rather than expect users to use `opm index add` in an unintuitive and semantically confusing way, we should instead enable this as a first class feature.

### Goals

* `opm index init` can generate fresh empty initial index images

### Non-Goals

* Remove the ability to bootstrap an index with `opm index add -b` with a fresh index image. This would simply allow the generation of specific index images.

## Proposal

Basic approach:

- Create a new subcommand `opm index init`
- Include all of the existing image customization flags that exist on `opm index add` to allow users to configure the base image
- Implement this by adding another library function to the registry lib which initializes the database without attempting the insert function

### Customization:

The following flags would be included on `index init`:

- `--generate` - generate the database and dockerfile to build separately
- `--binary-image` - base opm binary image to build from
- `--container-tool` - tool to interact with container images
- `--build-tool` - tool to build image from
- `--pull-tool` - tool to pull image from (only used if binary-image is also set)
- `--tag` - image tag for output container image

### Registry lib change

Add a new request type to the registry interface, `InitRegistryRequest` that inits a new database and returns. For now, that lib function would not need to be explicitly exposed by an `opm registry` subcommand, and would instead just be wrapped by the indexer.

## Migration and Backwards Compatibility

Explicitly, this does not change the behavior of `opm index add` when adding initial bundles when `--from-index` is not specified. That bootstrapping behavior is and should be retained in order to preserve existing workflows.
