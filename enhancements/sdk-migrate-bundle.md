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

The Operator Registry `opm` command has a command to build bundles,
`alpha build`, that we would reuse. The command is pretty well segmented that we
could easily reuse the `GenerateFunc` function to create the `bundle` as a
directory. There is also support for building the bundle as an image which we
could do using the `BuildBundleImage` function.

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

### Risks and Mitigations

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

20210208 - Initial proposal to migrate bundle

## Drawbacks

N/A

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used to
highlight and record other possible approaches to delivering the value proposed
by an enhancement.

## Infrastructure Needed [optional]

N/A
