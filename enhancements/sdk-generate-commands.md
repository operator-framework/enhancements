---
title: README describing new command workflows for OLM enabled operators
authors:
  - "@varshaprasad96"
  - "@estroz"
reviewers:
  - "@jlanford"
  - "@dmesser"
  - "@jmrodri"
approvers:
  - "@jlanford"
  - "@dmesser"
  - "@jmrodri"
creation-date: 2020-04-09
last-updated: 2020-05-19
status: implementable
see-also:
  - https://github.com/operator-framework/operator-sdk/pull/2860
---

# README describing new command workflows for OLM enabled operators

## Release Signoff Checklist

- \[x\] Enhancement is `implementable`
- \[x\] Design details are appropriately documented from clear requirements
- \[x\] Test plan is defined
- \[ \] Graduation criteria for dev preview, tech preview, GA
- \[ \] User-facing documentation is created.

## Open Questions:

* _(Old project layouts)_ Should a user be able to generate both a bundle and
package manifests format in the same directory (for migration purposes), or should
they be mutually exclusive, ex. `generate bundle` will return an error if a
`generate packagemanifests` has been run in the same output directory, and visa versa?

## Summary

`operator-sdk generate csv` is the current entrypoint into Operator packaging
for developers of Operator SDK projects. This command attempts to handle both
[bundle][bundle-format] and [package manifests][package-manifests-format] format
generation scenarios with a plethora of flags. The number of flags and confusing
command name (`generate csv` manages higher-level formats) creates non-ideal UX.
This proposal details new workflows for bundle and package manifests formats
for old and [new][ep-kb-integration] `operator-sdk`-build Operator projects.

To clearly define what old and new layouts mean:
* Old project layouts: scaffolded with `operator-sdk new`
* New project layouts: scaffolded with `operator-sdk init`

We plan to improve the developer experience for operator authors using Operator SDK by providing:
1. One `operator-sdk` subcommand _per format_ for generating manifests and metadata.
1. Interactive-by-default commands to get UI information for packaged manifests and metadata.

[bundle-format]:https://github.com/operator-framework/operator-registry/blob/master/docs/design/operator-bundle.md#bundle-manifest-format
[package-manifests-format]:https://github.com/operator-framework/operator-registry/tree/v1.5.3/#manifest-format
[ep-kb-integration]:https://github.com/operator-framework/operator-sdk/blob/master/proposals/kubebuilder-integration.md

## Motivation

Currently, the `operator-sdk generate csv` command is used for generating CSVs,
which scaffolds either a bundle or package manifests format and writes a
[`ClusterServiceVersion` (CSV)][doc-csv] manifest to disk depending on the format selected.
This process requires a developer to manually edit the CSV when any required
field is missing, or if they want to add certain pieces of [UI metadata][doc-csv-metadata].
Furthermore selecting the format is non-obvious; adding to the confusion is
the fact that the operator-framework has no plans to deprecate the package manifests format
in the near future, so having documentation for both formats in one command is confusing.

[doc-csv]:https://github.com/operator-framework/operator-lifecycle-manager/blob/0.14.2/doc/design/building-your-csv.md
[doc-csv-metadata]:https://github.com/operator-framework/operator-lifecycle-manager/blob/0.14.2/doc/design/building-your-csv.md#your-custom-resource-definitions

## Goals

* Simplify UX for generating bundle and package manifests formats.
* Delineate bundle and package manifests format generation and usage.
* _(New project layouts)_ Use a Makefile wherever possible.

## Non-Goals

* Deprecate or remove package manifests support from the Operator SDK.
* Discuss the `run` subcommand in relation to bundles or package manifests.

## Proposal

There are two parts to this proposal:

1. Add `generate bundle` and `generate packagemanifests` subcommands for both
old and new project layouts.
1. _(New project layouts)_ Add Makefile recipes `make bundle` and `make bundle-build`
that invoke `generate bundle` and `docker build -f bundle.Dockerfile .`, respectively.

### User Stories

#### Story 1

As an Operator developer of a new layout-style project, I want to generate a
bundle format on disk using `kustomize` or from manifests on disk in a custom location.

#### Story 2

As an Operator developer of an old layout-style project, I want to generate a
bundle format on disk using manifests existing in my `deploy` directory.

#### Story 3

As an Operator developer of a new layout-style project, I want to generate a
package manifests format on disk using `kustomize` or from manifests on disk in
a custom location.

#### Story 4

As an Operator developer of an old layout-style project, I want to generate a
package manifests format on disk using manifests existing in my `deploy` directory.

### Implementation Details/Notes/Constraints

#### `kustomize` usage (new project layouts)

Bundle and package manifests generation should leverage `kustomize` because manifests
in `config` are meant to be configurable, i.e. for a particular deployment strategy.
For example, `make deploy` will call `kustomize build` on the default `kustomization.yaml`
to generate a working set of Operator manifests that is directly applied to a cluster.
A bundle or package should contain manifests configured to be cluster-ready, therefore
both `bundle` and `packagemanifests` subcommands must be able to consume manifests
from stdin.

New project layouts rely on `kustomize build` to generate cluster-ready manifests,
the output of which written either to `kubectl apply -f -` or disk if desired.
However, each command in each scenario should be able to handle direct input from
manifests on disk. This mode is important for new project layouts in the case where the
environment cannot use `kustomize` when generating either format, ex. in a pipeline.
For old project layouts, this is the default. New project layouts,
the default is to use `kustomize build config/default | operator-sdk generate <bundle|packagemanifests>`.

#### `generate` subcommand commonalities

Both `bundle` and `packagemanifests` subcommands will have the same underlying
CSV generator configured differently on invocation. The former generates the CSV
in a `manifests` directory, while the latter does so in a semver directory `X.Y.Z`.
These subcommands generate different metadata, discussed below. The concept of a "base"
(where an existing bundle or package is read from) is common to both subcommands,
but the source of this base depends both on subcommand and project layout.

#### `generate bundle`

All files/manifests will be generated under `config/bundle`, ex. `config/bundle/manifests`.
The functionality of this command is split up into steps by flag:
`--kustomize` (for new project layouts only), `--manifests`, and `--metadata`.

###### `--kustomize` (new project layouts)

Generate a `kustomize` base CSV for a new project layout under `config/bundle/bases`.
This base will be used as an actual base for `kustomize build` eventually; for now
only `generate bundle` will read this base. This step involves generating UI
metadata from API types, although `--manifests` can do this as well. This subcommand
is interactive if a base does not exist.

New project layout defaults:
* Output directory: `config/bundle/bases`
* Read from: `--apis-dir="api(s)"`

###### `--manifests`

Generate a bundle containing a CSV and owned CRD manifests. This step uses a base
generated with `--kustomize`, or creates a new one. This subcommand is interactive
if a base does not exist.

New project layout defaults:
* Output directory: `config/bundle/manifests`
* Read from: stdin, `--apis-dir="api(s)"`

Old project layout defaults:
* Output directory: `deploy/olm-catalog/<operator-name>/manifests`
* Read from: `--manifest-root="deploy"`, `--crds-dir="deploy/crds"`, `--apis-dir="pkg/apis"`

###### `--metadata`

Generates a `bundle.Dockerfile` and `metadata/annotations.yaml`. Many flags exposed
by [`opm alpha bundle generate`][opm-bundle-generate] will also be exposed by this
step, and set by variables passed to `make`.

New project layout defaults:
* Output directory: `config/bundle/metadata`
* Read from: _none_

Old project layout defaults:
* Output directory: `deploy/olm-catalog/<operator-name>/metadata`
* Read from: _none_

Notable flags: `--channels=[channel-names]`, `--default-channel=<channel-name>`, `--overwrite=<bool>`

#### `genenerate packagemanifests`

All files/manifests will be generated under `config/packages`, ex. `config/packages/0.0.1`.
The functionality of this command is split up into steps by flag:
`--kustomize` (for new project layouts only) and `--manifests`.

###### `--kustomize` (new project layouts)

Generate a `kustomize` base CSV for a new project layout under `config/packages/bases`.
This base will be used as an actual base for `kustomize build` eventually; for now
only `generate packagemanifests` will read this base. This step involves generating UI
metadata from API types, although `--manifests` can do this as well. This subcommand
is interactive if a base does not exist.

New project layout defaults:
* Output directory: `config/packages/bases`
* Read from: `--apis-dir="api(s)"`

###### `--manifests`

Generate a package containing a CSV and owned CRD manifests, and updates the
[package manifest file][package-manifest-doc]. This step uses a base generated
with `--kustomize`, or creates a new one. This subcommand is interactive if a
base does not exist.

New project layout defaults:
* Output directory: `config/packages/<version>`
* Read from: stdin, `--apis-dir="api(s)"`

Old project layout defaults:
* Output directory: `deploy/olm-catalog/<operator-name>/<version>`
* Read from: `--manifest-root="deploy"`, `--crds-dir="deploy/crds"`, `--apis-dir="pkg/apis"`

Notable flags: `--version=<semver>`, `--channel=<channel-name>`, `--default-channel=<channel-name>`

#### `make` recipes (new project layouts)

New project layouts are scaffolded with a [Makefile][kb-makefile] containing a
set of pre-defined recipes for generating manifests, building images, etc. To align
with this idiom, recipes should be amended/added to include `generate bundle` by
default. If a user does not want these features turned on in their project, they
can manually remove the amendments/additions.

###### Modifying the base scaffold

`operator-sdk init` will scaffold a Makefile containing the following amendments/additions:

```make
# Current Operator version
VERSION ?= 0.0.1
# Default bundle image tag
BUNDLE_IMG ?= controller-bundle:$(VERSION)
# Options for 'bundle-build'
ifeq ($(origin BUNDLE_CHANNELS), undefined)
BUNDLE_CHANNELS = --channels=$(BUNDLE_CHANNELS)
endif
ifeq ($(origin BUNDLE_DEFAULT_CHANNEL), undefined)
BUNDLE_DEFAULT_CHANNEL := --default-channel=$(BUNDLE_DEFAULT_CHANNEL)
endif
BUNDLE_METADATA_OPTS ?= $(BUNDLE_CHANNELS) $(BUNDLE_DEFAULT_CHANNEL)

...

# Existing manifests recipe, append 'generate bundle'
manifests:
  ...
  operator-sdk generate bundle --kustomize

...

# Generate bundle manifests
bundle: manifests
	kustomize build config/bundle | operator-sdk generate bundle --manifests --version $(VERSION)

# Generate bundle manifests with a particular version and metadata,
# validate generated bundle files, and build the bundle image.
bundle-build: manifests
	kustomize build config/bundle | operator-sdk generate bundle --manifests --metadata --overwrite --version $(VERSION) $(BUNDLE_METADATA_OPTS)
	operator-sdk bundle validate config/bundle
	docker build -f bundle.Dockerfile -t $(BUNDLE_IMG) .
```

A user can either invoke `make bundle` or `make bundle-build` or both; there is
no precedence requirement.

[opm-bundle-generate]:https://github.com/operator-framework/operator-registry/blob/master/docs/design/operator-bundle.md#generate-bundle-annotations-and-dockerfile
[package-manifest-doc]:https://github.com/operator-framework/operator-lifecycle-manager#discovery-catalogs-and-automated-upgrades
[kb-makefile]:https://github.com/kubernetes-sigs/kubebuilder/blob/7bdcf98/pkg/scaffold/internal/templates/makefile.go#L58-L151

## Design Details

### Test Plan

New project layout scenarios:
* [BDD-style E2E tests][pr-e2e-new-layout] that perform the "happy path" workflow for both commands.
* Unit tests for underlying generators.

Old project layout scenarios:
* Subcommand tests in a similar style to what [currently exists][dir-e2e-old-layout].
* Unit tests for underlying generators.

### Graduation Criteria

#### New project layout examples

Both new project layout examples assume the following setup preceded them:

```console
$ mkdir -p $HOME/projects/example-inc/memcached-operator
$ cd $HOME/projects/example-inc/memcached-operator
$ operator-sdk init --domain example.com --repo github.com/example-inc/memcached-operator
$ operator-sdk create api --group cache --version v1alpha1 --kind Memcached --resource --controller
$ export USERNAME=<username>
```

##### `generate bundle`

```console
$ make bundle-build IMG=quay.io/$USERNAME/memcached-operator-bundle:v0.0.1
controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
operator-sdk generate bundle -q --kustomize
Provide DisplayName for your operator:		
Provide version for your operator:		
Provide minKubeVersion for CSV:		
...
kustomize build config/bundle | operator-sdk generate bundle -q --manifests --metadata --overwrite --version 0.0.1
INFO[0003] Building annotations.yaml                    
INFO[0003] Building Dockerfile                          
operator-sdk bundle validate config/bundle
INFO[0000] Found annotations file                        bundle-dir=/home/estroz/go/src/github.com/test-org/memcached-operator-kb/config/bundle
INFO[0000] Could not find optional dependencies file     bundle-dir=/home/estroz/go/src/github.com/test-org/memcached-operator-kb/config/bundle
INFO[0000] All validation tests have completed successfully  bundle-dir=/home/estroz/go/src/github.com/test-org/memcached-operator-kb/config/bundle
docker build -f bundle.Dockerfile -t controller-bundle:0.0.1 .
Sending build context to Docker daemon  42.33MB
Step 1/9 : FROM scratch
 --->
Step 2/9 : LABEL operators.operatorframework.io.bundle.mediatype.v1=registry+v1
...
$ make docker-push IMG=quay.io/$USERNAME/memcached-operator-bundle:v0.0.1
```

```console
$ make bundle
controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
operator-sdk generate bundle -q --kustomize
Provide DisplayName for your operator:		
Provide version for your operator:		
Provide minKubeVersion for CSV:		
...
kustomize build config/bundle | operator-sdk generate bundle -q --manifests --metadata --overwrite --version 0.0.1
operator-sdk bundle validate config/bundle
INFO[0000] Found annotations file                        bundle-dir=/home/estroz/go/src/github.com/test-org/memcached-operator-kb/config/bundle
INFO[0000] Could not find optional dependencies file     bundle-dir=/home/estroz/go/src/github.com/test-org/memcached-operator-kb/config/bundle
INFO[0000] All validation tests have completed successfully  bundle-dir=/home/estroz/go/src/github.com/test-org/memcached-operator-kb/config/bundle
```

##### `generate packagemanifests`

```console
$ make manifests
controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
operator-sdk generate packagemanifests -q --kustomize
Provide DisplayName for your operator:		
Provide version for your operator:		
Provide minKubeVersion for CSV:		
...
$ kustomize build config/packagemanifests | operator-sdk generate packagemanifests -q --manifests --channel beta --version 0.0.1
```

#### Old project layout examples

Both old project layout examples assume the following setup preceded them:

```console
$ mkdir -p $HOME/projects/example-inc/
$ cd $HOME/projects/example-inc/
$ operator-sdk new memcached-operator --repo github.com/example-inc/memcached-operator
$ cd memcached-operator
$ operator-sdk add api --api-version cache.example.com/v1alpha1 --kind Memcached
$ operator-sdk add controller --api-version cache.example.com/v1alpha1 --kind Memcached
$ export USERNAME=<username>
```

##### `generate bundle`

```console
$ operator-sdk generate bundle --version 0.0.1
Provide DisplayName for your operator:		
Provide version for your operator:		
Provide minKubeVersion for CSV:		
...
$ docker build -f bundle.Dockerfile -t quay.io/$USERNAME/memcached-operator-bundle:v0.0.1 .
$ docker push quay.io/$USERNAME/memcached-operator-bundle:v0.0.1
```

##### `generate packagemanifests`

```console
$ operator-sdk generate packagemanifests --channel beta --version 0.0.1
Provide DisplayName for your operator:		
Provide version for your operator:		
Provide minKubeVersion for CSV:		
...
```

### Risks and Mitigations

* _(New project layouts)_ Some Operators have different "subtypes",
ex. namespaced vs clusterwide, that use different CSV's. Therefore they may
have more than one bundle or package manifests format per project,
ex. `config/bundle/operator-namespaced` and `config/bundle/operator-clusterwide`.
They will have to set `--output-dir` in their Makefile's, and this should be
documented somewhere. This use case is atypical and probably shouldn't be
supported by default, much like [multigroup][kb-multigroup].

[pr-e2e-new-layout]:https://github.com/operator-framework/operator-sdk/pull/3060
[dir-e2e-old-layout]:https://github.com/operator-framework/operator-sdk/tree/v0.17.1/hack/tests
[kb-multigroup]:https://book.kubebuilder.io/migration/multi-group.html
