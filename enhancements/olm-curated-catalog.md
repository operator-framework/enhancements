---
title: olm-curated-catalog

authors:
  - "@cdjohnson"

reviewers:

approvers:

creation-date: 2020-03-19

last-updated: 2020-03-19

status: 

see-also:
  - "/enhancements/opm-mirroring.md"  

replaces:

superseded-by:
---

# OLM Curated Catalog

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions
- Where is the best place to physically mirror images.  `opm` or `oc`?
  - It appears that [opm-mirroring.md](opm-mirroring.md) was implmented as an OpenShift enhancment to: `oc adm catalog mirror`


## Summary

This enhancement outlines the changes necessary for `opm` to support curated catalog indexes to support distribution of portions of existing OLM catalogs for disconnected environments. This relies on the [Disconnected OLM enhancement](opm-mirroring.md).

## Motivation

There are two primary motivations:

### Disconnected Catalog Mirroring
Disconnected installation of OLM catalogs can be done with the following OpenShift Container Platform `oc` command:

```
$ oc adm catalog mirror quay.io/openshift/community-operators-catalog:latest disconnected-registry:5000/community-operators/

```

This will mirror the entire catalog to the disconnected, local registry.  As catalogs begin to include hundreds of operators, versions of operators, versions of operands and multiple architectures, it will not be feasible to mirror the entire online catalog.

### Curated Catalogs for Users
Companies would like to be able to limit the list of operators to those that are endorsed or supported by the customer and their lines of business.

Cluster administrators can build multiple catalog images and install them in different namespaces based on company policies.   Although, by itself, doesn't disable the ability to install other operators in other catalogs, it does provide a better user experience.

When used with the os/architecture filter, this MUST be used in conjunction with a disconnected, local image registry.  Since multi-architecture images employ a manifest index (also known as a manifest list or fat manifest), the digest of that image includes the digests of ALL supported os, architecture and variants within that image.  If an image is added or removed from the manifest index, then the digest changes.  In the case of an os/architecture filter, a subset of the manifest index will be created, or a _sparse manifest index_.  This manifest must be created in the local, disconnected image registry since the end-user will not be able to push this to the public image registry, and it isn't feasible for the public image owner to generate all known combinations of these index manifests. 


### Goals

* Define `opm` commands which can build or update a catalog index image from one or more other catalog index images using the following filters:
  * Catalog Index Image
  * Operator Package Name
  * Channels
  * Versions
  * OS/Architecture

### Non-Goals
* This assumes that the os and architecture values in the manifest image index are consistent between the image repository and the CSV, using the [GOARCH](https://golang.org/doc/install/source#environment) and [GOOS](https://golang.org/doc/install/source#environment) constant values.
* Enhance or improve the Docker or OCI registry specifications to support sparse manifest indexes.

## Proposal

The basic approach will be to:

- Generate catalog index container images that contain a subset of existing catalog container images
- Provide the ability to merge catalogs together, to allow periodic mirroring of catalogs that are updated over time.
- When filtering by version:
  - Update any broken graph links with a subset of the new, sparse graph
- When filtering by os/arch:
  - Update the CSVs in the catalog index to have updated `relatedImages` references to the sparse manifest indexes 
  - Update the CSVs in the catalog index to have updated `operatorframework.io/arch` and `operatorframework.io/os` labels.
  - This requires the multi-architecture images to be mirrored to a repository with the appropriate sparse manifest index created.


### Generate a curated catalog

`opm index build` will be created with flags:

```sh
$ opm index build \ 
  --from-index  "quay.io/openshift/community-operators-catalog:latest"
  --package etcd \
  --filter-by-channel "beta|stable"\
  --default-channel "stable" \
  --filter-by-version=">=0.9.2" \
  --filter-by-os="linux\/amd64|linux\/ppc64le" \
  --tag="disconnected-registry:5000/catalogs/curated-catalog:latest"
  --merge-index
```

It will have the following additional flags:
- `--from-index` - the index image to use as the source to retrieve operators from.
- `--package` - the name of an operator package to include.
- `--filter-by-channel` - a regular expression of one or more channels to include from the package.
- `--default-channel` - the default channel to set for the package.
- `--filter-by-version` - a semver range identifying a subset of the version graph (DAG) to include from the package.
- `--filter-by-os` - a regular expression of operating system and architectures to support.
- `--merge-index` - the index image to merge the filtered operator versions into.

The following flags are inherited from the `opm index add` command:
```
  -i, --binary-image opm        container image for on-image opm command
  -c, --container-tool string   tool to interact with container images (save, build, etc.). One of: [docker, podman] (default "podman")
      --generate                if enabled, just creates the dockerfile and saves it to local disk
  -h, --help                    help for add
  -d, --out-dockerfile string   if generating the dockerfile, this flag is used to (optionally) specify a dockerfile name
      --permissive              allow registry load errors
  -t, --tag string              custom tag for container image being built
```

`opm index build` will be enhanced to:

- Retrieve the source Catalog Index Image
- Extract the Catalog Index Image that match the following filters into a separate directory representing an Operator Bundle (CRDs, CSVs, Metadata).
  - Package Name (the operator name)
  - Channel names
  - Version range to include (semver range)
- If the `filter-by-version` is specified:
  - Remove any `replaces` versions for the beginning of the graph, if the previous versions are excluded.  This builds a new, sparse graph for the entire package.
- If the `filter-by-os` filter is specified:
  - For each extracted bundle:
    - If the CSV specifies all of the filtered `os/arch/variant` labels (see [the olm-book](https://github.com/operator-framework/olm-book/blob/master/docs/multiarch.md)):
      - Remove any `os/arch/variant` that are not specified in the filter.
      - For each `relatedImage`
        - Update the `relatedImage` reference with the digest of the, new sparse manifest index:
          - Retrieve the full manifest index
          - Construct a new, sparse manifest index with ONLY the specified architecture manifest images and retrieve the digest.
            - `oc image mirror --filter-by-os`, for example, has the ability to create a sparse manifest index
            - `skopeo manifest-digest`, for example,  has the ability to retrieve the manifest index digest.
          - Calculate the digest for the sparse manifest list.
            - The manifest index is discarded and will be constructed during the mirroring process.
          - Update the `relatedImage` digest to include a `sparseImage` key with the same repository, but the updated digest of the calculated sparse image
- For each extracted bundle:
  - Create the bundle image
    - `operator-sdk bundle create` has the ability to create a bundle image
- For each extracted package:
  - Add all bundles to the catalog image:
    - `opm index add --from-index <idx> --bundles <bundles>` has the ability to add a bundle image to a catalog index image
  
The resulting `--tag`ged image now contains a functional catalog index with only the filtered graph of operators and references to both the original manifest index and the sparse manifest index. 

An example of a multi-architecture CSV before and after filtering to one architecture:

Before:
```
labels:
    operatorframework.io/os.linux: supported
    operatorframework.io/arch.amd64: supported
    operatorframework.io/arch.ppc64le: supported
    operatorframework.io/arch.s390x: supported

  relatedImages: 
  - name: default
    image: quay.io/coreos/etcd@sha256:12345 
  - name: etcd-2.1.5
    image: quay.io/coreos/etcd@sha256:12345 
  - name: etcd-3.1.1
    image: quay.io/coreos/etcd@sha256:12345 
```

After:
```
labels:
    operatorframework.io/os.linux: supported
    operatorframework.io/arch.amd64: supported

  relatedImages: 
  - name: default
    image: quay.io/coreos/etcd@sha256:12345 
    sparseImage: quay.io/coreos/etcd@sha256:56789 
  - name: etcd-2.1.5
    image: quay.io/coreos/etcd@sha256:12345 
    sparseImage: quay.io/coreos/etcd@sha256:56789 
  - name: etcd-3.1.1
    image: quay.io/coreos/etcd@sha256:12345 
    sparseImage: quay.io/coreos/etcd@sha256:56789 
```

### Mirroring a curated catalog

To mirror the curated catalog index image, the mirroring technology, for example, `oc adm catalog mirror` or `opm registry images` with `oc image mirror` must support the ability to create the same manifest index that was generated during the build process.

Example using `opm` and `oc`
```
$ opm registry images --from=disconnected-registry:5000/catalogs/curated-catalog:latest --to=disconnected-registry:5000/operators --manifests=./mirror-manifests | xargs oc image mirror --filter-by-os="linux\/amd64|linux\/ppc64le"
```

In this example, `opm` will gather a list of all of the images and sparse images and feed them one-by-one into `oc image mirror`, which has the ability to automatically build the sparse index manifest when mirroring, when using the `--filter-by-os` argument.

Example using `oc adm catalog mirror`:
```
$ oc adm catalog mirror \
    disconnected-registry:5000/catalogs/curated-catalog:latest \
    disconnected-registry:5000/operators  \
    --filter-by-os="linux\/amd64|linux\/ppc64le"
```

```
oc adm catalog mirror \
    <registry_host_name>:<port>/olm/redhat-operators:v1 \
    <registry_host_name>:<port> \
    [-a <path_to_registry_credentials>] \
    [--insecure] 
    [--filter-by-os=<regex of os/arch]
```

When the `--filter-by-os` option is specified:
- For each package
  - For each CSV
    - If the CSV specifies all of the filtered `os/arch/variant` labels:
      - for each `relatedImages` image:
        - Mirror the `image` using `oc image mirror --filter-by-os`
          - This will copy the selected images and recreate the same manifest index with the digest.
          - The digest will match that specified in the `sparseImages` attribute and the annotations in the install deployment.
- Generate the `mapping.txt` file as before.
  - TODO: Verify that this works with the `--filter-by-os`

# TODO: FOLLOWING SECTIONS TO BE FILLED IN FROM TEMPLATE

### Implementation Details/Notes/Constraints [optional]

What are the caveats to the implementation? What are some important details that
didn't come across above. Go in to as much detail as necessary here. This might
be a good place to talk about core concepts and how they releate.

### Risks and Mitigations

## Design Details

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:
- Maturity levels - `Dev Preview`, `Tech Preview`, `GA`
- Deprecation

Clearly define what graduation means.

#### Examples

These are generalized examples to consider, in addition to the aforementioned
[maturity levels][maturity-levels].

##### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers

##### Tech Preview -> GA 

- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

##### Removing a deprecated feature

- Announce deprecation and support policy of the existing feature
- Deprecate the feature

### Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to make use of the enhancement?

### Version Skew Strategy

How will the component handle version skew with other components?
What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- During an upgrade, we will always have skew among components, how will this impact your work?
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI, CRI
  or CNI may require updating that component before the kubelet.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used to
highlight and record other possible approaches to delivering the value proposed
by an enhancement.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources
started right away.

