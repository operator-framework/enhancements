# Pipeline Authors have Tools to build, modify, and consume the latest bundle and registry formats

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift/docs]

## Summary

There are several internal pipelines that deal with OLM operator data, all of which need to build or consume the newer bundle and registry formats. We can build new bundle and registry formats as per the other 4.4 epics, but if no one internally is using them we will not derive much of the desired value from them.

### Goals

* Maintainers of Operator Registries can leverage container registries so that they can host Operator bundles and indexes of all available bundles in this catalog in the form of container images
* Maintainers of Operator Registries can use a tool to:
    * create a bundle index so that it is easy to employ in a CI environment.
    * migrate to and from both old and new format.
    * merge indexes
    * delete the latest version of an operator from an index
    * download the contents of index images
    * specify related images for CVE updates.
    * validate the contents of an index before publishing it.
    * package/release/version tool(s)
* Pipeline admins can publish multiple applications simultaneously without conflicts.
* Developers or QE can use downstream images in a dev/stage/qe repository
* There is sufficient documentation for operator owners and maintainers of operatory registries.
* Design the migration of Community Operators to the new bundle format.

### Non-Goals

* Migrate Community Operators / operatorhub.io to the new bundle format.
* Pipeline admins rely on the tooling to extrapolate defined selective metadata from full set of each application operator manifests and include that in a combined catalog for index object
* Pipeline admins can update official catalog object with newly created single application operator and publish

---

## User Stories

### Migrate to and from each registry format

As an Operator Registry maintainer, I want to continue to push updates to old (appregistry-based) catalogs after having migrated to the new bundle format without having to maintain two repositories.

#### Design Proposal

Add a command to `opm index` and `opm alpha bundle` called `migrate` which:

* takes a collection of bundles migrates these to the old manifest format.

    **Example CLI:**
    `opm alpha bundle migrate --bundles quay.io/my/bundle:v1,quay.io/my/bundle:v2 --out path/to/my/dir`
    
* takes an index and migrate it to the old manifest format.
    
    **Example CLI:**
    `opm index migrate quay.io/my/index --out path/to/my/dir`
    
    where `path/to/my/dir` would look like:
    
    ```
    dir
    ├── bundle
    │   ├── v1
    │   │   ├── some.crd.yaml
    │   │   └── bundle.clusterserviceversion.yaml
    │   ├── v2
    │   │   ├── some.crd.yaml
    │   │   ├── some.other.crd.yaml
    │   │   ├── bundle.v2.clusterserviceversion.yaml
    │   └── bundle.package.yaml
    ```

* takes a manifest directory for a given operator and migrates it to an individual bundle

    Part of this is already implemented in the `opm alpha bundle build` command which takes the manifest for a specific version - for example `path/to/my/dir/bundle/v1` from above:
    
    ```
    dir
    ├── bundle
    │   ├── v1
    │   │   ├── some.crd.yaml
    │   │   └── bundle.clusterserviceversion.yaml
    ```
    
    and converts that to the new bundle format - including a Dockerfile to push this bundle to a container registry.
    
    Q: does it make sense to implement a wrapper for this command that takes a manifest directory instead of a single version to output an index or multiple bundles?

### Download the contents of an index

As an Operator Registry maintainer, Pipeline admin, or QE, I want to download or inspect the contents of a given index image.

#### Design Proposal

Add a command to `opm index` and `opm alpha bundle` called `extract` which extracts:

* the manifest and metadata folders for a given bundle

    **Example CLI:**
    
    `opm alpha bundle extract quay.io/my/bundle:v1 --out path/to/my/dir`

* a list of all the bundle images that are in an index
    
    **Example CLI 1:**
    
    `opm index extract quay.io/my/index --out path/to/my/dir --images-only`
    
    **Example CLI 2:** prints to stdout
    
    `opm alpha bundle extract quay.io/my/bundle:v1 --image-only`


* the manifest and metadata folders for all bundles in a given index

    **Example CLI:**
    
    `opm index extract quay.io/my/index --out path/to/my/dir`


### Delete latest operator version

As an Operator Registry maintainer, I want to have a way to remove the latest version of an operator (from an index)?

#### Design Proposal

Add a new flag to `opm registry rm` and `opm index rm` commands called `--latest` which:

* takes an index reference and package name, and removes the latest bundle version if present.
    
    **Example CLI:**
    `opm index rm --latest --from quay.io/my/index -o "operator1,operator2" -c=docker`

* takes a registry database and package names, and removes the latest bundle version if present.

    **Example CLI:**
    `opm registry rm --latest -d=path/to/my.db -o="operator1,operator2"`
    
### Merging Indexes

As an Operator Registry Maintainer I want to merge two indexes into one index.

#### Design Proposal

Initially we will only support disjoint indexes (i.e. indexes with no common packages) so that we don't have to deal with compatible versions and insertion order (which we currently cannot do implicitly). Once we change our update graph to rely solely on semver we could support merging intersecting indexes.

We need to add a command to `opm index` called `merge` which takes two index images and outputs either a new index image or generates a directory and dockerfile that can be used to build an image.

**Example 1:**

`opm index merge quay.io/my/index1 quay.io/my/index2 -t quay.io/my/superindex -c=docker`

**Example 2:**

`opm index merge quay.io/my/index1 quay.io/my/index2 --generate -c=podman`

### Get related images from an index

As a pipeline admin, I want to know all the related images of a particular index so that I can know to update the index image when CVE fixes affect the related images of a bundle in my index.

#### Design Proposal

Add a flag to `opm index extract` called `related-images` that takes an index image and returns a list of related images from the database.

**Example:**

`opm index extract quay.io/my/index --related-images -c=docker`

### CVE updates to running operators

TODO(lance)

### Design migration of upstream-community-operators

As a maintainer of the upstream-community-operators git repository, I want my catalog build automation to use the new bundle/index format so that I can:

- avoid storing operator manifests in the repository
- simplify the addition, update, and removal of operators
- reduce the time between an operator being submitted and it being displayed on operatorhub.io

#### Design Proposal

To design a migration, we'll paint a picture of a plausible final and work backward from there.

On top of conformance with the rest of the ecosystem, the value-add of bundles and indexes for upstream-community-operators a reduction in operator manifests managed in the repository. Ideally, these new features will shrink the operator inclusion mechanism from a set of directories to a finite number of config files.

Once fully migrated to the new formats, the general flow for updating the catalog's content might look like:

1. The git repository is forked
2. A bundle image reference is added, updated, or blacklisted in an index config file
3. A PR is opened against the git repository
4. An index is built with the patched config, using the current production index as a base
5. The index content is verified and tested
6. If tests pass, and the PR is approved, it enters a merge queue
7. If the production index image changes before it merges, the PR drops from the queue (GOTO 4)
8. Once merged, build a new production index image

An configuration file will inform a build of the bundles to add or remove, if any, from an index:

```yaml
from: image://quay.io/operator-framework/community-operators:latest
included:
- ref: quay.io/coreos/something-bundle:v1
  package: something
  channel: stable
- ref: quay.io/coreos/something-bundle@sha256:abcdef...
  package: something
  channel: alpha
- ref: quay.io/redhat/something-else-bundle:v5.5
  package: something-else
  channel: v5.Y
exluded:
- ref: quay.io/redhat/something-else-bundle:v5.4
  package: something-else
  channel: v5.Y
```

A new `-f` option will be added to the `opm index add` subcommand which specifies the path of an index config file. The command will parse the given file and idempotently:

- add bundles referenced by the `included` directive to the index specified by `from`
- remove bundles referenced by the `excluded` directive from the index specified by `from`

The command will also be capable of sourcing "from indexes" in varying formats using reference prefixes:

- "image://" for container image references
- "file://" for database file paths

TODO(njhale): staged migration to bundle images

1. create an index configuration specifying the existing catalog image in `from`
2. reduce the number of manifest versions kept for each operator to the head of each package/channel
3. walk manifest directories adding to the index database
4. run `opm index add` using the index config as an overlay

### Migrate away from appregistry

Detailed steps about migration from the pipeline perspective can be found here: https://docs.engineering.redhat.com/display/DELIVERY/New+OCP+Application+Registry%3A+Requirements

### Catalogs for minor OCP versions

As an operator registry maintainer, I need a way to specify what OCP versions are supported by the operators in the registry.

#### Design Proposal

We can use names of branches/tags on dist-git repos as convention for what ocp versions are supported. We can then update each OCP versioned index for pushes to each branch/tag. If this is not possible we can include this in a config file similar to `art.yaml` for openshift operators.

### Backing up the index image

As a pipeline admin I want to recover/rebuild an index that has been lost or corrupted.

#### Proposal

On every update to an index, run `opm index extract` on that newly created index and push that content (list of images, files, or dbs) to a git repo as a backup.

Q: Should we backup the db file, the image references, or the manifest/metadata folders?

Note: Order currently matters for insert into the db so backing up the db file may be the best first option - a list of bundles will work after some index changes involving adhering to semver for operator upgrade paths.

### Packaging and releasing `opm`

To start, we can use git releases to release the `opm` binary.

Q: Where do we compile it before we release it?

Q: If we start versioning it - how do versions of `opm` relate to versions of `operator-registry` and `olm`?
