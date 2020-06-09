---
title: opm-content-management 
authors:
  - "@anik120"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: yyyy-mm-dd
last-updated: yyyy-mm-dd
status: implementable
---

# OPM Content Management

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

This enhancement is a proposal for extending the existing capabilites of the `opm` command line utility, to provide more fine grained control over the management of catalog indexes.   

`opm` will significantly improve the UX around working with indexes by allowing removal of individual bundles from a package and removal of channels from a package. Indexes will now also be able to be queried directly for inspection, and update graph visualization for individual packages will be added. Bundle validation without a deamon by default is also proposed in this enhancement, keeping docker users in mind. 

## Motivation

As more and more operators are migrated to the new bundle format, and more index images are created for Operator Catalogs, creators of catalogs of operators, such as the Red Hat pipeline responsible for maintaining the `redhat-operators` catalog, should have more fine grained control over the management of the catalogs using the `opm` tooling.  

### Goals

- Allow `opm` users to perform bundle validations in environments without a Docker daemon. Bundle validation also includes validation of bundle image labels against `metadata/annotations.yaml`
- Allow `opm` users to safely edit indexes, which includes removing bundle version from a package in an index, and removing a channel from a bundle. 
- Allow `opm` users to inspect catalog content and return the update graph of Operators in machine-readable format.

### Non-Goals


## Proposal
 
### Implementation Details

### Validate the bundle image labels against bundle annotations

Currently, `opm alpha bundle validate` validates only the content of `metadata/annotations.yaml` file, but not the image labels of the DockerFile. The image labels are used when the bundle is added to an index image. As a result, if there are mismatches between the content of the bundle image (i.e `metadata/annotations.yaml`) and the bundle image labels, the bundle image is invalid and could potentially cause issues when it is added to an index image.

The `opm alpha bundle validate` command will be enhanced with an extra step that will check to see if the contents of `metadata/annotations.yaml` matches with the labels of the bundle image. If the `metadata/annotations.yaml` contain fields that are not there in the labels of the image or vice versa, the validation will fail.  

### Explain and visualize the update graph of an operator in an index

A new sub-command `graph` will be added under `opm index` to allow operator authors or cluster admins to understand the update graph for a given Operator in a catalog, so that the update graph can be reasoned about for correctness and available versions.

The `graph` sub-command will have an `out` flag, to denote the type of graph visualization, one of `text`, `png`, [`dot`](https://en.wikipedia.org/wiki/DOT_(graph_description_language)) (possibly using (go-graphviz)[https://github.com/goccy/go-graphviz]), or `json` (for a machine readable data format of the graph). The default option will be `json`. 

For example, `opm index graph --out=graphical --package=example-operator quay.io/example-namespace/example-index:v0.0.1`

### Support for deamonless bundle validation

As an operator author, I want to be able to validate a bundle image without requiring a daemon or shelling out to a local binary. 

Currently, `opm` requires the docker deamon for bundle validation (mostly for unpacking container image) by default. Bundle validation should be self-contained without requiring any external tool being installed. This dependency on a deamon will be removed by using containerd or podman as default container tools to pull and unpack container images for validating the bundle.    


### Support for "removing" an individual Operator bundle from an index

Currently, `opm` has an `index rm` option that allows for the deletion of a package entirely from an index. 

`opm` will be enhanced to provide an option to "remove" an operator from a package. Removing a bundle from a package would essentially mean to mark the bundle as unavailable in the update graph. 

Operator-registry change: 

A new field `recalled` will be added to the `operatorbundle` table. The default value of the field will be `false`

| NAME | CSV | BUNDLE | BUNDLEPATH | SKIPRANGE | VERSION | REPLACES | SKIPS | RECALLED |
|------|-----|--------|------------|-----------|---------|----------|-------|----------|


 When a bundle is marked for removal from the package, the `recalled` field will be set to `true`. When a bundle is set to `recalled:true`, it'll be treated as a node in the update graph that has no edges going into it, but can have an edge going out of it. 

When a predecessor bundle to a `recalled` bundle is requested to be upgraded, the operator is either upgraded to the successor bundle of the `recalled` bundle if it's available, or the upgrade request is rejected. If the `recalled` bundle is already installed in a cluster, the operator is upgraded automatically to the successor bundle if/when it is available.  

`opm` change:

Two flags will be introduced under the `index rm` sub-command, `package` and `bundle`.

`opm index rm --package=<package-name> quay.io/example-namespace/example-index:v0.0.1` will remove a package from an index (already implemented as `index rm`, will only be moved under the `package` sub-command)

`opm index rm --bundle=<bundle-path> quay.io/example-namespace/example-index:v0.0.1` will "remove" the bundle from a package in an index, i.e mark is as `recalled` in the index. 

### Support for removing a channel from a package in a bundle 

`opm index rm` will also have a `channel` flag, that will be paired with the `package` flag to remove a channel from a package. When a channel is removed from an index, all the bundle in the channel will be marked as `recalled`

```
$ opm index rm --channel=beta --package=example-operator quay.io/example-namespace/example-indexv-0.0.1

Removed bundles example-operatorv0.0.1, example-operatorv0.0.2 from package example-operator. 
Deleted channel beta from example-operator.

``` 
Operator-registry change: 

A new field `recalled` will be added to the `channel` table. The default value of the field will be `false`. When a channel is marked for removal, the value will be switched to `true`

NOTE: Although installation/upgrade from the channel can be prevented by marking the bundles in the channel as `recalled`, marking a channel in addition will leave scope for providing feedback to any user who's trying to upgrade from one version to another from the removed channel, but is unaware of the removal.  


### Package inspection using opm 

Currently, the procedure of inspecting an index requires several manual steps: 

1) Pull the latest index image 
2) Build and serve the registry database to expose the grpc api 
3) Query the database 

The `inspect` sub-command will be introduced under the `opm index` command to facilitate single step inspection of an index, via a machine-readable representation of all the packages present in the index. The `inspect` sub-command will automate the steps listed above by 
1) Using a container tool to pull the latest index image from a container registry. It will have a `--container` flag to indicate the user's desire to use either `docker`, `podman` or `containerd`. 
2) Build, serve and query the database to finally output a machine-readable representation of all packages(with the bundles in them) in the index. 

For example,

`opm index inspect quay.io/example-namespace/example-index:v0.0.1` 

```
{
    packages: 
    {
        name: "example-operator1",
        channels: {
            name: "alpha",
            bundles: {
                {
                    name: "example-operator1v0.0.1
                },
                {
                    name: "example-operator1v0.0.2
                },
            },
        },
    },
    {
        name: "example-operator2",
        channels: {
            name: "beta",
            bundles: {
                {
                    name: "example-operator2v1.0.0
                },
                {
                    name: "example-operator2v1.0.1
                },
            },
        },
    } 
    
}
```

### Test Plan

Unit and e2e tests will be added to the code base as proof of concepts for above mentioned additions. 


