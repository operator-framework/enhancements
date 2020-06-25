---
title: ImageStream Bundle Proxying
authors:
  - "@njhale"
reviewers:
  - "@shawn-hurley"
  - "@kevinrizza"
approvers:
  - "@benluddy"
creation-date: 2020-06-25
last-updated: 2020-06-25 
status: provisional
---

# imagestream-bundle-proxying

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

- External images can be pulled into OpenShift’s integrated image registry
- Pulling an external image to the integrated registry can be triggered by creating/updating an ImageStream resource
- No non-standard cluster configuration is needed to allow external images to be pulled into the integrated registry
- Pulls of external images into the registry abide by ImageContentSourcePolicy and other applicable cluster configurations
- Direct -- non-pod spec -- pulling of images from the integrated registry transitively satisfies ImageContentSourcePolicy constraints

## Summary

When deployed to OpenShift, OLM can use an ImageStream to stage a bundle image in the internal image registry and access its image metadata while still meeting the OpenShift-specific constraint to pull images via CRI-O.

## Motivation

Today, when a bundle image is pulled and unpacked by OLM, a Job is created which specifies three containers:

- a bundle util init container
  - contains a statically linked bundle tool that can search the container’s filesystem for bundle metadata -- annotations.yaml -- and copy the referenced manifest files to a bundle volume
  - mounts a single bundle util volume
  - at runtime, uses cp to copy this bundle tool to the bundle util volume
- a bundle data init container
  - pull spec is used to delegate pulling of the bundle image via CRI-O
  - mounts two volumes: bundle util and bundle data
  - at runtime, executes the bundle tool from the bundle util mount to copy the bundle to the bundle data mount
- a bundle extract container
  - contains opm
  - mounts the bundle data volume
  - runs `opm extract` against the bundle data mount which results in a ConfigMap being created that contains the bundle’s kubectl-able content

While this works for both OpenShift and upstream Kubernetes, the inability to inspect the image annotations -- where the path of the metadata directory is specified -- means that we cannot verify that the bundle metadata directory found in the bundle’s filesystem contains the data intended by the author.

Using opm to directly interact with the image, instead of relying on a pod pull spec, allows for unambiguous resolution of the bundle metadata directory. In the future, this will also enable bundle data to be stored entirely as image annotations and/or manifest lists (w/o a run configuration or union filesystem).

### Goals

- Disambiguate the result of unpacking a bundle on-cluster
- When deployed to OpenShift, adhere to constraints imposed by ImageContentSourcePolicy when pulling bundle images

### Non-Goals

- Make ImageStreams a hard requirement for bundle unpacking on platforms other than OpenShift

## Proposal

### Implementation Details/Notes/Constraints

To unpack a bundle image, OLM will generate a job:

- with a single container spec
- that uses opm to directly pull the image (daemonless)

When deployed to OpenShift, before unpacking OLM will additionally:

- generate a respective ImageStream
- switch the target pull image to the respective mirror from the integrated registry

This means that opm will have access to the image metadata required to identify the proper bundle metadata directory.

