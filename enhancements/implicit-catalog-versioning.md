---
title: implicit-catalog-versioning
authors:
  - "@kevinrizza"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2019-12-02
last-updated: 2019-12-02
status: implementable
---

# implicit-catalog-version

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

The purpose of this enhancement is to modify the definition and creation of Operator Catalog update graphs for individual operators. It outlines a proposal to replace the explicit catalog definition schema (based on fields in each ClusterServiceVersion that relate to names of other ClusterServiceVersions) with an implicit update graph based on [Semantic Versioning](https://semver.org/#semantic-versioning-200).

## Motivation

With the move to Operator Bundle and Operator Index images to define the Catalogs that define the metadata available for OLM to install operators on clusters, we need to rethink the way that the metadata is managed and published. The previous expectation was that the entire historical set of manifests for a given operator were stored in a source controlled directory and used to generate a catalog (via the [operator-registry project](https://github.com/operator-framework/operator-registry)), for example [these manifests would be used](https://github.com/operator-framework/community-operators/tree/master/upstream-community-operators/etcd) to generate a catalog with etcd included:

```
manifests
└── etcd
    ├── 0.6.1
    │   ├── etcdcluster.crd.yaml
    │   └── etcdoperator.clusterserviceversion.yaml
    ├── 0.9.0
    │   ├── etcdbackup.crd.yaml
    │   ├── etcdcluster.crd.yaml
    │   ├── etcdoperator.v0.9.0.clusterserviceversion.yaml
    │   └── etcdrestore.crd.yaml
    ├── 0.9.2
    │   ├── etcdbackup.crd.yaml
    │   ├── etcdcluster.crd.yaml
    │   ├── etcdoperator.v0.9.2.clusterserviceversion.yaml
    │   └── etcdrestore.crd.yaml
    └── etcd.package.yaml
```

In that format, it is fairly straightforward to define an explicit update graph -- all of the information about the graph history is defined in the same directory and can be republished. But when each version (an Operator Bundle) is published separately, then the expected format of the source control repository becomes just the latest manifests:

```
etcd
├── manifests
│   ├── etcdbackup.crd.yaml
│   ├── etcdcluster.crd.yaml
│   ├── etcdoperator.clusterserviceversion.yaml
│   └── etcdrestore.crd.yaml
└── metadata
    └── annotations.yaml
```

In this format, it becomes significantly less clear how to define your update graph. If you are only ever publishing a specific version of your operator, how do you know how it fits into the graph? What has even been published previously? Where do you even get the metadata needed to define your update graph? The author of this operator for this trivial example would have to either piece together their previous releases based on a hopefully accurate source control history, or they would need to use some external tooling (which does not currently exist today) to inspect the already created Index image to see what has been released and define their graph that way.

The primary motivation for this enhancement is to simplify all of that complexity and guesswork by redefining the main driver for the update graph to just use semantic versioning. If the versioning is implicit (i.e. 1.3.1 is an update from 1.3.0) all the operator author needs to be aware of is "is my new version the latest thing?" -- they do not need to inspect the manifests of existing versions to construct the graph.

In the future, this proposal also helps to solve the ordering problem when building and modifying index images. Today when adding to Index images via the `opm index` command, you need to specify your images in the order that they replace previous images because of the possibility that the explicit data will make an invalid intermediate graph. With the use of semantic version, it becomes trivial to insert elements into the middle of the graph. It also allows a future in which specific versions can be deleted from graphs without a need to edit the yaml.

### Goals

* Define Catalog update graphs for individual operators based implicitly on semantic version rather than asking users to explicitly mention what version the previous version replaces

### Non-Goals

* Modify any of the features of OLM or the Operator Registry to use new information from the semantic version. This proposal simply suggests using the semantic version to define the graph
* Resolve the migration of existing subscriptions to use the new Catalog Index images
* Make any explicit changes to existing OLM API versions (CSV will remain `v1alpha1`)
* Make any explicit feature changes around how OLM manages skipping updates (skipRange) and how it handles patch versions

## Proposal

Basic approach:

- Make the `spec.version` field required going forward when building registry images with `opm registry` commands (note that this requirement is only in the registry and NOT in OLM's CSV type definition)
- Modify the operator registry to build dependency graph based simply on `spec.version` field rather than using a combination of channel heads and `spec.replaces` for each graph
- Add a required `--mode` flag to `opm index` and `opm bundle` commands that can determine how fine grained the update is inserted into each channel's graph

### Ignore the `replaces` field when building the registry graph, instead use `version`:

Currently, the update graph defined when creating an operator catalog is explicit. Essentially, the ClusterServiceVersion (CSV) file has an explicit `replaces` field:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
metadata:
  name: etcdoperator.v0.9.2
  namespace: placeholder
spec:
    displayName: etcd
    description: Etcd Operator
    version: 0.9.2
    replaces: etcdoperator.v0.9.0
```

The `replaces` field is the source for the update graph for each install channel in a given operator. When the `operator-registry` builds the update graph, it actually walks through each CSV in a given Operator and, for each channel, constructs a chain of CSVs that define the update path. Instead, we can just define our spec without the replaces field:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
metadata:
  name: etcdoperator.v0.9.2
  namespace: placeholder
spec:
    displayName: etcd
    description: Etcd Operator
    version: 0.9.2
```

We would use the fact that 0.9.2 > 0.9.0 to implicitly assume that 0.9.0 is replaced by 0.9.2 through the rules of semantic version.

To accomplish this:
- Make `spec.version` a required field in the operator-registry project
- Update the `operator-registry` to walk through the set of CSVs and construct the replacement chain by reading the `version` field instead of `replaces`. This will be done by modifying the Channel tables to point to next ordered semantic versions when defining the graph
- Ignore `skips` field entirely when building the graph
- Update OLM to use the query for the replacement rather than the current value in the existing CSV

Note that these requirements will only be the case when generating update graphs with the latest version of the operator-registry's `opm registry` commands. 

### Mode

First, a quick note about `skipRange`: The `skipRange` annotation was originally added to allow an explicit definition of version ranges that can be skipped over -- generally, this was done in order to allow patch versions of released operators to upgrade directly. However, given that version is becoming a first class citizen, some users may expect that patch version to work in that fashion in general.

In order to account for this possibility, we will explicitly ask the user when adding bundles which method they want to use to upgrade. To handle this, `opm index` and `opm bundle` will add a new flag `--mode` that enumerates to two different options:

`opm index add quay.io/namespace/my-operator --mode=semver`

In this case, we will use the semantic version to define the graph. Simple comparison drives the list, so 1.1.0 is replaced by 1.1.1 is replaced by 1.1.2 is replaced by 1.2.0. ex:

`opm index add quay.io/namespace/my-operator --mode=semver-skippatch`

In this case, we will use the semantic version to define the graph while skipping directly to the latest patch version. Here 1.1.0 is replaced by 1.1.1 until 1.1.2 is added. At that point, 1.1.2 skips directly over 1.1.1 when upgrading 1.1.0. But 1.1.2 is replaced by 1.2.0.

## Migration and Backwards Compatibility

For now, these graph changes are enclosed in the operator-registry project and simply modify what is returned when OLM asks for a replacement in a given channel. In this way, both OLM and pipelines will continue to function as they previously did as long as they do not adopt the new `opm` tool to build their registries.

### Of note
There was some significant discussion around bumping the CSV API version to `v1alpha2` and actually making these API changes mandatory. Among those things discussed:

Simply making the change outright given that the CSV API is still in an `alpha` version. We chose not to go this route given there are already many production workloads in the wild using this API.

Attempting to only make this requirement exist in the case where new catalog images built from `opm` are present, in which case anything could be migrated by looking up the relevant subscription during the migration to `v1alpha2` by querying the API for the new catalog image. For CSVs not tied to the subscription, just remove the `replaces` field and choose a sane default for the version. We chose not to go this route because there are simply too many moving pieces and it significantly increased the scope of this enhancement and tied it too closely with a similar migration effort around replacing app-registry support with native container images.

### Tooling

In order to actually reason about and interact with existing indexes, we need a convenient method of inspecting them. Currently, the only way to determine what is actually *in* the index is to fetch the image, run it, and query the served grpc API. Even then, you need a somewhat involved understanding of the replacement chain abstraction to follow the graph down for a given channel. Instead, we will create a new `opm` command `inspect` that abstracts the need for someone to serve the database to determine what packages and versions are in the registry. For example:

`opm index inspect packages`

```json
{
  "packages": [
    {
      "name": "etcd",
      "channels": ["stable", "latest"],
      "defaultChannel": "stable"
    },
    {
      "name": "prometheus",
      "channels": ["stable"],
      "defaultChannel": "stable"
    }
  ]
}
```

Will return a list of packages included in the index.

`opm index inspect package etcd`

```json
{
    "name": "etcd",
    "defaultChannel": "stable",
    "channels": [
        {
            "name": "alpha",
            "bundles": [
                {
                    "version": "0.6.1",
                    "csv": "etcdoperator.v0.6.1",
                    "bundlePath": "quay.io/etcd/etcd-operator-bundle@sha256:abcdef",
                    "replaces": null,
                    "replacements": [
                        {
                            "version": "0.9.0",
                            "csv": "etcdoperator.v0.9.0"
                        },
                        {
                            "version": "0.9.2",
                            "csv": "etcdoperator.v0.9.2"
                        }
                    ]
                },
                {
                    "version": "0.9.0",
                    "csv": "etcdoperator.v0.9.0",
                    "bundlePath": "quay.io/etcd/etcd-operator-bundle@sha256:defghi",
                    "replaces": [
                        {
                            "version": "0.6.1",
                            "csv": "etcdoperator.v0.6.1"
                        }
                    ],
                    "replacements": [
                        {
                            "version": "0.9.2",
                            "csv": "etcdoperator.v0.9.2"
                        }
                    ]
                },
                {
                    "version": "0.9.2",
                    "csv": "etcdoperator.v0.9.2",
                    "bundlePath": "quay.io/etcd/etcd-operator-bundle@sha256:ghijkl",
                    "replaces": [
                    {
                        "version": "0.6.1",
                        "csv": "etcdoperator.v0.6.1"
                    },
                    {
                        "version": "0.9.0",
                        "csv": "etcdoperator.v0.9.0"
                    }
                    ],
                    "replacements": null
                }
            ]
        },
        {
            "name": "stable",
            "bundles": [
                {
                    "version": "0.9.0",
                    "csv": "etcdoperator.v0.9.0",
                    "bundlePath": "quay.io/etcd/etcd-operator-bundle@sha256:defghi",
                    "replaces": null,
                    "replacements": [
                        {
                            "version": "0.9.2",
                            "csv": "etcdoperator.v0.9.2"
                        }
                    ]
                },
                {
                    "version": "0.9.2",
                    "csv": "etcdoperator.v0.9.2",
                    "bundlePath": "quay.io/etcd/etcd-operator-bundle@sha256:ghijkl",
                    "replaces": [
                        {
                            "version": "0.9.0",
                            "csv": "etcdoperator.v0.9.0"
                        }
                    ],
                    "replacements": null
                }
            ]
        }
    ]
}
```

Will return a list of bundles along with the bundle image they were built from and lists of (zero or more) bundles that they replace and are replaced by. One thing to note about this result is that, in it's current state `version` could still be null at this time. In the future, if CSV-less bundles become a common standard that requirement may be updated where CSV becomes an optional field and version becomes required.

Additionally, now that semantic version is used to insert bundles anywhere in the graph, we can also delete individual bundles in any part of the graph as long as we are in `--semver` or `--semver-skippatch` mode. We will create a new command `opm index rmb`:

`opm index rmb quay.io/etcd/etcd-operator-bundle@sha256:defghi --mode=semver`

This command will delete the `quay.io/etcd/etcd-operator-bundle@sha256:defghi` bundle from the registry. In our case, that bundle refers to etcd 0.9.0, so we would use the semver mode rules to synthetically stitch other versions of etcd togeother:

`opm index inspect package etcd`

```json
{
    "name": "etcd",
    "defaultChannel": "stable",
    "channels": [
        {
            "name": "alpha",
            "bundles": [
                {
                    "version": "0.6.1",
                    "csv": "etcdoperator.v0.6.1",
                    "bundlePath": "quay.io/etcd/etcd-operator-bundle@sha256:abcdef",
                    "replaces": null,
                    "replacements": [
                        {
                            "version": "0.9.2",
                            "csv": "etcdoperator.v0.9.2"
                        }
                    ]
                },
                {
                    "version": "0.9.2",
                    "csv": "etcdoperator.v0.9.2",
                    "bundlePath": "quay.io/etcd/etcd-operator-bundle@sha256:ghijkl",
                    "replaces": [
                        {
                            "version": "0.6.1",
                            "csv": "etcdoperator.v0.6.1"
                        }
                    ],
                    "replacements": null
                }
            ]
        },
        {
            "name": "stable",
            "bundles": [
                {
                    "version": "0.9.2",
                    "csv": "etcdoperator.v0.9.2",
                    "bundlePath": "quay.io/etcd/etcd-operator-bundle@sha256:ghijkl",
                    "replaces": null,
                    "replacements": null
                }
            ]
        }
    ]
}
```