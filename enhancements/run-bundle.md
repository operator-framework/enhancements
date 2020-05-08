---
title: run-bundle
authors:
  - "@jmrodri"
reviewers:
  - "@joelanford"
  - "@estroz"
  - "@kevinrizza"
  - "@ecordell"
  - "@shawn-hurley"
approvers:
  - "@joelanford"
  - "@estroz"
  - "@ecordell"
  - "@shawn-hurley"
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

## Open Questions

1. We currently check OLM installation status with `operator-sdk olm status`.
   Can we change that to using the discovery API?
   * Based on discussions, using the discovery API is the way to go.

2. How do we handle --mode= flag on registry add
   * should we ALWAYS default to --mode=semver?
     * this has less testing and isnot yet the default. We should stick with
        the default.
   * should we have a flag that allows our default mode handling to be overridden?
     * no

## Summary

Enhance the Operator SDK `run` command to support the new bundle format.

## Motivation

One of the primary goals of the Operator SDK is to make it easy for operator
developers to run and test their operators with Operator Lifecycle
Manager (OLM). There currently exists a `run --olm` command which supports the
packagemanifest format. In order to add support for the new bundle format, we
are migrating to subcommands on the run command instead of increasing the
number of flags.

### Goals

Provide a command line that works with bundle image format.

### Non-Goals

This design does not cover the replacement of run --olm with run
packagemanifest. The change to `run packagemanifest` is covered by
[OSDK-1126](https://issues.redhat.com/browse/OSDK-1126).

## Proposal

### Implementation Details/Notes/Constraints [optional]

The Operator SDK currently supports running an operator with OLM using the
`run` command with the `--olm` flag. The existing command only supports the
package manifest format. This enhancement will add a new subcommand, `bundle`,
to the `run` command. This new command, `run bundle`, will support the new
bundle format. More information about the different formats can be found at
the URLs below:

 * Operator bundles: https://github.com/operator-framework/operator-registry/blob/master/docs/design/operator-bundle.md
 * Package manifests format: https://github.com/operator-framework/operator-registry/tree/v1.5.3/#manifest-format

The idea is to consolidate the run command into different subcommands to make
things easier to understand and eliminate flag bloat.

 * `run local` - exists today
 * `run packagemanifest` - covered by [OSDK-1126](https://issues.redhat.com/browse/OSDK-1126)
 * `run bundle` - covered by this design document

#### Command Line Interface
The new command line will look as follows:

```
operator-sdk run bundle <bundle-image> [--index-image=] [--namespace=] [--install-mode=(AllNamespace|OwnNamespace|SingleNamespace=)]
```

* `<bundle-image>` is a positional argument that specifies the bundle image.
* `[--index-image=]` is an optional flag that specifies the index image.
  * If `--index-image` is not provided, default to
    `quay.io/operator-framework/upstream-registry-builder`
  * If `--index-image` is provided, use the provided value
* `[--namespace=]` is an optional flag that specifies the namespace to install
  the operator. It will default to the `KUBECONFIG` context.
* `[--install-mode=]` is an optional flag that specifies the
  [OLM install mode](https://red.ht/2YK0ksD).
  * We support 3 modes:
    * AllNamespaces
    * OwnNamspace
    * SingleNamespace=ns1
      * NOTE: if ns1 is the same as the operator's namespace, a warning will be
        printed
  * When `--install-mode` is not supplied, the default precendence order is:
    * `OperatorGroup`, if it already exists
    * `AllNamespaces`, if the CSV supports it
    * `OwnNamespace`, if the CSV supports it
    * If the CSV does not support either of these, `--install-mode` flag
      will be **required**

#### Assumptions
Like the existing `run` command, we **assume** the following:

 1. OLM is already installed
 1. kube API is available with the permissions to create resources
 1. kube server can see and pull the provided images (i.e. bundle and index)
 1. If an index image is provided, we assume that it has:
    * bash, so we can change the entrypoint
    * dependencies required for `opm registry add` so we can inject
       the bundle into the DB
    * the existing registry DB located at `/database/index.db`

#### Example UX Scenarios

Running the following scenarios results in different results. While not an
exhaustive list, these are some example uses of the `run bundle` command.

* `operator-sdk run bundle <bundle-image>` would install into the default kube namespace,
creates a cluster-wide `OperatorGroup`.
* `operator-sdk run bundle <bundle-image> --namespace foo` would install into
  namespace `foo`, creates a cluster-wide `OperatorGroup`
* `operator-sdka run bundle <bundle-image> --namespace foo --install-mode
  SingleNamespace=bar` would install into namespace foo, would determine if
  OperatorGroup exists and is sufficient, if not it would error
* `operator-sdk run bundle <bundle-image> --install-mode SingleNamespace=bar`
  would install into the default kube namespace, does the OperatorGroup logic

#### Simple Example

Let's take the simplest example from above.

```
operator-sdk run bundle <bundle-image>
```

What happens when the above is invoked?

 1. Since `--index-image` was not provided, assign variable `registryImage`
    equal to `quay.io/operator-framework/upstream-registry-builder`
 1. Create pod using `registryImage`, set container command (entrypoint) to:
    ```
    /bin/bash -c ‘ \
      /bin/mkdir -p /database && \
      /bin/opm registry add   -d /database/index.db -b {.BundleImage} && \
      /bin/opm registry serve -d /database/index.db’
    ```
 1. If pod fails, clean it up, and error out with message of unsupported
    index image
 1. Create GRPC CatalogSource, pointing to <pod-name>.<namespace>:50051
 1. Create OperatorGroup, only if it does not exist, since `--install-mode` was
    not supplied, defaults to `AllNamespaces`.
 1. Create Subscription
 1. Operator installed into the default namespace defined by KUBECONFIG
 1. Verify operator is installed.
 1. Exit with success (leaving operator running)

#### Complex Example

Now let's look at one of the more complex examples:

```
operator-sdk run bundle <bundle-image> --namespace foo --install-mode SingleNamespace=bar
```

 1. Since `--index-image` was not provided, assign variable `registryImage`
    equal to `quay.io/operator-framework/upstream-registry-builder`
 1. Create pod using `registryImage`, set container command (entrypoint) to:
    ```
    /bin/bash -c ‘ \
      /bin/mkdir -p /database && \
      /bin/opm registry add   -d /database/index.db -b {.BundleImage} && \
      /bin/opm registry serve -d /database/index.db’
    ```
 1. If pod fails, clean it up, and error out with message of unsupported
    index image
 1. Create GRPC CatalogSource, pointing to <pod-name>.<namespace>:50051
 1. Creates a `SingleNamespace` OperatorGroup watching namespace `bar`.
 1. Create Subscription
 1. Operator installed into the default namespace `foo`
 1. Verify operator is installed.
 1. Exit with success (leaving operator running)

#### Prototype Links

 * PR upstream-opm-builder.Dockerfile: add bash and ca-certificates
   https://github.com/operator-framework/operator-registry/pull/320
 * Examples setup script:
   https://gist.github.com/joelanford/f7a7d89977638547b410db3eafc35a62
 * Modified upstream-opm-builder:
   https://github.com/joelanford/operator-registry/blob/9f8b6b19709a333ede683b89ac0fda1abf9b14b8/upstream-opm-builder.Dockerfile#L16-L17
 * Bundle runner Helm chart:
   https://github.com/joelanford/bundle-runner
 * Static validation sync:
   https://docs.google.com/document/d/1Lf9foKtObj1ACk67a_0BNn7rupWAHEejZ6vynJggBCw/edit#
 * Bundle format:
   https://github.com/operator-framework/operator-registry/blob/master/docs/design/operator-bundle.md

### Risks and Mitigations

If we do not get the modified
[upstream-opm-builder](https://github.com/operator-framework/operator-registry/blob/master/docs/design/operator-bundle.md)
referenced above the following assumptions will not work:

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

### Version Skew Strategy

N/A

## Implementation History

20200508 - Do not wrap CLI command in document
20200508 - Updated questions, clarified CLI flags, defined UX, detailed examples
20200507 - Initial proposal to add `run bundle` subcommand

## Drawbacks

N/A

## Alternatives

N/A

## Infrastructure Needed [optional]

N/A
