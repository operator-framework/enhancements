---
title: thindex-images
authors:
  - "@ecordell"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-04-25
last-updated: 2020-04-25
status: implementable
---

# Thindex Images

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

## Summary

Operator Lifecycle Manager (OLM) currently talks to registry images (aka index images), containers which contain the registry database content and the grpc interface that serves data to consumers.

This has the advantage of being very portable - index images can be moved from one cluster to another and easily mirrored.

However, there is a major drawback. Because the server is embedded in the registry, care must be taken to ensure that the server is used with a compatible version of OLM (one that consumes the same version of the grpc api). 

This enhancement describes a mechanism to decouple the server from the content being served, so that index images can be used forward-compatibly (in clusters with OLM installed at a newer version than the index was created for). Existing images can be made into thindex images by annotating them with the metadata described below (such an image could function as a version-pinned index image, or a floating thindex image.)

## Motivation

Decouple the server components from the data served via index images today.

## Proposal

Define metadata that follows the [bundle image spec](https://github.com/openshift/enhancements/blob/master/enhancements/olm/operator-bundle.md).

This matadata will be available both as labels on the final image, and inside the image in an `annotations.yaml` file:

```Dockerfile
LABEL operators.operatorframework.io.index.mediatype.v1="index+v1"
LABEL operators.operatorframework.io.index.path.v1="database/index.db"
LABEL operators.operatorframework.io.index.metadata.v1="metadata/"
```

It is expected that `annotations.yaml` or `annotations.json` lives in the directory indicated by `operators.operatorframework.io.index.metadata.v1` and which contains the same information:

```yaml
annotations:
  operators.operatorframework.io.index.mediatype.v1: "thindex+v1"
  operators.operatorframework.io.index.path.v1: "database/index.db"
  operators.operatorframework.io.index.metadata.v1: "metadata/"
```

The `opm` tool (see [operator-registry](https://github.com/operator-framework/operator-registry)) will have tooling that supports building thindex images and keeping them in sync.

Optionally, a migration identifier can be added to the metadata to indicate at what migration the database was constructed:

```Dockerfile
LABEL operators.operatorframework.io.index.migration.v1="6"
```
```yaml
  operators.operatorframework.io.index.migration.v1="6"
```

This metadata is used to define a `thindex image` as an image containing just a registry database file.

Below is an example directory layout for a thindex image:

```sh
$ tree
/
├── data
│   └── index.db
└── metadata
    └── annotations.yaml
```

On-cluster, we will define a new CatalogSource type:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: example-manifests
  namespace: default
spec:
  sourceType: thin
  image: example-thindex:latest
```

OLM will run a pod that contains the registry server (that is pinned to the version of OLM that is running) and the thindex pod, containing the data to be served.

### User Stories

#### Forward-compatible indexes

As an OLM user, I would like index images built with older versions of OPM to continue working on newer versions of OLM, so that I do not have to manually rebuild them.

#### Use on thindex

As an OLM user, I would like to use thindex images in a cluster as an image for CatalogSources.

Note: This user story is the motivation for the requirement that image labels be replicated inside the image as `annotations.yaml`. On-cluster we will not have access to the labels on the image, but OLM will need to make use of the metadata to find and use the database. A similar issue drives the requirements for bundle images.

### Implementation Details/Notes/Constraints 

Note that this is not necessary for any content shipped via Red Hat, which is continually built using tooling pinned to corresponding OCP (and therefore OLM) versions.
