---
title: sdk-external-and-pluggable-validations
authors:
  - "@jmrodri"
reviewers:
  - "@camilamacedo86"
  - "@joelanford"
  - "@bparees"
approvers:
  - "@camilamacedo86"
  - "@joelanford"
creation-date: 2021-10-01
last-updated: 2021-10-01
status: implementable
see-also:
  - "/enhancements/this-other-neat-thing.md"
---

# SDK External and Pluggable Validations

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [X] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

 1. what validations do we have today?
 1. can we use scorecard to replace all of these validations?
    * scorecard uses the cluster to run the tests, these validations typically
      run locally or in a pipeline. They are also done before the expensive
      operator tests are run.
 1. what would it take to convert an existing validator to an executable format?

## Summary

Today, validations used by CVP are compiled into the operator-sdk. Any changes
to the validation rules requires a release of operator-framework/api followed by
a release of operator-sdk. This proposal attempts to design a way where the
validations can be hosted in their own repos as well as updated without
requiring new releases of the operator-sdk.

## Motivation

Every time the business changes validation rules, it requires an update to the
operator-framework/api library. A release of said library, then it needs to get
included into the operator-sdk. Then a release of the SDK needs to be cut in
order for the new validation rule change to be usable.

This process slows down the business by having to wait for this release process
to occur for what could be a very small rule change. It also causes downstream
rules to require an immediate upstream operator-sdk release, which may
otherwise be as far as 3 weeks away. It is difficult to explain to the community
why we need a new release for a CVP need.

Having the validators external to the SDK will allow for vendor specific
validators to be created allowing for greater flexibility.

### Goals

* allow validations to be updated when the business needs them to be
* allow validations to release when the authors need them to be
* do not require newer builds of the operator-sdk to get updated rules
* allow validations to be hosted in their own repos
* allow validations to be external to operator-sdk

TODO: are there any other goals?

### Non-Goals

* existing validations do NOT have be migrated unless they want to

## Proposal

### User Stories [optional]

#### Story 1
* as a CVP admin I would like to change the validation for XXX and have it
  useable as soon as I can do a validator release.

#### Story 2
* as a validation author, I can write a validation for a bundle and use it
  without having to get a new operator-sdk release

TODO: Stories from the epic; consolidate with the above stories

#### Story 3
* Users want to edit/add/remove a set of validation rules for Operator bundle validation.

#### Story 4
* Users can host a set of validation rules either locally or in remote.

#### Story 5
* Users can define the target set of validation rules for SDK's bundle validation command (either from local or remote source).

### Implementation Details/Notes/Constraints [optional]

There are a few alternatives, but I think wrapping the validations in their
own executables makes sense. It aligns with the work we've done with
[Phase 2][phase-2] plugins. It allows the most flexibility in terms of
implementation. It also has a simple API -
input: bundle dir; output: ManifestResult JSON.

Wrapping validations in their own executable simply means that there
needs to be something that the operator-sdk can run like a shell script
or a binary.

The validation executable will need to accept a single input value which
is the bundle directory. The output will need to be a ManifestResult JSON. This
JSON will be parsed by the operator-sdk converted and output as a Result.

For example, the existing validations could easily be wrapped with a `main.go`
file and compiled into a binary. They could be copied to their own repos for
easier release. Now you might be thinking that this means we'll have many
releases still, we actually would only have one (1) release which is the
validator itself. For go based validators this might mean a new binary, but you
could also release your validator in python and make it a single file or even a
shell script.

From the operator-sdk's point of view, we don't care what you use
to create your validator only that we can run it and pass it a bundle directory.


TODO: how will validators be found?

#### Validators

##### Running validators

The [existing validators](#default-validators-run-by-operator-sdk) operate
on a bundle. Each validator should be some executable that accepts a
single argument, the bundle root directory. So `operator-sdk` will pass them
the bundle directory. It will be the validator's responsibility to parse
the bundle to get what it needs.

For example, the validator executable will be invoked from the
`operator-sdk` in the following manner:

```
/path/to/validator/executable /path/to/bundle
```

A concrete example might look like this:

```
./validator-poc /home/user/dev/gatekeeper-operator/bundle
```

The actual implementation is up to the author. Here we have an example of
the [`OperatorHubValidator`][validator-poc1] as a Go binary.

##### Validator results

As stated earlier, each validator should be some executable that accepts a
single argument, the bundle root directory. Th validators should also output
ManifestResult JSON to stdout.

Because the [existing validators](#default-validators-run-by-operator-sdk)
currently return a [ManifestResult][manifest-result], it seems logical that we
use the same object as JSON for the output.

The validator executable should exit non-zero ONLY if the entrypoint failed to
run NOT if the bundle validation fails.

For example, let's say the validator is given a path to the gatekeeper bundle.
The validator should validate the given bundle and output the ManifestResult JSON.
Here is an example of a run:

```json
{
    "Name": "gatekeeper-operator.v0.2.0-rc.3",
    "Errors": null,
    "Warnings": [
        {
            "Type": "CSVFileNotValid",
            "Level": "Warning",
            "Field": "",
            "BadValue": "",
            "Detail": "(gatekeeper-operator.v0.2.0-rc.3) csv.Spec.minKubeVersion is not informed. It is recommended you provide this information. Otherwise, it would mean that your operator project can be distributed and installed in any cluster version available, which is not necessarily the case for all projects."
        }
    ]
}
```

The above JSON will be read by the `operator-sdk` during `bundle validate`
command and output the results as it does today. The example below shows what
`operator-sdk bundle validate` would printout if given the ManifestResult from
above.

```json
{
    "passed": true,
    "outputs": [
        {
            "type": "warning",
            "message": "Warning: Value : (gatekeeper-operator.v0.2.0-rc.3) csv.Spec.minKubeVersion is not informed. It is recommended you provide this information. Otherwise, it would mean that your operator project can be distributed and installed in any cluster version available, which is not necessarily the case for all projects."
        }
    ]
}
```

Allowing the validators to output ManifestResult should make it easier to
transition existing validators to the external format with minimal code.

##### Migrating existing validator to executable

In the short to near term, you create a new `main.go` for each of the validators.
Then import the validator code from operator-framework/api. The `main.go` would
take in one (1) argument, the bundle directory.

Since the [existing validators](#default-validators-run-by-operator-sdk) already
output `ManifestResult`, it's easiest if we simply print that out as JSON to
stdout.

An example POC that takes an existing validator and outputs `ManifestResult` can
be found at [validator-poc][validator-poc1]

Another example of a migration can be find at [ocp-olm-catalog-validator][camila-poc].
This particular example does NOT yet output `ManifestResult` format.

TODO: do we need to add `json` tags to these structs?
https://github.com/operator-framework/api/blob/master/pkg/validation/errors/error.go#L9-L16


#### CLI

The big question at hand is how do we indicate to the `operator-sdk` CLI that
we want to run an external validator? Today, the `bundle validate` command takes
in a few flags, here we will discuss how each might need to be changed to work
with external validators.

```
Usage:
  operator-sdk bundle validate [flags]

...

Flags:
  -h, --help                                                 help for validate
  -b, --image-builder string                                 Tool to pull and unpack bundle images. Only used when validating a bundle image. One of: [docker, podman, none] (default "docker")
      --list-optional                                        List all optional validators available. When set, no validators will be run
      --optional-values --optional-values=k8s-version=1.22   Inform a []string map of key=values which can be used by the validator. e.g. to check the operator bundle against an Kubernetes version that it is intended to be distributed use --optional-values=k8s-version=1.22 (default [])
  -o, --output string                                        Result format for results. One of: [text, json-alpha1]. Note: output format types containing "alphaX" are subject to change and not covered by guarantees of stable APIs. (default "text")
      --select-optional string                               Label selector to select optional validators to run. Run this command with '--list-optional' to list available optional validators

Global Flags:
      --plugins strings   plugin keys to be used for this subcommand execution
      --verbose           Enable verbose logging
```

* *--help* works as is
* *--image-builder* works as is
* *--list-optional* would need to be updated to locate external validators.

  TODO: How do we locate these validators? Do we use the XDG_CONFIG like Phase
  2? Or allow users to pass in the location of the validators?

* *--optional-values* would continue to be passed to the validators
* *--output* indicates how we want to output the results.

  TODO: what types do we want to support?

* *--select-optional* takes in a label selector to specify which optional test
  to run. We could allow a new label to indicate the location of the executable
  to run as the validator. i.e. `validator-path=/path/to/validator/the-executable`

TODO: need to define what the CLI for operator-sdk will look like. Are there
any new flags? Do we use the environment variable?

NOTE: what would the CLI look like?


##### Default validators run by operator-sdk

List of current validators:
TODO: do we need these validators called out like this?

* BundleValidator
  * validates the bundle
  * looks for duplicate keys in bundle
  * verifies all owned keys match a CRD in the bundle
  * verifies that all CRDs present in the bundle are in the CSV
  * validates the service account

* ClusterServiceVersionValidator
  * operator-sdk checks to see if the CSV is nil on the bundle (that's the first
    check)
  * checks that the CSV name is a valid format
    * DNS1123
    * valid label
  * verifies replaces name is also a valid format
    * DNS1123
    * valid label
  * ensures that both `alm-examples` and `olm.examples` are not both present
  * decodes the example yaml, to validate its format
  * checks provided APIs
  * validates the `InstallModes`
    * verifies that conversion CRDs support `AllNamespaces`
  * checks for missing mandatory fields
  * validates the annotation names
  * validates the version and kind

* CustomResourceDefinitionValidator
  * operator-sdk puts the v1beta1 and v1 CRDs together for validation
  * validates v1beta1 CRDs
  * validates v1 CRDs
  * validates internal CRDs
    * https://github.com/kubernetes/apiextensions-apiserver/blob/master/pkg/apis/apiextensions/validation/validation.go#L49-L78

##### Optional validators

* PackageManifestValidator

* OperatorHubValidator

* ObjectValidator

* OperatorGroupValidator

* CommunityOperatorValidator

* AlphaDeprecatedAPIsValidator

* AllValidators
  * Uses ALL of the validators
    * BundleValidator
    * ClusterServiceVersionValidator
    * CustomResourceDefinitionValidator
    * PackageManifestValidator
    * OperatorHubValidator
    * ObjectValidator
    * OperatorGroupValidator
    * CommunityOperatorValidator
    * AlphaDeprecatedAPIsValidator

* DefaultBundleValidators
  * Uses the following 3 validators
    * BundleValidator
    * ClusterServiceVersionValidator
    * CustomResourceDefinitionValidator
### Risks and Mitigations

There is little risk, if this does not pan out we keep going on the current path
of compiling them in.

## Design Details

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:
- Maturity levels - `Dev Preview`, `Tech Preview`, `GA`
- Deprecation

Clearly define what graduation means.

#### Examples

These are generalized examples to consider, in addition to the aforementioned
[maturity levels][maturity-levels].

##### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers

##### Tech Preview -> GA 

- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

##### Removing a deprecated feature

- Announce deprecation and support policy of the existing feature
- Deprecate the feature

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

* The version of the validators can change however the validator author sees fit.
* The API or contract between `operator-sdk` and validators will be
  * input to validator: bundle directory
  * output from validator: `ManifestResult` JSON

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

Validators would have to be their own executables which could result in a
compilation step being needed depending on the language used to implement them.

## Alternatives

* put validations in their own images
  * need to define "API" contract what is the entrypoint and what parameters do
    we give it
  * pro:
    * already have a precendence for running things from images
    * familiar tech
  * con:
    * would need a cluster to run these validations
    * authors would have to create binaries of their validations anyway

TODO: ---------- vvvvvv CUT vvvvvv ----------
* wrap validations in their own executables
  * validations would be wrapped in an executable that would be called from
    operator-sdk
  * pro:
    * could reuse the existing go validations and put a main.go in front to make
      them executable
    * follows a similar phase 2 plugin path
    * could reuse some of the phase 2 tech to run this
    * would not need a cluster necessarily to run the validation
  * con:
    * authors would have to create binaries of their validations
TODO: ---------- ^^^^^^ CUT ^^^^^^ ----------

* use a language like JavaScript or CUE to define all validations
  * validations could be run from a git repo, i.e. operator-sdk could pull it
    and then evaluate it
  * pro:
    * simpler delivery, expose via a gitrepo and done
  * con:
    * all existing validations would have to be re-written in a new language
      structure which could introduce new bugs
    * unproven technology
    * would have to write the engine to know how to execute these

* use scorecard to do the validations
  * create validations written in scorecard as custom tests
  * pro:
    * infrastructure required to run is already built within scorecard
  * con:
    * would need a cluster to run these validations

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources
started right away.

https://github.com/operator-framework/api/tree/master/pkg/validation

[phase-2]: https://github.com/kubernetes-sigs/kubebuilder/blob/master/designs/extensible-cli-and-scaffolding-plugins-phase-2.md
[manifest-result]: https://github.com/operator-framework/api/blob/master/pkg/validation/errors/error.go#L9-L16
[validator-poc1]: https://github.com/jmrodri/validator-poc/tree/poc1-manifestresults
[camila-poc]: https://github.com/camilamacedo86/ocp-olm-catalog-validator
