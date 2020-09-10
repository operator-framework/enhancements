---
title: Tool for listing malformed bundles from a Catalog Source
authors:
  - "@harishsurf"
reviewers:
  - "@kevinrizza"
  - "@ecordell"
  - "@exdx"
approvers:
  - "@kevinrizza"
  - "@ecordell"
creation-date: 2020-06-10
last-updated: 2020-06-10
status: implementable
see-also:
  - "N/A"  
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Tool for listing malformed bundles from a CatalogSource (client-side validation)

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary
The goal of this enhancement is to create a tool that given a [catalog source](https://olm.operatorframework.io/docs/concepts/crds/catalogsource/) image, it lists all malformed bundles. This tooling is achieved by extending `opm` CLI in [operator-registry](https://github.com/operator-framework/operator-registry) 
> Note: Malformed bundles are those that do not meet the static validation rules.


## Motivation
A cluster administrator or release pipeline owner would want a way to validate a catalog source image to ensure that it does not have any "bad" bundles that could potentially harm the cluster. This tool can help them keep the cluster health in check and prevent unexpected cluster failure.


### Goals

- Design a tool that lists malformed bundles for a given [catalog source](https://olm.operatorframework.io/docs/concepts/crds/catalogsource/) image.
- The tool should have a resumable workflow - canceling and running the same command should start from where it left off.
- A way to validate only the latest channels within the catalog source.


### Non-Goals

- The tool cannot be used for on cluster validation.
- It doesn't remove malformed bundles from the catalog source image.

## Open Questions
    
1. `opm index export` also does unpacking of an index image, but the downloaded content is in the old app-registry format. Should the `ExportFromIndex` API be modified to align downloaded content to be in the new bundle format and reuse that to validate catsrc image?
2. Should the tool validate only the default channel head for each package by default and use a `bool` (`--latest`) flag to validate all the bundles if needed or the other way around?


## Proposal

### User Story

As a cluster administrator or release pipeline owner, I would like to be aware of malformed Operator bundles so that I can prevent them from being released into a catalog for cluster tenants to see.

### Design Details

The basic approach will be to extend [`opm`](https://github.com/operator-framework/operator-registry/blob/master/docs/design/opm-tooling.md) tooling to: 
- Add a new command: `opm index validate --index <index image>` - which unpacks and validates all of the bundles for a given index image.
- The new command will consume a new API `ExportFromIndexWithCache` which includes: 
    - parts of `opm index export` which does the unpacking but modify it to download manifest in the new [bundle format](https://github.com/operator-framework/operator-registry#manifest-format). 
    - The API will also provide the ability to cache bundle images which are required to have a resumable workflow (in case of interruptions).
- Once the unpacking is done, the [`opm alpha bundle validate`](https://github.com/operator-framework/operator-registry/blob/master/cmd/opm/alpha/bundle/validate.go#L84) command would be run for each of the bundles within the downloaded directory. 


Things to consider:
- Exporting of an index image can be time-consuming(at times) and so we need to provide a control option to resume extraction from its previous state.
- There should be an option to unpack only the latest version from each package from a catalog image.

#### Add a new `opm index validate` command  

```shell=
$ opm index validate --index=quay.io/operator-framework/upstream-community-operators:latest --latest --download-folder=./bundles --container-tool=docker
```
- The `--index` points to the catalog source image which has the running index image.
- The optional `--latest` flag when specified, it will only validate default channel heads for each package.
- The optional `--download-folder` flag points to a directory location where the extracted file(s) gets stored. When omitted, it will unpack into a default folder.
- The optional `--container-tool` to be used for unpacking, one of [none, docker, podman].

#### Having a resumable workflow

During the process of unpacking which could at times take longer depending on no. of packages that are present in a given index image, if there is any unforeseen interruption, in order to have a resumable workflow, the following approach is taken: 
- Introduce a new API `ExportFromIndexWithCache` which reuses parts of [`ExportFromIndex`](https://github.com/operator-framework/operator-registry/blob/master/pkg/lib/indexer/indexer.go#L479). This new API would: 
    - Get the database file from the index image. 
    - Fetch and cache each of the bundle image along with other manifests from the database file.
    - Store each of the bundle image along with its sha in a data struture before unpacking and keep adding the unpacked package names and its corresponding new sha (after the `Pull`) into another data structure such that in case of interruptions, the API would do a diff of both data structures and add only those that are not already unpacked. It would also do a diff of shas to reload bundles that were interrupted midway.

#### Option to validate only the latest channel heads 

This option allows the user to validate only the latest bundle versions (default channel heads) for each package.
- It can be enabled by setting `--latest` boolean flag
- A new Query API `GetLatestBundles` would be added which fetches all the bundles from `operatorbundle` table that have not null entries for both `csv` and `bundle` column - which gives us only the default channel bundles.
- Once the packages are known, they are unpacked and validate as explained above.

### Test Plan

The Operator registry's e2e testing suite will be upgraded to handle the following use cases:

- Validate all the bundles in an index image.
- Validate only the `latest` channel for each package from the given catalog image.
 
## Future Work
- Have server-side/on-cluster validation. [APIServer dry-run](https://kubernetes.io/blog/2019/01/14/apiserver-dry-run-and-kubectl-diff/) could be a viable option.