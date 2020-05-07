---
title: run-bundle
authors:
  - "@jmrodri"
reviewers:
  - "@joelanford"
  - "@estroz"
approvers:
  - "@joelanford"
  - "@estroz"
creation-date: 2020-05-07
last-updated: 2020-05-07
status: implementable
see-also:
replaces:
superseded-by:

---

# Run Bundle

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [X] Design details are appropriately documented from clear requirements
- [X] Test plan is defined
- [X] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

1. We currently check OLM installation status with `operator-sdk olm status`.
   Can we change that to using the discovery API?
2. What other flags should the command take?
   a. “--operator-namespace”
   b. “--watch-namespaces”
3. How do we handle --mode= flag on registry add
   a. should we ALWAYS default to --mode=semver?
   b. should we have a flag that allows our default mode handling to be overridden?

## Summary

Enhance the Operator SDK `run` command to support the new bundle format.

## Motivation

One of the primary goals of the Operator SDK is to make it easy for operator
developers to run and test their operators with Operator Lifecycle
Manager (OLM). There currently exists a run --olm command which supports the
packagemanifest format. In order to add support for the new bundle format, we
are migrating to subcommands on the run command instead of increasing the
number of flags.

### Goals

Provide a command line that works with bundle image format.

### Non-Goals

This design does not cover the replacement of run --olm with run packagemanifest.

## Proposal

### Implementation Details/Notes/Constraints [optional]

The operator-sdk currently supports running an operator with OLM using the
`run` command with the `--olm` flag. The existing command only supports the
package manifest format. This enhancement will add a new subcommand, `bundle`,
to the `run` command. This new command, `run bundle`, will support the new
bundle format. More information about the different formats can be found at
the URLs below:

 * Operator bundles: https://github.com/operator-framework/operator-registry/blob/master/docs/design/operator-bundle.md
 * Package manifests format: https://github.com/operator-framework/operator-registry/tree/v1.5.3/#manifest-format

The idea is to consolidate the run command into different subcommands to make
things easier to understand and eliminate flag bloat.

 * run local - exists today
 * run packagemanifest - covered by OSDK-1126 (https://issues.redhat.com/browse/OSDK-1126)
 * run bundle - this design docs focus

The new command line looks as follows:

```
operator-sdk run bundle --bundle=<imageref> [--index-image=]
```

What happens when the above is invoked?

 1. If `--index-image` is not provided, assign variable `registryImage`
    equal to `quay.io/operator-framework/upstream-registry-builder`
 1. If `--index-image` is provided, assign variable `registryImage`
    equal to provided value
 1. Create deployment (or maybe just pod?) using registryImage, set
    container command to:
    ```
    /bin/bash -c ‘ \
      /bin/mkdir -p /database && \
      /bin/opm registry add   -d /database/index.db -b {.BundleImage} && \
      /bin/opm registry serve -d /database/index.db’
    ```
 1. If deployment/pod fails, clean it up, and error out with message of
    unsupported index image
 1. Create Service for deployment on port 50051
 1. Create GRPC catalogsource, pointing to service:50051
 1. Create OperatorGroup
 1. Create Subscription
 1. Verify operator is installed.
 1. Exit with success (leaving operator running)

 Like the existing run command, we *assume* the following:

 1. OLM is already installed
 1. If an index image is provided, we assume that it has:
    a. bash, so we can change the entrypoint
    b. dependencies required for `opm registry add` so we can inject
       the bundle into the DB
    c. the existing registry DB located at `/database/index.db`

#### Prototype Links

 * Examples setup script
   https://gist.github.com/joelanford/f7a7d89977638547b410db3eafc35a62
 * Modified upstream-opm-builder
   https://github.com/joelanford/operator-registry/blob/9f8b6b19709a333ede683b89ac0fda1abf9b14b8/upstream-opm-builder.Dockerfile#L16-L17
 * Bundle runner Helm chart
   https://github.com/joelanford/bundle-runner
 * Static validation sync
   https://docs.google.com/document/d/1Lf9foKtObj1ACk67a_0BNn7rupWAHEejZ6vynJggBCw/edit#
 * Bundle format
   https://github.com/operator-framework/operator-registry/blob/master/docs/design/operator-bundle.md

### Risks and Mitigations

If we do not get the modified upstream-opm-builder referenced above the
following assumptions will not work:

  1. if image doesn't have bash, we can't modify the entrypoint
  1. if opm is building index images without bash, then there will be no index
     images we support

## Design Details

### Test Plan

  * add tests for `internal/olm/operator/operator.go`
  * investigate adding e2e tests for `run bundle`, there are currently no e2e
    tests for `run --olm`
  * manual testing of `run bundle` will be done as well

### Graduation Criteria

The `run bundle` command will come in as non-alpha and be ready to use once
merged.

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

N/A

## Implementation History

20200507 - Initial proposal to add `run bundle` subcommand

## Drawbacks

N/A

## Alternatives

N/A

## Infrastructure Needed [optional]

N/A
