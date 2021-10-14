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

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

 1. what validations do we have today?
 1. do these validations run against a cluster?
 1. can we use scorecard to replace all of these validations?
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
rules to require an immediate upstream operator-sdk release which may be as far
as 3 weeks away.

### Goals

List the specific goals of the proposal. How will we know that this has succeeded?

* allow validations to be updated when the business needs them to be
* do not require newer builds of the operator-sdk to get updates
* allow validations to be hosted in their own repos
* allow validations to be run external to operator-sdk

### Non-Goals

TODO:
What is out of scope for this proposal? Listing non-goals helps to focus discussion
and make progress.

## Proposal

### User Stories [optional]

Detail the things that people will be able to do if this is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.

#### Story 1
* as a CVP admin I would like to change the validation for XXX and have it
  useable right now.

#### Story 2
* as a user

### Implementation Details/Notes/Constraints [optional]

There are a few alternatives, I think putting the validations in their own
images similar to how bundles are done makes sense. It is a known and easy
transport.

NOTE: if it's in an image is it just a data container or is it an entrypoint
that needs to run in a cluster?

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


List of current validators:

##### Default validators run by operator-sdk

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

The current validators all return a
[ManifestResults](https://github.com/operator-framework/api/blob/master/pkg/validation/errors/error.go#L9-L16)

Each validator should have an "entrypoint", some executable that accepts a
single argument, the bundle root.

For example, the validator binary will be invoked from the `operator-sdk` in the
following manner:

```
./validator-poc ~/dev/gatekeeper-operator/bundle
```

It should write its output to stdout to be parsed by operator-sdk. Output should
be JSON.

Exit non-zero ONLY if the entrypoint failed to run NOT if the bundle validation
fails.

Output

Result format used by the `bundle validate` today.

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

#### Migrating existing validator to executable

In the short to near term, you create a new main.go for each of the validators.
Then import the validator code from o-f/api. The main.go would take in 1
argument which is the bundle directory.

TODO: jmrodri determine the output format
The output would be a Result json. Unless we want the more complicated
ManifestResults

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

How will the component handle version skew with other components?
What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- During an upgrade, we will always have skew among components, how will this impact your work?
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI, CRI
  or CNI may require updating that component before the kubelet.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

* put validations in their own images
  * need to define "API" contract what is the entrypoint and what parameters do
    we give it
  * pro:
    * already have a precendence for running things from images
    * familiar tech
    * could run locally and simply use the image as a data transport
  * con:
    * would need a cluster to run these validations
    * authors would have to create binaries of their validations

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

* use a language like JavaScript or CUE to define all validations
  * validations could be run from a git repo, i.e. operator-sdk could pull it
    and then evaluate it
  * pro:
    * simpler delivery, expose via a gitrepo and done
  * con:
    * all existing validations would have to be re-written in new language
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
