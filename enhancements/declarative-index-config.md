---
title: declarative-index-config
authors:
  - "@anik120"
reviewers:
  - @njhale
  - @benluddy
  - @dmesser
  - @exdx
  - @Jamstah
  - @bparees
  - @ecordell
approvers:
  - @dmesser
  - @bparees
  - @ecordell
  - @krizza
creation-date: 2021-02-05
last-updated: 2021-02-05
status: implementable  
---

# Package representation and management in an index 

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary 

This enhancement proposes the representation of packages in an index in a standard declarative way(using json/yaml), so that index authors and operator authors can communicate with each other about the content of an index using json/yaml files, without needing to communicate with each other using containers(index images, operator bundle images etc). Using the declarative representation of the packages in the index, it also proposes allowing index authors and individual package owners to alter the properties of the packages(package name, channel information etc) as well as the structure of the packages (upgrade graphs of channels etc) of the index without having to rebuild bundles+republish the index. Finally, it discusses using the representations as source of truth for serving data from the index over a gRPC api, instead of the sqlite database that is used currently, along with migration strategy for existing indexes from the sqlite databases to the package representations.

## Motivation

In the current state of the world, operators are introduced to operator-lifecycle-manager(OLM) by representing them as [Operator Bundles](https://github.com/operator-framework/operator-registry/blob/master/docs/design/operator-bundle.md#operator-bundle). These operator bundles are then loaded onto a container using the [opm](https://github.com/operator-framework/operator-registry/blob/master/docs/design/opm-tooling.md) tool (`opm index add --bundles`) to build an index of operators. Operator bundles conceptually belong to a channel in a package, with each bundle having the ability to upgrade to a different bundle in the channel. 

![Alt text](assets/community-operators.png?raw=true "community-operators")

The way we describe the intent of a bundle to be on a particular channel in a package is by hard-coding that information in the [bundle annotations](https://github.com/operator-framework/operator-registry/blob/release-4.6/docs/design/operator-bundle.md#bundle-annotations) inside the bundle metadata itself. However, the channel that the bundle wants to be in is more of a package level information. Hard-coded channel information inside a bundle restricts a bundle in a particular channel once the bundle is built and loaded into an index. This makes it difficult to change channel info retroactively. When a bundle is added to an index using `opm`, the `mode` (one of `replaces`|`semver`|`semver-skippatch`) in which the bundle is added (`opm index add --bundles <list-of-bundles> --mode <replaces|semver|semver-skippatch>`) determines the upgrade graph of bundles in the channel (which bundle can be upgrade to which). This upgrade graph for bundles in a channel can also not be changed retroactively. The only way to change the channel info, the upgrade graph or any package level information currently is to rebuild the bundles with the modified package information and then republish the index. This is done by making additive changes to the index using the`opm` tool. 

Furthermore, a lot of the rules about the structure of an index are implicitly documented in various sources that are scattered. A best effort is being made to keep the rules updated in the form of a library in the [api repo](https://github.com/operator-framework/api/tree/master/pkg/validation) that is imported by tools like `opm`, `operator-courier` etc that perform index validation. However, like all tools that aspire to provide data validation, these tools, or api libraries that these tools import the rules from, have to be constantly maintained to keep up with the latest rule changes, while also maintaining older rules in previous releases. This introduces a lot of overhead in ensuring that olm is easy to use by users, while also leaving windows for lapses due to mis-coordination between different tools across different versions.

Since indexes are manipulated by making additive changes to it, the way an index can be reproduced today is by managing a knowledge base of the series of changes made to the index (i.e storing the `opm` commands chronologically) along with a set of intermediary container images of the index with their corresponding digest etc. This introduces a lot of overhead in maintaining indexes, and as a by product has significantly burdened the `opm` tool with feature requests to add an increasing number of sub-commands to allow index authors to manipulate their indexes in various ways. 

## Goals

A human consumable representation of packages in an index in the form of json/yaml will: 

1. Allow for editing of package level information without rebuilding and republishing artifacts(remove bundle/s from a channel, switch bundle/s to a different channel, edit the upgrade graph in a channel etc).
2. Allow modification of upgrade graph for a package without needing to rebuild and republish the index the package belongs to(i.e combine packages from two indexes, create multiple indexes with packages from a single index, remove bundles from the index, switch channels bundles belong to etc).
3. Allow for validation of the structure of an index against a standard definition of an index.
4. Provide a way for index authors to reason about their indexes via a visual representation of the index, without having to curl a grpc api that in turn queries a sqllite database underneath, etc.
5. Provide a way for tools like `opm`, `operator-registry` etc to consume the content of an index without needing a sqlite database as an input whenever they need information about the index to perform a task. 
6. Allow for reproducibility of an index using just the yaml/json representation of the index. 

## Non-goals 

1. Enumerate/implement tooling to allow different operations to be performed on the package representations. Numerous tooling(eg those that allow operation on files) that already exists can be leveraged to perform various operations on package representations. 
2. Advanced tooling to verify dependency satisfiability for individual bundles, channel upgrade graph validity for each channel in a package etc. This will be considered in a separate enhancement.
3. Define an exhaustive or authoritative set of bundle properties. This proposal will document only the properties necessary to maintain backwards-compatibility with the existing GRPC API.

## User Stories 

### Story 1

As a user with pull permission from the namespace an index is hosted in/as a component of OLM, I can query an index for a json/yaml representation of the packages of an index.
The representation of individual packages will allow individual package owners to reason about/make changes to their individual packages in isolation.  

`opm index inspect --image=docker.io/my-namespace/community-operators --output=json`

pulls the index image from the registry, creates a new folder `community-operators` and copies the package representations for the packages that are in the image to the local folder. 

```bash
$ cd community-operators
$ tree
.
├── etcd.json
├── amqstreams.json

$ cat etcd.json
{
    "schema": "olm.package",
    "name": "etcd",
    "defaultChannel": "singlenamespace-alpha",
    "icon": {
        "base64data":"iVBORw0KGgoAAAANSUhEUgAAA.....",
        "mediatype":"image/png"
    },
    "description": "A message about etcd operator, a description of channels"
}
{
    "schema": "olm.bundle",
    "name": "etcdoperator-community.v0.6.1",
    "package": "etcd",
    "image": "quay.io/operatorhubio/etcd:v0.6.1",
    "version": "0.6.1",
    "properties":[
        {
            "type": "olm.package.provided",
            "value": {
                "packageName": "etcd",
                "version": "0.6.1"
            }
        },
        {
            "type":"olm.gvk.provided",
            "value": {
                "group": "etcd.database.coreos.com",
                "kind": "EtcdCluster",
                "version": "v1beta2"
            }
        },
        {
            "type": "olm.channel",
            "value": {
                "name": "alpha"
            }
        }
    ],
    "relatedImages": [
        {
            "name": "etcdv0.6.1",
            "image": "quay.io/coreos/etcd-operator@sha256:bd944a211eaf8f31da5e6d69e8541e7cada8f16a9f7a5a570b22478997819943"
        }
    ]
}
{
    "schema": "olm.bundle",
    "name": "etcdoperator.v0.9.0",
    "package": "etcd",
    "image": "quay.io/operatorhubio/etcd:v0.9.0",
    "version": "0.9.0",
    "properties":[
        {
            "type": "olm.package.provided",
            "value": {
                "packageName": "etcd",
                "version": "0.9.0"
            }
        },
        {
            "type": "olm.gvk.provided",
            "value": {
                "group": "etcd.database.coreos.com",
                "kind": "EtcdBackup",
                "version": "v1beta2"
            }
        },
        {
            "type": "olm.channel",
            "value": {
                "name": "singlenamespace-alpha"
            }
        },
        {
            "type": "olm.channel",
            "value": {
                "name": "clusterwide-alpha"
            }
        }
    ],
    "relatedImages" : [
        {
            "name": "etcdv0.9.0",
            "image": "quay.io/coreos/etcd-operator@sha256:db563baa8194fcfe39d1df744ed70024b0f1f9e9b55b5923c2f3a413c44dc6b8"
        }
    ]
}
{
    "schema": "olm.bundle",
    "name": "etcdoperator.v0.9.2",
    "package": "etcd",
    "image": "quay.io/operatorhubio/etcd:v0.9.2",
    "version": "0.9.2",
    "properties":[
        {
            "type": "olm.package.provided",
            "value": {
                "packageName": "etcd",
                "version": "0.9.2"
            }
        },
        {
            "type": "olm.gvk.provided",
            "value": {
                "group": "etcd.database.coreos.com",
                "kind": "EtcdRestore",
                "version": "v1beta2"
            }
        },
        {
            "type": "olm.channel",
            "value": {
                "name": "singlenamespace-alpha",
                "replaces": "etcdoperator.v0.9.0"
            }
        }
    ],
    "relatedImages":[
        {
            "name":"etcdv0.9.2",
            "image": "quay.io/coreos/etcd-operator@sha256:c0301e4686c3ed4206e370b42de5a3bd2229b9fb4906cf85f3f30650424abec2"
        }
    ]
}
{
    "schema": "olm.bundle",
    "name": "etcdoperator.v0.9.2-clusterwide",
    "package": "etcd",
    "image": "quay.io/operatorhubio/etcd:v0.9.2-clusterwide",
    "version": "0.9.2-clusterwide",
    "properties":[
        {
            "type": "olm.package.provided",
            "value": {
                "packageName": "etcd",
                "version": "0.9.2-clusterwide"
            }
        },
        {
            "type": "olm.gvk.provided",
            "value": {
                "group": "etcd.database.coreos.com",
                "kind": "EtcdBackup",
                "version": "v1beta2"
            }
        },
        {
            "type": "olm.skipRange",
            "value": ">=0.9.0 <0.9.2-0"
        },
        {
            "type": "olm.skips",
            "value" : "etcdoperator.v0.6.0"
        },
        {
            "type": "olm.skips",
            "value" : "etcdoperator.v0.6.1"
        },
        {
            "type": "olm.channel",
            "value": {
                "name": "clusterwide-alpha",
                "replaces": "etcdoperator.v0.9.0"
            }
        }
    ],
    "relatedImages":[
        {
            "name":"etcdv0.9.2",
            "image":"quay.io/coreos/etcd-operator@sha256:c0301e4686c3ed4206e370b42de5a3bd2229b9fb4906cf85f3f30650424abec2"
        }
    ]
}
{
    "schema": "olm.bundle",
    "name" : "etcdoperator.v0.9.4",
    "package": "etcd",
    "image": "quay.io/operatorhubio/etcd:v0.9.4",
    "version": "0.9.4",
    "properties":[
        {
            "type": "olm.package.provided",
            "value": {
                "packageName": "etcd",
                "version": "0.9.4"
            }
        },
        {
            "type": "olm.package.required",
            "value": {
                "packageName": "test",
                "versionRange": ">=1.2.3 <2.0.0-0"
            }
        },
        {
            "type": "olm.gvk.provided",
            "value": {
                "group": "etcd.database.coreos.com",
                "kind": "EtcdBackup",
                "version": "v1beta2"
            }
        },
        {
            "type": "olm.gvk.required",
            "value": {
                "group": "testapi.coreos.com",
                "kind": "Testapi",
                "version": "v1"
            }
        },
        {
            "type": "olm.channel",
            "value": {
                "name": "singlenamespace-alpha",
                "replaces": "etcdoperator.v0.9.2"
            }
        }
    ],
    "relatedImages":[
        {
            "name":"etcdv0.9.2",
            "image": "quay.io/coreos/etcd-operator@sha256:66a37fd61a06a43969854ee6d3e21087a98b93838e284a6086b13917f96b0d9b"
        }
    ]
}
{
    "schema": "olm.bundle",
    "name": "etcdoperator.v0.9.4-clusterwide",
    "package": "etcd",
    "image": "quay.io/operatorhubio/etcd:v0.9.4-clusterwide",
    "version": "0.9.4-clusterwide",
    "properties":[
        {
            "type": "olm.package.provided",
            "value": {
                "packageName": "etcd",
                "version": "0.9.4-clusterwide"
            }
        },
        {
            "type": "olm.gvk.provided",
            "value": {
                "group": "etcd.database.coreos.com",
                "kind": "EtcdBackup",
                "version": "v1beta2"
            }
        },
        {
            "type": "olm.channel",
            "value": {
                "name": "clusterwide-alpha",
                "replaces": "etcdoperator.v0.9.2-clusterwide"
            }
        }
    ],
    "relatedImages":[
        {
            "name":"etcdv0.9.2",
            "image": "quay.io/coreos/etcd-operator@sha256:66a37fd61a06a43969854ee6d3e21087a98b93838e284a6086b13917f96b0d9b"
        }
    ]
}
```

### Story 2

As an index author(with push permission to the namespace my index is hosted in), I can edit the packages in my index (remove bundles from a package, move a bundle to a different channel, etc), using only the json representation of the index/packages inside the index. 

```bash
$ cd community-operators 
$ //edit etcd.json to add etcdoperator.v0.9.0 to a the channel `alpha`, and add an upgrade edge from etcdoperator-community.v0.6.1 to etcdoperator.v0.9.0 in channel `alpha`
$ cd ..
$ opm index update community-operators --tag=docker.io/my-namespace/community-operators:latest
$ rm -rf community-operators
$ opm index inspect --image=docker.io/my-namespace/community-operators --output=json
$ cat community-operators/etcd.json
{
    "schema": "olm.package",
    "name": "etcd",
    "defaultChannel": "singlenamespace-alpha",
    "icon": {
        "base64data":"iVBORw0KGgoAAAANSUhEUgAAA.....",
        "mediatype":"image/png"
    },
    "description": "A message about etcd operator, a description of channels"
}
{
    "schema": "olm.bundle",
    "name": "etcdoperator-community.v0.6.1",
    "package": "etcd",
    "image": "quay.io/operatorhubio/etcd:v0.6.1",
    "version": "0.6.1",
    "properties":[
        {
            "type": "olm.package.provided",
            "value": {
                "packageName": "etcd",
                "version": "0.6.1"
            }
        },
        {
            "type": "olm.gvk.provided",
            "value": {
                "group": "etcd.database.coreos.com",
                "kind": "EtcdCluster",
                "version": "v1beta2"
            }
        },
        {
            "type": "olm.channel",
            "value": {
                "name": "alpha"
            }
        }
    ],
    "relatedImages": [
        {
            "name": "etcdv0.6.1",
            "image": "quay.io/coreos/etcd-operator@sha256:bd944a211eaf8f31da5e6d69e8541e7cada8f16a9f7a5a570b22478997819943"
        }
    ]
}
{
    "schema": "olm.bundle",
    "name": "etcdoperator.v0.9.0",
    "package": "etcd",
    "image": "quay.io/operatorhubio/etcd:v0.9.0",
    "version": "0.9.0",
    "properties":[
        {
            "type": "olm.package.provided",
            "value": {
                "packageName": "etcd",
                "version": "0.9.0"
            }
        },
        {
            "type": "olm.gvk.provided",
            "value": {
                "group": "etcd.database.coreos.com",
                "kind": "EtcdBackup",
                "version": "v1beta2"
            }
        },
        {
            "type": "olm.channel",
            "value": {
                "name": "alpha",
                "replaces":"etcdoperator-community.v0.6.1"
            }
        },
        {
            "type": "olm.channel",
            "value": {
                "name": "singlenamespace-alpha"
            }
        },
        {
            "type": "olm.channel",
            "value": {
                "name": "clusterwide-alpha"
            }
        }
    ],
    "relatedImages" : [
        {
            "name": "etcdv0.9.0",
            "image": "quay.io/coreos/etcd-operator@sha256:db563baa8194fcfe39d1df744ed70024b0f1f9e9b55b5923c2f3a413c44dc6b8"
        }
    ]
}
.
.
.
.
```

Similarly, removing a `.json` from `community-operators` and pushing that update up to the container image removes the operator from the index. 

### Story 3

As a user with pull permission from the namespace an index is hosted in/as a component of OLM, I can validate the structure of the content of an index using the json representation.
 
```bash
$ cd community-operators 
$ sed '/defaultChannel/d' etcd.json > etcd.json //delete the line "defaultChannel":"singlenamespace-alpha"
$ git status 
	modified:    etcd.json

$ cd ..
$ opm index validate community-operators
marshal error: etcd.defaultchannel: cannot convert incomplete value "string" to JSON
$ opm index update community-operator --tag=docker.io/mynamespace/community-operators:latest
$ opm index validate docker.io/mynamespace/community-operators:latest 
marshal error: etcd.defaultchannel: cannot convert incomplete value "string" to JSON
$ cd community-operators 
$ git checkout etcd.json
$ cd ..
$ opm index validate community-operators 
No errors found!
```

### Story 4 

Given a set of package representations, I can author a new index using just those representations.

```bash
$ tree community-operators 
community-operators
├── amqstreams.json
└── etcd.json
└── servicemesh.json
$ rm community-operators/servicemesh.json
$ opm index create --from=community-operators --tag=docker.io/someothernamespace/new-community-operators:latest
```
## Implementation Details
### Representing a package using json/yaml 

The representation of the content of an index using json/yaml provides a unique opportunity to rethink the structure of an index conceptually. 

![Alt text](assets/new-community-operators.png?raw=true "community-operators")

Since each bundle in the index belongs to a channel in a package, the index itself can be represented as a collection of packages. Each package is then represented as a json/yaml, a config file that will live inside the index container image alongside the individual bundle metadata.

#### Contents of the config file

The config file for each package will have a stream of json objects, representing the package and bundle information for that package. Currently, this information is stored in a sql database inside the index. To capture all the information previously stored in this database but in a normalized way, the config file will have two kinds of json blob: 

##### Declarative Package Format
```json
{
    "schema": "olm.package",
    "name": "<package-name>",
    "defaultChannel": "<channel-name>",
    "icon": "embedded or remote",
    "description": "A message about the operator, a description of channels..."
}
```
This information is currently captured in the `package` table.

##### Declarative Bundle Format
```json
{
    "schema": "olm.bundle",
    "name": "<operatorbundle_name>",
    "package": "<package_name>",
    "image": "<operatorbundle_path>",
    "version": "<version>",
    "properties":["<list of properties of bundle that encode bundle dependencies(provided and required apis) upgrade graph info(skips/skipRange), and bundle channel/s info>"],
    "relatedImages" : ["<list-of-related-images>"]
}
```

This information is currently captured in the `operatorbundle`, `properties`, `channel`, `channel_entry`, `api`, `api_provider`, `api_requirer` and `related_image` tables in the sql database.

##### Extension formats

While this enhancement proposal does not specify any schemas other than `olm.package` and `olm.bundle`, it does not preclude them either. When reading/writing blobs, `opm` will search for a minimal set of fields to determine validity and associations:

```json
{
  "schema": "<string>", /* required: "olm." prefix is reserved; blobs without schemas will be ignored. */
  "package": "<string>", /* optional: blobs that define a their associated package will be stored in `<packageName>.json`, others will be stored in `__global.json` */
  "properties": [
    {
      "type": "<string>", /* required: "olm." prefix is reserved */
      "value": <arbitrary_json_defined_by_type> /* required */
    }
  ] /* optional: list of 0..n properties; not every schema is required to have properties, but if properties are present, they must follow the property spec */
}
```

###### A note on properties

Defining bundle properties is not within the scope of this proposal. For the purposes of the declarative configuration format, properties are opaque JSON blobs that can convey arbitrary metadata about a particular bundle.

However, for backwards compatibility reasons, a subset of well-defined properties will be understood by `opm` so that declarative config representations can be transparently swapped in for the deprecated database backend while still serving the same GRPC API.
These properties are:

| Type                   | Value Schema                           | Example                                                                                                                   |
|------------------------|----------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| `olm.package.provided` | `{ packageName, version string }`      | `{"type":"olm.package.provided", "value": {"packageName":"etcd", "version":"0.9.4"}}`                                     |
| `olm.package.required` | `{ packageName, versionRange string }` | `{"type":"olm.package.required", "value": {"packageName":"test", "versionRange":">=1.2.3 <2.0.0-0"}}`                     |
| `olm.gvk.provided`     | `{ group, version, kind string }`      | `{"type":"olm.gvk.provided", "value": {"group": "etcd.database.coreos.com", "version": "v1beta2", "kind": "EtcdBackup"}}` |
| `olm.gvk.required`     | `{ group, version, kind string }`      | `{"type":"olm.gvk.required", "value": {"group": "test.coreos.com", "version": "v1", "kind": "Testapi"}}`                  |
| `olm.skips`            | `string`                               | `{"type":"olm.skips", "value": "etcdoperator.v0.9.0"}`<br>`{"type":"olm.skips", "value": "etcdoperator.v0.9.2"}`          |
| `olm.skipRange`        | `string`                               | `{"type":"olm.skipRange", "value": "<0.9.4"}`                                                                             |
| `olm.channel`          | `{ name, replaces string }`            | `{"type":"olm.channel", "value":{"name":"singlenamespace-alpha", "replaces":"etcdoperator.v0.9.2"}}`<br>`{"type":"olm.channel", "value":{"name":"clusterwide-alpha"}}`         |

To support both old and new clients, `opm` will ensure that there are matching `olm.gvk`/`olm.gvk.provided` and `olm.package`/`olm.package.provided` properties present in bundle properties when migrating from the sqlite database representation and also during build time and validation. When all supported clients have been transitioned to understand the new property names, `olm.gvk` and `olm.package` will be retired.

#### Filesystem structure

Declarative configs are expected to exist in a directory structure. When reading a declarative config index, `opm` will
traverse the file tree and load valid blobs from each file it finds. When writing a declarative config, package-specific
blobs are concatenated into a single `<packageName>.json` file, and non-package-specific blobs are concatenated into
`__global.json`.

The output directory structure will be:
```bash
$ tree <index_name>
<index_name>
├── amqstreams.json
├── etcd.json
├── __global.json
└── servicemesh.json
```

When stored in a tarball, the output structure will be:
```bash
$ tar tvf <index_name>.db
drwxr-xr-x nobody/nobody     0 2021-03-17 17:18 index/
-rw-r--r-- nobody/nobody     0 2021-03-17 15:44 index/servicemesh.json
-rw-r--r-- nobody/nobody     0 2021-03-17 17:18 index/__global.json
-rw-r--r-- nobody/nobody     0 2021-03-17 15:39 index/amqstreams.json
-rw-r--r-- nobody/nobody     0 2021-03-17 15:38 index/etcd.json
```

When migrating an sqlite file to a tarball, `opm` will create the tarball with the declarative directory structure
(described above) with the root directory always called `index`.

For example, if `my-index.db` is an sqlite file `opm registry add --database /path/to/my-index.db --bundle my-bundle:v0.1.0`
will replace the input `/path/to/my-index.db` file with a tarball, also called `/path/to/my-index.db`. The tarball
`my-index.db` file will have an `index` directory in its root, which contains global and package JSON files.

```bash
$ tar tvf my-index.db
drwxr-xr-x nobody/nobody     0 2021-03-17 17:18 index/
-rw-r--r-- nobody/nobody     0 2021-03-17 15:44 index/servicemesh.json
-rw-r--r-- nobody/nobody     0 2021-03-17 17:18 index/__global.json
-rw-r--r-- nobody/nobody     0 2021-03-17 15:39 index/amqstreams.json
-rw-r--r-- nobody/nobody     0 2021-03-17 15:38 index/etcd.json
```

#### Representing the upgrade graph in the channel json blob

Currently, a bundle can be added into the index using `opm index add --bundles <list-of-bundle-paths> --mode replaces|semver|semver-skippatch --tag=<index-image-tag>` where the bundle images (like `quay.io/operatorhubio/etcd:v0.9.0`) are included in `<list-of-bundle-path>`. The bundle can also mention bundles it can be upgrade from using the `skips` or `skipRange` fields in the bundle ClusterServiceVersion. With all of these information provided, the upgrade graph of the bundles in the package is calculated and stored in the `channel_entry` table of the sql database that is built inside the index, while the `skips`/`skipRange` information is persisted in the `operatorbundle` table. The `channel_entry` table is always authoritative in terms of calculating the upgrade graph in a package. The operatorbundle table persists the values of `skips`/`skipRange`/`replaces` when the bundle was first unpacked/added to the index, and other index operations could have changed the real graph in `channel_entry`.  

```bash
sqlite> .schema channel_entry
CREATE TABLE channel_entry (
			entry_id INTEGER PRIMARY KEY,
			channel_name TEXT,
			package_name TEXT,
			operatorbundle_name TEXT,
			replaces INTEGER,
			depth INTEGER,
			FOREIGN KEY(replaces) REFERENCES channel_entry(entry_id) DEFERRABLE INITIALLY DEFERRED, 
			FOREIGN KEY(channel_name, package_name) REFERENCES channel(name, package_name) ON DELETE CASCADE
		);
sqlite> select * from channel_entry where package_name="etcd";

entry_id  channel_name           package_name  operatorbundle_name              replaces  depth
--------  ---------------------  ------------  -------------------------------  --------  -----
1818      alpha                  etcd          etcdoperator-community.v0.6.1              0    
1819      clusterwide-alpha      etcd          etcdoperator.v0.9.4-clusterwide  1820      0    
1820      clusterwide-alpha      etcd          etcdoperator.v0.9.2-clusterwide  1821      1    
1821      clusterwide-alpha      etcd          etcdoperator.v0.9.0                        2    
1822      singlenamespace-alpha  etcd          etcdoperator.v0.9.4              1823      0    
1823      singlenamespace-alpha  etcd          etcdoperator.v0.9.2              1824      1    
1824      singlenamespace-alpha  etcd          etcdoperator.v0.9.0                        2 

.schema operatorbundle
CREATE TABLE operatorbundle (
			name TEXT PRIMARY KEY,
			csv TEXT,
			bundle TEXT,
			bundlepath TEXT, skiprange TEXT, version TEXT, replaces TEXT, skips TEXT);

sqlite> select name,bundlepath,version,skiprange,skips,replaces from operatorbundle where name like 'etcd%';

name                             bundlepath                                     version            skiprange  skips  replaces                       
-------------------------------  ---------------------------------------------  -----------------  ---------  -----  -------------------------------
etcdoperator.v0.9.0              quay.io/operatorhubio/etcd:v0.9.0              0.9.0                                                               
etcdoperator-community.v0.6.1    quay.io/operatorhubio/etcd:v0.6.1              0.6.1                                                               
etcdoperator.v0.9.2-clusterwide  quay.io/operatorhubio/etcd:v0.9.2-clusterwide  0.9.2-clusterwide                    etcdoperator.v0.9.0            
etcdoperator.v0.9.2              quay.io/operatorhubio/etcd:v0.9.2              0.9.2                                etcdoperator.v0.9.0            
etcdoperator.v0.9.4-clusterwide  quay.io/operatorhubio/etcd:v0.9.4-clusterwide  0.9.4-clusterwide                    etcdoperator.v0.9.2-clusterwide
etcdoperator.v0.9.4              quay.io/operatorhubio/etcd:v0.9.4              0.9.4                                etcdoperator.v0.9.2   
```

This information will be captured in the `properties.<"olm.channel">.value.replaces`, `properties.<"olm.skips">.value` and `properties.<"olm.skipRange">.value` of the `olm.bundle` blob.

When building an internal model of the upgrade graph for a particular channel, the `properties.<"olm.channel">.value.replaces` and `properties.<"olm.skips">.value` fields are used to count the number of incoming edges that each bundle has.
To be valid, a channel's bundles must form a directed acyclic graph, and there must be exactly one bundle with zero incoming edges.

The `properties.<"olm.skipRange">` property is currently consumed only by the OLM resolver to allow bundles to "override" the graph and form new implicit upgrade edges to allow direct upgrades from other bundles whose versions fall within the `skipRange`.

#### Creating the json/yaml config file to represent a package

The `opm index add` command will be enhanced to create a config file for representing the package the bundle is being added to if it is the first bundle in the package that is being added to this index. If the package the bundle is being added to already exists, the `olm.bundle` blob for the bundle will be added to the existing package config file.

```bash
$ opm index inspect --image=docker.io/my-namespace/community-operators:latest --output=json
$ tree community-operators
.
├── etcd.json

$ cat community-operators/etcd.json
{
    "schema": "olm.package",
    "name": "etcd",
    "defaultChannel": "singlenamespace-alpha",
    "icon": {
        "base64data":"iVBORw0KGgoAAAANSUhEUgAAA.....",
        "mediatype":"image/png"
    },
    "description": "A message about etcd operator, a description of channels"
}
{
    "schema": "olm.bundle",
    "name": "etcdoperator-community.v0.6.1",
    "package": "etcd",
    "image": "quay.io/operatorhubio/etcd:v0.6.1",
    "version": "0.6.1",
    "properties":[
        {
            "type": "olm.package.provided",
            "value": {
                "packageName": "etcd",
                "version": "0.6.1"
            }
        },
        {
            "type": "olm.gvk.provided",
            "value": {
                "group": "etcd.database.coreos.com",
                "kind": "EtcdCluster",
                "version": "v1beta2"
            }
        },
        {
            "type": "olm.channel",
            "value": {
                "name": "alpha"
            }
        }
    ],
    "relatedImages": [
        {
            "name": "etcdv0.6.1",
            "image": "quay.io/coreos/etcd-operator@sha256:bd944a211eaf8f31da5e6d69e8541e7cada8f16a9f7a5a570b22478997819943"
        }
    ]
}
$ opm index add --bundles quay.io/operatorhubio/etcd:v0.9.0 --mode replaces --tag docker.io/my-namespace/community-operators:latest
$ docker push docker.io/my-namespace/community-operators:latest
$ opm index inspect --image=docker.io/my-namespace/community-operators:latest --output=json
$ cat community-operators/etcd.json
{
    "schema": "olm.package",
    "name": "etcd",
    "defaultChannel": "singlenamespace-alpha",
    "icon": {
        "base64data":"iVBORw0KGgoAAAANSUhEUgAAA.....",
        "mediatype":"image/png"
    },
    "description": "A message about etcd operator, a description of channels"
}
{
    "schema": "olm.bundle",
    "name": "etcdoperator-community.v0.6.1",
    "package": "etcd",
    "image": "quay.io/operatorhubio/etcd:v0.6.1",
    "version": "0.6.1",
    "properties":[
        {
            "type": "olm.package.provided",
            "value": {
                "packageName": "etcd",
                "version": "0.6.1"
            }
        },
        {
            "type": "olm.gvk.provided",
            "value": {
                "group": "etcd.database.coreos.com",
                "kind": "EtcdCluster",
                "version": "v1beta2"
            }
        },
        {
            "type": "olm.channel",
            "value": {
                "name": "alpha"
            }
        }
    ],
    "relatedImages": [
        {
            "name": "etcdv0.6.1",
            "image": "quay.io/coreos/etcd-operator@sha256:bd944a211eaf8f31da5e6d69e8541e7cada8f16a9f7a5a570b22478997819943"
        }
    ]
}
{
    "schema": "olm.bundle",
    "name": "etcdoperator.v0.9.0",
    "package": "etcd",
    "image": "quay.io/operatorhubio/etcd:v0.9.0",
    "version": "0.9.0",
    "properties":[
        {
            "type": "olm.package.provided",
            "value": {
                "packageName": "etcd",
                "version": "0.9.0"
            }
        },
        {
            "type": "olm.gvk.provided",
            "value": {
                "group": "etcd.database.coreos.com",
                "kind": "EtcdBackup",
                "version": "v1beta2"
            }
        },
        {
            "type": "olm.channel",
            "value": {
                "name": "singlenamespace-alpha"
            }
        }
    ],
    "relatedImages" : [
        {
            "name": "etcdv0.9.0",
            "image": "quay.io/coreos/etcd-operator@sha256:db563baa8194fcfe39d1df744ed70024b0f1f9e9b55b5923c2f3a413c44dc6b8"
        }
    ]
}
```

The `opm index add` command will also be enhanced to create `olm.bundle` json blobs for bundles passed onto the command along with a package config file, and append them to the config file.

```bash
$ cat community-operators/etcd.json
{
    "schema": "olm.package",
    "name": "etcd",
    "defaultChannel": "singlenamespace-alpha",
    "icon": {
        "base64data":"iVBORw0KGgoAAAANSUhEUgAAA.....",
        "mediatype":"image/png"
    },
    "description": "A message about etcd operator, a description of channels"
}
{
    "schema": "olm.bundle",
    "name": "etcdoperator-community.v0.6.1",
    "package": "etcd",
    "image": "quay.io/operatorhubio/etcd:v0.6.1",
    "version": "0.6.1",
    "properties":[
        {
            "type": "olm.package.provided",
            "value": {
                "packageName": "etcd",
                "version": "0.6.1"
            }
        },
        {
            "type": "olm.gvk.provided",
            "value": {
                "group": "etcd.database.coreos.com",
                "kind": "EtcdCluster",
                "version": "v1beta2"
            }
        },
        {
            "type": "olm.channel",
            "value": {
                "name": "alpha"
            }
        }
    ],
    "relatedImages": [
        {
            "name": "etcdv0.6.1",
            "image": "quay.io/coreos/etcd-operator@sha256:bd944a211eaf8f31da5e6d69e8541e7cada8f16a9f7a5a570b22478997819943"
        }
    ]
}
$ opm index add --bundles quay.io/operatorhubio/etcd:v0.9.0 --mode replaces
$ cat community-operators/etcd.json
{
    "schema": "olm.package",
    "name": "etcd",
    "defaultChannel": "singlenamespace-alpha",
    "icon": {
        "base64data":"iVBORw0KGgoAAAANSUhEUgAAA.....",
        "mediatype":"image/png"
    },
    "description": "A message about etcd operator, a description of channels"
}
{
    "schema": "olm.bundle",
    "name": "etcdoperator-community.v0.6.1",
    "package": "etcd",
    "image": "quay.io/operatorhubio/etcd:v0.6.1",
    "version": "0.6.1",
    "properties":[
        {
            "type": "olm.package.provided",
            "value": {
                "packageName": "etcd",
                "version": "0.6.1"
            }
        },
        {
            "type": "olm.gvk.provided",
            "value": {
                "group": "etcd.database.coreos.com",
                "kind": "EtcdCluster",
                "version": "v1beta2"
            }
        },
        {
            "type": "olm.channel",
            "value": {
                "name": "alpha"
            }
        }
    ],
    "relatedImages": [
        {
            "name": "etcdv0.6.1",
            "image": "quay.io/coreos/etcd-operator@sha256:bd944a211eaf8f31da5e6d69e8541e7cada8f16a9f7a5a570b22478997819943"
        }
    ]
}
{
    "schema": "olm.bundle",
    "name": "etcdoperator.v0.9.0",
    "package": "etcd",
    "image": "quay.io/operatorhubio/etcd:v0.9.0",
    "version": "0.9.0",
    "properties":[
        {
            "type": "olm.package.provided",
            "value": {
                "packageName": "etcd",
                "version": "0.9.0"
            }
        },
        {
            "type": "olm.gvk.provided",
            "value": {
                "group": "etcd.database.coreos.com",
                "kind": "EtcdBackup",
                "version": "v1beta2"
            }
        },
        {
            "type": "olm.channel",
            "value": {
                "name": "singlenamespace-alpha"
            }
        }
    ],
    "relatedImages" : [
        {
            "name": "etcdv0.9.0",
            "image": "quay.io/coreos/etcd-operator@sha256:db563baa8194fcfe39d1df744ed70024b0f1f9e9b55b5923c2f3a413c44dc6b8"
        }
    ]
}
```

### Using the package json representation

Once every package inside an index is represented using a json/yaml config file, the config files can be leveraged to inspect/update/validate the packages inside the index without needing to rebuild the index from scratch.  
#### Inspecting the packages inside an index

A new sub-command `inspect` will be introduced under `opm index`.

When `opm index inspect` is summoned, the index image will be downloaded, and the json representations of the packages inside the index will be unfurled in a folder with the same name as the index. 

```bash 
$ opm index inspect --image=docker.io/my-namespace/community-operators --output=json
$ cd community-operators
$ tree
.
├── etcd.json
├── amqstreams.json
```
#### Editing a package inside an index

A new sub-command `update` will be introduced under `opm index`, which will take a folder containing json representations of packages as it's input along with an existing index image tag, and will replace the edited config files in the index container. 

```bash
$ opm index inspect --image=quay.io/some-namespace/my-index 
$ // edit a my-index/package-config.json 
$ opm index update my-index --tag=docker.io/some-namespace/my-index:latest
``` 

> Note: For a seamless UX, the following things should be consider during implementation when the `opm index update` command is summoned: 
> 1. Validate the files inside the folder given as input to the command to ensure they are valid json representation of packages (and not random files with random content, which could be potential security risks) 
> 2. If the files are valid, take a git diff between the present content of the index (i.e the existing json representations) and the content of the folder given as input. Display the diff as an output of the command with the following message 
> "The following changes will be updated in the index:"
> If there is no diff between the content in the container image and the content of the local folder, throw an error with the message "No diff found between index and local content to update."    


#### Creating new indexes using the config files from an existing index

A new sub-command `create` will be introduced under `opm index`, that will take a folder containing json/yaml representation of packages, and a container image tag as input. The sub-command will create a new index container, walk the tree of files to find the json representations and store them in the container. 

```bash
$ tree community-operators 
community-operators
├── amqstreams.json
└── etcd.json
$ opm index create --from=community-operators --tag=docker.io/some-namespace/community-operators
```   

#### Validating a package inside the index

A new sub-command `validate` will be introduced under `opm index`, that will validate the content of each package's json representation. These validations will be in the context of rules about the structure of a package config in an index (required fields, naming conventions etc). A best effort is currently being made to aggregate these rules from various sources and maintained in the [api project's validation library](https://github.com/operator-framework/api/tree/master/pkg/validation). The `opm index validate` will be responsible for validating the index for authors to make sure that the packages in the index conforms to these rules. 

#### Usage of the json representation of packages by olm components

The sqlite database that is currently created inside the index is used to serve contents of the index over a gRPC api.

```go
rpc ListPackages(ListPackageRequest) returns (stream PackageName) {}
rpc GetPackage(GetPackageRequest) returns (Package) {}
rpc GetBundle(GetBundleRequest) returns (Bundle) {}
rpc GetBundleForChannel(GetBundleInChannelRequest) returns (Bundle) {}
rpc GetChannelEntriesThatReplace(GetAllReplacementsRequest) returns (stream ChannelEntry) {}
rpc GetBundleThatReplaces(GetReplacementRequest) returns (Bundle) {}
rpc GetChannelEntriesThatProvide(GetAllProvidersRequest) returns (stream ChannelEntry) {}
rpc GetLatestChannelEntriesThatProvide(GetLatestProvidersRequest) returns (stream ChannelEntry) {}
rpc GetDefaultBundleThatProvides(GetDefaultProviderRequest) returns (Bundle) {}
rpc ListBundles(ListBundlesRequest) returns (stream Bundle) {}
```

Once the declarative package configs are available inside an index, these configs will be used to serve the content for these api endpoints instead of the sqlite database.
The `opm registry serve` will auto-detect the filetype of the `--database` flag and transparently handle both sqlite database files and declarative configs. If the database is formatted as sqlite, `opm registry serve` will internally migrate it to its declarative config representation prior to serving the GRPC API. When `opm registry serve` exits, the sqlite database will be preserved, and any temporary declarative config files will be cleaned up.

## Deprecations

Support for databases in the sqlite format will be marked for deprecation. All `opm` commands that interact with the database will print a deprecation warning on the
command line when they detect an sqlite database is present. Users should plan to explicitly migrate their index images to the declarative config format because support for sqlite will be removed in a future release.

Additionally, `prune`, `prune-stranded` and `rm` sub-commands under the `opm index` command will also be marked for deprecation.


## Migration Plan

The `opm` command will automatically migrate an sqlite-based database to a declarative config-based database according to the following rules:
1. Existing `opm index` commands will support images containing the deprecated sqlite representation or the declarative config-based tar or directory representation. All `opm index` subcommands will automatically migrate the input database to the declarative config directory representation, and then perform the desired underlying registry action against the declarative config representation.
2. Existing `opm registry` commands will support input databases with the deprecated sqlite representation or the declarative config-based tar or directory representation. If the input database is a file (e.g. sqlite or declarative config tar), `opm registry` commands that modify the database will always output the declarative config tar format. This ensures that existing clients of `opm registry` who have not explicitly migrated can continue treating the database as an opaque file. If the input database is a declarative config directory, `opm registry` preserves the fact that it is a directory. `opm registry serve` does not have any permanent filesystem side effects.
3. For existing commands, no new flags are being introduced to distinguish sqlite input from declarative config input. All input database formats can be provided via the `--database` flag.


The index images that exist today have the sqlite database in them, and have been built with the following Docker configuration:

```Dockerfile
ENTRYPOINT ["/bin/opm"]
CMD ["registry", "serve", "--database", "/database/index.db"]
```

Since the new `opm` binary will implicitly migrate the database in-place to the declarative config format, no change is required in the index image entrypoint.

```Dockerfile
ENTRYPOINT ["/bin/opm"]
CMD ["registry", "serve", "--database", "/database/index.db"]
```

## Test plan

* New tests that test the behavior of the `add`, `update`, `create`, `validate` sub-commands for `opm index` will be added to `opm`. 

* Since the configs will be used to serve content of the index over the gRPC api instead of the sqllite database, passing of the existing portfolio of tests for the api endpoints listed above will serve as proof that the configs provide the same data as the sqllite database used to, once the switch is made to serve content with the configs. 


