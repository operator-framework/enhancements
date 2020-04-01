---
title: olm-curated-catalog

authors:
  - "@cdjohnson"

reviewers:

approvers:

creation-date: 2020-03-19

last-updated: 2020-04-01

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
- Should the filter spec be YAML, JSON, both (auto-detect)?  I think we should support both automatically like kubectl does.
- Should the filter use patterns/expressions for the channel filters?
- Verify it's OK that we create new bundle images on the fly.  I don't see a way around this unless unless we remove the bundle image references from the index database.
- Consider how the filterspec could be used as a Kube manifest in the future.  I can imagine creating an operator running in a micro-bastion-cluster that keeps a docker registry synchronized.

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


### Goals

* Define `opm` commands which can build or update a catalog index image from one or more other catalog index images using the following filters:
  * Catalog Index Image
  * Operator Packages
  * Channels
* Support `opm registry` and `opm image` use cases

### Non-Goals



## Proposal

The basic approach will be to:

- Generate catalog index container images that contain a subset of one or more existing catalog container images
- Provide the ability to merge catalogs together, to allow periodic mirroring of catalogs that are updated over time.

### Generate a curated index catalog

`opm index filter` will be created with flags:
- `-f, --filter-spec` - a path to a filter YAML file

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

Examples:
```sh
$ opm index filter -t dockerreg.local/catalogs/company-catalog:latest -f ./filter.yaml
```

or
```sh
$ cat filter.yaml | opm index filter -t dockerreg.local/catalogs/company-catalog:latest  -f -
```

The `filterSpec` yaml specification is as follows:

* `filterSpec` (array of objects): array of filter specification objects as follows
  * `indexImage` (string): the catalog index image refererence
  * `filter` (array of objects):  array of filter objects as follows:
    * `package` (string) : the name of the operator package to include.
    * `includedChannels` (array of string): the array of channel names to include from the package (Optional, defaults to all channels)
    * `excludedChannels` (array of string): the array of channels to exclude (Optional, defaults to no excluded channels)
    
    This is useful to use to support future channels without updating the filter and takes precedence over the `includedChannels`
     

 "quay.io/openshift/community-operators-catalog:latest"

Example (in yaml for readabiilty)
```yaml
filterSpec:
- indexImage:  "quay.io/openshift/community-operators-catalog:latest"
  filter:
  - package: etcd
    # Include only the clusterwide channels.
    includedChannels:
    - beta-clusterwide
    - stable-clusterwide
  - package: postresql
    # Include only the stable channel.
    includedChannels:
    - stable
- indexImage:  "mydockerreg.local/myapp-catalog:latest"
  - package: myapp1
    # No includedChannels or excludedChannels, implies ALL channels
  - package: myapp2
    includedChannels:
    # Include any 1.1.x and 2.x operators
    - v1.1
    - v2
  - package: myapp3
    # Include every channel except for the 1.x and 2.x beta channels
    excludedChannels:
    - v1
    - v2-alpha
    - v2-beta
```
#### Filter Flow
`opm index filter` will be added to perform the equivalent steps (non-optimized):

- For each `filterSpec` as `FSPEC`:
  - Extract the source `FSPEC.indexImage` into a temp file structure as follows:
    ```
    catalog
      package
        bundle
          metadata/annotations.yaml (with channels info)
          ...
    ```
  - Delete any packages and bundles that don't match the filter:

    For each extracted `catalog.package` as `PKG`:
    - if `PKG` NOT IN `FSPEC.filter.packages`, delete the package
    - else: 
      - for each `PKG.bundle.channels` AS `CHN`
        - if `CHN` IN `FSPEC.filter.PKG.excludedChannels` OR (`FSPEC.filter.PKG.includedChannels` != Empty AND `CHN` NOT IN `FSPEC.filter.PKG.includedChannels`)
          - delete `PKG.bundle`
      - Create the bundle image with only the intersecting `PKG.bundle.channels` and the filtered channels as `NEWBNDL`
        - `operator-sdk bundle create` and `opm alpha bundle generate` have the ability to create a bundle image
          - The bundle images also need to be pushed to a docker registry, ideally teh same registry where the curated image will be located.
        - `opm index add --from-index <curated-idx> --bundles NEWBNDL` has the ability to add a bundle image to a catalog index image
- Tag and Push the curated index image specified by `--tag`
  
The resulting `--tag`ged image now contains a functional catalog index with only the filtered graph of operators and references from all of the specified index images.

### TODO Generate a curated registry database
This section will describe how this works for the `opm registry filter` command.


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

