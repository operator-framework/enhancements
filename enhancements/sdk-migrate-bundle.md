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
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

1. The bundles we build today are done via `docker` from the project's
   `Makefile` where users can adapt their image builder to whatever they want.
   Do we want to allow this from the CLI?

## Summary

Add a subcommand to the Operator SDK CLI that will allow users to migrate
their operator's packagemanifest format to the new bundle format.

## Motivation

There are operators that exist that are still using the packagemanifest
format. This subcommand will allow them to migrate these packagemanifests to
bundles which is the preferred OLM packaging format.

### Goals

- Provide a subcommand to migrate one packagemanifest to a set of bundles.

### Non-Goals

- Solve the world?
- There will be no migration from bundles to packagemanifests

## Proposal

### User Stories [optional]

#### Story 1

As an operator developer, I would like to migrate my operator which uses
the package manifest format to the new bundle format.

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

The Operator SDK will make use of the API available from the Operator
Registry's `opm` command. The command is pretty well segmented that we could
easily reuse the `GenerateFunc` function to create the `bundle` as a
directory. There is also support for building the bundle as an image which
we could do using the `BuildBundleImage` function.

#### Generating bundle to disk

The `GenerateFunc` takes in the following arguments:

- directory string - local directory where the manifests and metadata are
  located
- outputDir string - where manifests and metadata are copied
- packageName string - the name of the package that bundle image belongs to
- channels string - the list of channels that bundle image belongs to
- channelDefault string - the default channel for the bundle image
- overwrite bool - flag to enable overwriting annotations.yaml

In order to use this function, we will obtain the required parameters from the
listed *source*:

| parameter | source |
| --------- | ------ |
| directory | package manifest directory |
| outputDir | default `bundle` dir or flag `--bundle-dir` |
| packageName | packageName from `*.package.yaml` |
| channels | ??? |
| channelDefault | defaultChannel from `*.package.yaml` or `stable` |
| overwrite | `--overwrite` parameter |

#### Creating bundle image

If we choose to build bundle images, then we can leverage the `BuildBundleImage`
function from `opm`. This takes in two arguments:

- imageTag string - tag to use for your container image
- imageBuilder string - the container image builder i.e. `docker`, `podman`, etc

This would output a new bundle image.

In order to use this function, we will obtain the required parameters from the
listed *source*:

| parameter | source |
| --------- | ------ |
| imageTag | version of packagemanifest |
| imageBuilder | `docker` |

#### Command Line Interface

The new command line will look as follows:

```
operator-sdk migrate bundle <packagemanifestdir> [--build-image=] \
    [--bundle-dir=] [--overwrite]
```

- `<packagemanifestdir>` is a positional argument that specifies the
  packagemanifest directory.
- `[--build-image=]` is an optional flag that indicates whether we want to build
  the bundle as an image or just output to a directory, defaults to `bundle`
  directory.
- `[--bundle-dir=]` is an optional flag that indicates the directory to write the
  bundle to, defaults to `bundle` directory.
- `[--overwrite]` is an optional flag that indicates whether we should overwrite
  the `bundle`.

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
operator-sdk migrate bundle manifests
```

The command will generate 3 bundles in the default `bundle` directory:

```
bundle
├── bundle-0.6.1
│   ├── manifests
│   │   ├── etcdoperator.clusterserviceversion.yaml
│   │   ├── etcdoperator-controller-manager-metrics-service_v1_service.yaml
│   │   ├── etcdoperator-metrics-reader_rbac.authorization.k8s.io_v1beta1_clusterrole.yaml
│   │   └── etcdcluster.crd.yaml
│   └── metadata
│       └── annotations.yaml
├── bundle-0.6.1.Dockerfile
├── bundle-0.9.1
│   ├── manifests
│   │   ├── etcdoperator.clusterserviceversion.yaml
│   │   ├── etcdoperator-controller-manager-metrics-service_v1_service.yaml
│   │   ├── etcdoperator-metrics-reader_rbac.authorization.k8s.io_v1beta1_clusterrole.yaml
│   │   ├── etcdbackup.crd.yaml
│   │   ├── etcdcluster.crd.yaml
│   │   └── etcdrestore.crd.yaml
│   └── metadata
│       └── annotations.yaml
├── bundle-0.9.1.Dockerfile
├── bundle-0.9.2
│   ├── manifests
│   │   ├── etcdoperator.clusterserviceversion.yaml
│   │   ├── etcdoperator-controller-manager-metrics-service_v1_service.yaml
│   │   ├── etcdoperator-metrics-reader_rbac.authorization.k8s.io_v1beta1_clusterrole.yaml
│   │   ├── etcdbackup.crd.yaml
│   │   ├── etcdcluster.crd.yaml
│   │   └── etcdrestore.crd.yaml
│   └── metadata
│       └── annotations.yaml
└── bundle-0.9.2.Dockerfile
```

#### Complex Example

```
operator-sdk migrate bundle manifests --bundle-dir=my-bundle --overwrite
```

This will generate the bundles in a `my-bundle` directory. If there was an
existing `my-bundle` the `--overwrite` will delete that directory before writing
the new bundles.

```
my-bundle
├── my-bundle-0.6.1
│   ├── manifests
│   │   ├── etcdoperator.clusterserviceversion.yaml
│   │   ├── etcdoperator-controller-manager-metrics-service_v1_service.yaml
│   │   ├── etcdoperator-metrics-reader_rbac.authorization.k8s.io_v1beta1_clusterrole.yaml
│   │   └── etcdcluster.crd.yaml
│   └── metadata
│       └── annotations.yaml
├── my-bundle-0.6.1.Dockerfile
├── my-bundle-0.9.1
│   ├── manifests
│   │   ├── etcdoperator.clusterserviceversion.yaml
│   │   ├── etcdoperator-controller-manager-metrics-service_v1_service.yaml
│   │   ├── etcdoperator-metrics-reader_rbac.authorization.k8s.io_v1beta1_clusterrole.yaml
│   │   ├── etcdbackup.crd.yaml
│   │   ├── etcdcluster.crd.yaml
│   │   └── etcdrestore.crd.yaml
│   └── metadata
│       └── annotations.yaml
├── my-bundle-0.9.1.Dockerfile
├── my-bundle-0.9.2
│   ├── manifests
│   │   ├── etcdoperator.clusterserviceversion.yaml
│   │   ├── etcdoperator-controller-manager-metrics-service_v1_service.yaml
│   │   ├── etcdoperator-metrics-reader_rbac.authorization.k8s.io_v1beta1_clusterrole.yaml
│   │   ├── etcdbackup.crd.yaml
│   │   ├── etcdcluster.crd.yaml
│   │   └── etcdrestore.crd.yaml
│   └── metadata
│       └── annotations.yaml
└── my-bundle-0.9.2.Dockerfile
```

#### Prototype Links

- migrate-prototype: to see what `GenerateFunc` can do:
  https://github.com/jmrodri/migrate-prototype/blob/master/main.go

### Risks and Mitigations
If we can not change the behavior of `GenerateFunc` to write the
`bundle.Dockerfile` in a particular location this could make this process
difficult.

## Design Details

### Test Plan

- add unit tests for migration code
- add e2e tests to migrate packagemanifests to bundles
- manual testing of `migrate bundle` will be done as well

### Graduation Criteria

The `migrate bundle` command will come in as non-alpha and be ready to
use once merged.

### Upgrade / Downgrade Strategy

N/A

### Version Skew Strategy

N/A

## Implementation History

20210210 - Add prototype and more explicit details.
20210209 - Generate 2 bundles
20210208 - Initial proposal to migrate bundle

## Drawbacks

N/A

## Alternatives

There are 2 alternatives that I can come up with:

1. Use a different command name `convert packagemanifest` which would take the
   same arguments and flags as the `migrate bundle`. The idea is that we are
   converting a packagemanifest into a bundle.

1. Second alternative would be shell script that could migrate packagemanifests
   instead of a subcommand. This would mean distributing a new script which
   isn't really something we have in place today. It would probably get out of
   sync since it isn't part of the main codebase.

## Infrastructure Needed [optional]

- `GenerateFunc` will need a way to write the `bundle.Dockerfile` to a different
  directory. Today it writes it to the current working directory with no
  overrides.
