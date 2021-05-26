---
title: big-bundle-unpacking
authors:
  - "@njhale"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-08-12
last-updated: 2020-09-01
status: implementable
see-also: - "https://docs.google.com/document/d/1WWbZCyG1sw3vi-1gtMOshmaTaYiMht3cb-Up4NAx4_I/edit#"

---

# Big Bundle Unpacking

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

To ensure Operator Bundles of arbitrary size can be unpacked and used in a Kubernetes cluster by OLM, bundle content will be partitioned across several ConfigMaps. This unpacking behavior will be abstracted behind a new Kubernetes API to allow for future extensibility.

## Motivation

Currently, OLM stages Operator Bundles within a cluster before installation by writing their content into ConfigMaps. There is a 1:1 relationship between bundle and ConfigMap; i.e. each bundle is unpacked to its own ConfigMap. A limitation of this staging method is that the maximum size of a ConfigMap for a given k8s cluster becomes the upper bound for the size of a bundle installed to it.

### Goals

- Enable OLM to install operators from bundles containing more content than fits in a single ConfigMap

### Non-Goals

- Increase the upper storage bound for API resources in general
- Support bundles containing individual resources that surpass the upper storage bound of a cluster

## Proposal

A new Kubernetes resource kind will be introduced into the `porcelain.operators.coreos.com/v1` group version: `Bundle`. A `Bundle` resource will abstract the pulling and unpacking of an operator bundle from clients (OLM and cluster admins) and will feature the following fields:

| ----- | ----------- |
| spec.from | A union type which locates the bundle (e.g. a docker image digest) |
| status.conditions | [Standard k8s conditions](https://github.com/kubernetes/apimachinery/blob/94222d04a59075a01fddedd696037db9e61db6e9/pkg/apis/meta/v1/types.go#L1367), surfacing info about the unpack job |
| status.mediaType | Discriminates the type of bundle pulled (e.g. `registry+v1`) |

```yaml
apiVersion: porcelain.operators.coreos.com/v1alpha1
kind: Bundle
metadata:
  name: my-operator
  namespace: olm
spec:
  from:
    type: Image
    ref: quay.io/njhale/my-operator@sha256:123def...
status:
  mediaType: registry+v1
  conditions:
    type: UnpackFailed
    status: True
    reason: InvalidBundle
    message: "no manifests detected in bundle"
```

Most importantly, a `Bundle` will expose a `/content` subresource which -- upon a successful unpackaging -- will return a [JSON encoded representation]() of that bundle.

OLM will come equipped with a default aggregated API-server to provide the `porcelain` API group. The controller embedded in the default API-server will reconcile `Bundle` resources and spawn "unpack" `Job` resources similar to those currently employed, with a few key differences:

- Unpack `Jobs` will now discretize bundle contents into chunks that are then stored in distinct `ConfigMaps`. With a sufficiently small chunk size, a bundle that is too large to fit into a single `ConfigMap` can be stored using several
- When a bundle has been successfully unpackaged, the controller will ensure the `/content` subresource is served for the corresponding `Bundle`

### User Stories

#### OLM Can Unpack Big Bundles

As an operator author, I want OLM to be capable of unpacking and installing a bundle that contains an arbitrary number of resource manifests, each no larger than the limit for an individual API resource on a target cluster, so that the size of what I can package as an operator is no more limited than that target cluster.

### Implementation Details/Notes/Constraints

#### PodSpec Pull Requirement

In order to support pulling bundle images on OpenShift, OLM must adhere to the platform's [pull policies](https://docs.openshift.com/container-platform/4.5/openshift_images/image-configuration.html). The lowest risk way to do this is to rely on CRI to pull images and apply the policies opaquely, and the most straightforward initiate a pull is via a PodSpec. This means that, when a bundle image is pulled and unpacked by OLM today, a `Job` is created which specifies three containers:

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
  - runs `opm extract` against the bundle data mount which results in a `ConfigMap` being created that contains the bundle’s kubectl-able content

#### Partitioning and Storage

Instead of extracting bundle content directly to a `ConfigMap` -- as outlined in the [previous section](#podspec-pull-requirement) -- the unpack `Job` will first encode it as JSON and compress the result. The compressed JSON will then be split into chunks of a [fixed size](#determining-chunk-size), each wrapped with a header to keep track of its position in the chunk sequence. Each chunk is then encoded as JSON and stored in a respective `ConfigMap`. To each of these `ConfigMaps` a `porcelain.operators.coreos.com/bundle: <bundle-name>` label is applied to faciliate later retrieval. A `porcelain.operators.coreos.com/signature: <signature>` annotation is also generated for each chunk and is used to validate its integrity. When the content for a particular bundle is then retrieved, the `ConfigMaps` with the matching bundle name label are collected, sorted by the position tracked in each header, reconstituted into a single artifact, and decompressed before being returned for a `/content` request. If the signature verification fails for any chunk, a status condition indicating will be added to the associated bundle which will trigger a new unpack `Job` to run in a bid to rectify the issue.

#### Determining Chunk Size

Chunk-size will be determined by an option on the `catalog-operator` command, which if unset, directs the operator to probe the cluster for the resource storage limit. One potential probing strategy could be to create ConfigMaps of various sizes; akin to a "binary search" for the limit.

### Risks and Mitigations

Decompressing bundle content for each request may add significant time complexity. Some options for mitigating this are:

- building an [MRU cache](https://en.wikipedia.org/wiki/Cache_replacement_policies#Most_recently_used_(MRU)) of bundle content
- avoiding content compression entirely

## Design Details

### Version Skew Strategy

#### The `Bundle` API and OLM

The default aggregated API-server to service `Bundle` requests will be compiled and shipped w/ the rest of OLM; i.e. it will always match the version of OLM running on a given cluster.

#### Unpack Jobs and OPM

A small, statically linked binary will be generated from in-tree OLM source, for use in the unpack `Job`. This binary will vendor OPM packages to tightly couple content extraction logic to OLM version.

#### Extraction Logic and Operator Bundle Mediatypes

A given version of OLM will support unpackaging a fixed set of bundle mediatypes. If a bundle's mediatype is not within this set, an error will be surfaced in the status conditions of the respective `Bundle`. If OLM will drop support for a mediatype in an upcoming release, it will add a warning to the status conditions of all `Bundles` of said mediatype on a given cluster.

## Alternatives

- Instead of abstracting the unpackaging process behind a new API (`Bundle`), `opm` can be updated to partition bundle content across `ConfigMaps` directly. This would tightly couple OLM to one specific strategy.
- Cluster etcd or similar persistent storage can be granted to OLM. This would require OLM to be even more broadly trusted -- for etcd -- and/or be dependent on a secondary storage provider.
- Long-lived pods which directly serve bundle content over HTTP or gRPC
-
