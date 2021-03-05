---
title: sdk-migrate-bundle
authors:
  - "@jmrodri"
reviewers:
  - "@estroz"
  - "@ecordell"
  - "@rashmigottipati"
approvers:
  - "@estroz"
  - "@rashmigottipati"
creation-date: 2021-02-08
last-updated: 2021-02-08
status: implementable
see-also:
replaces:
superseded-by:
---

# Migrate Bundle

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [X] Design details are appropriately documented from clear requirements
- [X] Test plan is defined
- [X] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

1. The bundles we build today are done via `docker` from the project's
   `Makefile` where users can adapt their image builder to whatever they want.
   Do we want to allow this from the CLI?
   - if we do build images, we should add image builder flag
1. What will happen to the existing command in SDK for deploying an operator
   packagemanifests using OLM? Do we still want to support that and will it stay intact?
   - we will deprecate the `run packagemanifests` command and remove in SDK 2.0

## Summary

Add a subcommand to the Operator SDK CLI that will allow users to migrate
their operator's packagemanifest format to the new bundle format.

## Motivation

There are operators that exist that are still using the packagemanifest
format. This subcommand will allow them to migrate these packagemanifests to
bundles which is the preferred OLM packaging format.

### Goals

- Provide a subcommand to migrate one packagemanifest to a set of bundles.
- Deprecate `packagemanifests` features:
  - Deprecate `run packagemanifests`
  - Deprecate `generate packagemanifests`

### Non-Goals

- Solve the world?
- There will be no migration from bundles to packagemanifests

## Proposal

### User Stories [optional]

#### Story 1

As an operator developer, I would like to migrate my operator which uses
the packagemanifest format to the new bundle format.

#### Story 2

As an catalog admin for my operators, I would like to be able to migrate a
collection of packagemanifests to bundles.

### Implementation Details/Notes/Constraints [optional]

The Operator SDK currently supports generating & validating
packagemanifests & bundles. This enhancement will add a new command, `migrate
bundle` to the CLI. This new command will migrate an existing packagemanifest to
a bundle.

More information about the different format can be found at the URLs below:

- Operator bundles: https://github.com/operator-framework/operator-registry/blob/master/docs/design/operator-bundle.md
- Package manifests format: https://github.com/operator-framework/operator-registry/tree/v1.5.3/#manifest-format

The Operator SDK will traverse the given packagemanifest directory and for each
of the versions create a new bundle.

#### Command Line Interface

The new command line will look as follows:

```
operator-sdk pkgman-to-bundle <packagemanifestdir> [--build-image=] \
    [--output-dir=] [--image-base=] [--build-cmd=]
```

- `<packagemanifestdir>` is a positional argument that specifies the
  packagemanifest directory.
- `[--build-image=]` is an optional flag that indicates whether we want to build
  the bundle as an image or just output to a directory, defaults to `bundle`
  directory.
- `[--output-dir=]` is an optional flag that indicates the directory to write the
  bundle to, defaults to `bundle` directory.
- `[--image-base]` optional flag that indicates the base container name for
  the bundles; e.g. `quay.io/example/memcached-operator-bundle`
- `[--build-cmd]` optional flag that indicates the fully qualified build command;

For more information about the `image-base` and `build-cmd` see the
[creating bundle image](#creating-bundle-image) section.

#### Generating bundle on disk

In order to generate bundles to disk, we will refactor the `generate bundle`
logic to be reusable in generating the bundles.

Today the logic lives in `cmd/operator-sdk/generate/bundle/bundle.go`, this
would likely need to move to a shared location so that this new subcommand can
reuse it. The `pkgman-to-bundle` subcommand will have logic to read the
packagemanifests and supply the above bundle generate with the appropriate
input. This input will be TBD depending on how the refactoring goes.

#### Creating bundle image

If we choose to build bundle images, we will likely need a sane default build
command but also offer an override.

This would add 2 more flags to the CLI:

- `[--image-base]` optional flag that indicates the base container name for
  the bundles; e.g. `quay.io/example/memcached-operator-bundle`
- `[--build-cmd]` optional flag that indicates the fully qualified build command;

These flags would output a bundle image for each bundle created. We would use
the `<dirname>` for the tags.

The `--image-base` needs to be valid docker image name without the tag. We will
use the bundle directory for the imagetag.

Since we can't always anticipate what folks want to do with their builds, we'll
have an escape hatch flag, `--build-cmd`, which will take in the full build
command to run. This will include the builder i.e. `podman`, `docker`, etc. This
command will need to be in the `PATH` or they must supply the fully qualified
path name. For example, `podman build -t quay.io/example/bundle ...` or
`/usr/bin/docker build -t ...` .

#### Assumptions

- Existing packagemanifest directory

#### Example UX Scenarios

For each of the scenarios below, assume we have a packagemanifest dir with the
following layout.


```
manifests
└── etcd
    ├── 0.6.1
    │   ├── etcdcluster.crd.yaml
    │   └── etcdoperator.clusterserviceversion.yaml
    ├── 0.9.0
    │   ├── etcdbackup.crd.yaml
    │   ├── etcdcluster.crd.yaml
    │   ├── etcdoperator.v0.9.0.clusterserviceversion.yaml
    │   └── etcdrestore.crd.yaml
    ├── 0.9.2
    │   ├── etcdbackup.crd.yaml
    │   ├── etcdcluster.crd.yaml
    │   ├── etcdoperator.v0.9.2.clusterserviceversion.yaml
    │   └── etcdrestore.crd.yaml
    └── etcd.package.yaml
```

#### Simple Example

Assuming the packagemanifest above, let's take the simplest example:

```
operator-sdk pkgman-to-bundle manifests
```

The command will generate 3 bundles in the default `bundle` directory:

```
bundle
├── bundle-0.6.1
│   ├── bundle.Dockerfile
│   ├── manifests
│   │   ├── etcdcluster.crd.yaml
│   │   ├── etcdoperator.clusterserviceversion.yaml
│   │   ├── etcdoperator-controller-manager-metrics-service_v1_service.yaml
│   │   └── etcdoperator-metrics-reader_rbac.authorization.k8s.io_v1beta1_clusterrole.yaml
│   └── metadata
│       └── annotations.yaml
├── bundle-0.9.1
│   ├── bundle.Dockerfile
│   ├── manifests
│   │   ├── etcdbackup.crd.yaml
│   │   ├── etcdcluster.crd.yaml
│   │   ├── etcdoperator.clusterserviceversion.yaml
│   │   ├── etcdoperator-controller-manager-metrics-service_v1_service.yaml
│   │   ├── etcdoperator-metrics-reader_rbac.authorization.k8s.io_v1beta1_clusterrole.yaml
│   │   └── etcdrestore.crd.yaml
│   └── metadata
│       └── annotations.yaml
└── bundle-0.9.2
    ├── bundle.Dockerfile
    ├── manifests
    │   ├── etcdbackup.crd.yaml
    │   ├── etcdcluster.crd.yaml
    │   ├── etcdoperator.clusterserviceversion.yaml
    │   ├── etcdoperator-controller-manager-metrics-service_v1_service.yaml
    │   ├── etcdoperator-metrics-reader_rbac.authorization.k8s.io_v1beta1_clusterrole.yaml
    │   └── etcdrestore.crd.yaml
    └── metadata
        └── annotations.yaml
```

#### Complex Example

```
operator-sdk pkgman-to-bundle manifests --output-dir=my-bundle
```

This will generate the bundles in a `my-bundle` directory. If there was an
existing `my-bundle` the command will *delete* that directory before writing
the new bundles.

```
my-bundle
├── my-bundle-0.6.1
│   ├── bundle.Dockerfile
│   ├── manifests
│   │   ├── etcdoperator.clusterserviceversion.yaml
│   │   ├── etcdoperator-controller-manager-metrics-service_v1_service.yaml
│   │   ├── etcdoperator-metrics-reader_rbac.authorization.k8s.io_v1beta1_clusterrole.yaml
│   │   └── etcdcluster.crd.yaml
│   └── metadata
│       └── annotations.yaml
├── my-bundle-0.9.1
│   ├── bundle.Dockerfile
│   ├── manifests
│   │   ├── etcdoperator.clusterserviceversion.yaml
│   │   ├── etcdoperator-controller-manager-metrics-service_v1_service.yaml
│   │   ├── etcdoperator-metrics-reader_rbac.authorization.k8s.io_v1beta1_clusterrole.yaml
│   │   ├── etcdbackup.crd.yaml
│   │   ├── etcdcluster.crd.yaml
│   │   └── etcdrestore.crd.yaml
│   └── metadata
│       └── annotations.yaml
└── my-bundle-0.9.2
    ├── bundle.Dockerfile
    ├── manifests
    │   ├── etcdoperator.clusterserviceversion.yaml
    │   ├── etcdoperator-controller-manager-metrics-service_v1_service.yaml
    │   ├── etcdoperator-metrics-reader_rbac.authorization.k8s.io_v1beta1_clusterrole.yaml
    │   ├── etcdbackup.crd.yaml
    │   ├── etcdcluster.crd.yaml
    │   └── etcdrestore.crd.yaml
    └── metadata
        └── annotations.yaml
```

#### Deprecating packagemanifests commands

As part of this enhancement proposal, we will deprecate the following commands:

- `run packagemanifests`
- `generate packagemanifests`

To accomplish this we need to fill in the [`Deprecated string`][deprecated-cobra]
field of both [`run packagemanifests`][deprecate-run-pkgman] and [`generate
packagemanifests`][deprecate-gen-pkgman].

The exact wording for the message will be determined later.

#### Prototype Links

- migrate-prototype: to see what `GenerateFunc` can do:
  https://github.com/jmrodri/migrate-prototype/blob/master/main.go

### Risks and Mitigations
If we cannot change the behavior of `GenerateFunc` to write the
`bundle.Dockerfile` in a particular location this could make this process
difficult.

## Design Details

### Test Plan

- add unit tests for migration code
- add e2e tests to migrate packagemanifests to bundles
  - using a sample packagemanifest, run `operator-sdk pkgman-to-bundle`,
    then validate the bundle with `bundle validate`
- manual testing of `pkgman-to-bundle` will be done as well

### Graduation Criteria

The `pkgman-to-bundle` command will come in as non-alpha and be ready to
use once merged.

### Upgrade / Downgrade Strategy

N/A

### Version Skew Strategy

N/A

## Implementation History

20210306 - Remove usage of `GenerateFunc` and `BuildBundleImage`
20210225 - Add details about deprecated run|generate packagemanifests
20210224 - Deprecated packagemanifests features
20210224 - Fix typos; Remove overwrite flag; change bundle-dir to output-dir;
20210210 - Add prototype and more explicit details.
20210209 - Generate 2 bundles
20210208 - Initial proposal to migrate bundle aka pkgman-to-bundle

## Drawbacks

N/A

## Alternatives

There are 4 alternatives that I can come up with:

1. Use a different command name `convert packagemanifest` which would take the
   same arguments and flags as the `pkgman-to-bundle`. The idea is that we are
   converting a packagemanifest into a bundle. Because this is a short lived
   command it seems bad to waste the `convert` subcommand on this when that
   could be used for some other more useful purpose in the future.

1. Second alternative would be shell script that could migrate packagemanifests
   instead of a subcommand. This would mean distributing a new script which
   isn't really something we have in place today. It would probably get out of
   sync since it isn't part of the main codebase.

1. Another option is `migrate bundle` which would take the same arguments and
   flags as `pkgman-to-bundle`. Because this is a short lived command it seems
   like a bad idea to waste the `migrate` subcommand on this when that could be
   used for some other more useful purpose in the future.

1. Yet another option is `bundle migrate` which would take the same arguments and
   flags as `pkgman-to-bundle`. This command would fit with the existing
   `bundle` command but break the paradigm of the other commands which is
   `<verb> <noun>`.

## Infrastructure Needed [optional]

- [`GenerateFunc`][generatefunc] will need a way to write the `bundle.Dockerfile`
  to a different directory. Today it writes it to the current working directory
  with no overrides.

[generatefunc]: https://github.com/operator-framework/operator-registry/blob/master/pkg/lib/bundle/generate.go#L50
[deprecated-cobra]: https://github.com/spf13/cobra/blob/master/command.go#L85
[deprecate-gen-pkgman]: https://github.com/operator-framework/operator-sdk/blob/master/internal/cmd/operator-sdk/generate/packagemanifests/cmd.go#L57-L80
[deprecate-run-pkgman]: https://github.com/operator-framework/operator-sdk/blob/master/internal/cmd/operator-sdk/run/packagemanifests/packagemanifests.go#L32-L57
