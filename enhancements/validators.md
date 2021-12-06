---
title: Standard definition for the Validators (linters)
authors:
- "@camilamacedo86"
  reviewers:
- TBD
  approvers:
- TBD
  creation-date: 2021-12-02
  last-updated: 2021-12-02
  status:implemented
  see-also:
- https://github.com/operator-framework/enhancements/pull/98
---

# Standard definition for external validators

## Release Signoff Checklist

- [x] Enhancement is `implemented`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

The Validators are static checks (linters) that can scan the manifests and provide
with low-cost valuable results to ensure the quality of the package of distributions(bundle or package formats) which will be distributed via OLM,
to ensure that they respect some specific criteria for vendors and/or catalogs.

For example, Openshift
catalog it is composed of more than one image ([More info](https://docs.openshift.com/container-platform/4.6/operators/admin/olm-managing-custom-catalogs.html)). To publish via each image is required to respect some specific criteria
such as to be part of the vendor catalog.

This proposal aims to provide a standard definition to reduce the complexity of keeping these rules
maintained and consumed.

## Motivation

- Reduces the effort and complexity to keep the validations and rules maintained
- Bring visibility and clarity over what common criteria are for distributing via OLM, for publishing to OpenShift or for publishing via certain Index catalogs.
- Defining a standard definition leveraging the [operator-framework/API][of-api] without the need to re-implement and to keep maintaining what is in common.
- A clear strategy definition about what can be introduced and supported in [operator-framework/API][of-api]
- A standard definition that brings flexibility and allows leveraging what exists and is already in use.
### Goals

- [operator-framework/API][of-api] providing and defining what is in common use for any validator such as:
  - possible output formats
  - common criteria, rules and suggestions to distribute Operators via OLM under Operator Framework, which are not a vendor and/or catalog specific
  - validators interface
  -  a standard format for the results of the tests
-  Ability to use the [operator-framework/API][of-api] to build external validators, which are vendor or Index Catalog specific
- Ability to consume various validators from different maintainers and sources by working with a single common result format
- Ability to use the Validators via CLI tools (such as SDK and OPM) or any project by importing as a dependency or indeed address the common need to call them in Continuous Integration (CI) process
- Remove the need to have vendor or catalog specific validators in-tree

### Non-Goals

- Amending the Operator SDK so that it works with external validators. (_It is the scope of another PE, see [here](https://github.com/operator-framework/enhancements/pull/98)_)
- New options to output the results such as JUnit or changes in the operator-framework/api interface definitions that might be beneficial in long term
- Changes or improvements to the current interfaces, functionalities or validators

## Proposal

The goal of this proposal is to ensure that we agree on a defined standard concept and vision
which can help us in the medium/long term with improving maintainability, usability and readability.
 leveraging what we have already implemented.

**Domain of Responsibilities**

[operator-framework/API][of-api]:
- providing a common framework for internal and external validators
- providing common validators (which are useful and required for any solution which will be distributed via OLM)

External Validators:
- location for Index Catalog or vendor-specific validators
- implementation of catalog or vendor-specific validation logic
- Extending [operator-framework/API][of-api] and using it as a lib to ensure a standard for its consumers

### Implementation Details/Notes/Constraints

[operator-framework/API][of-api]:
- Interfaces: https://github.com/operator-framework/api/blob/v0.10.7/pkg/validation/interfaces/validator.go
- Implementation of the interfaces for the validations: https://github.com/operator-framework/api/blob/v0.10.7/pkg/validation/validation.go
- Structure for results (`ManifestResults`): https://github.com/operator-framework/api/blob/v0.10.7/pkg/validation/errors/error.go

**External validators:**
- To ensure the standard criteria to publish in Openshift catalog: https://github.com/redhat-openshift-ecosystem/ocp-olm-catalog-validator
- To ensure specific criteria to publish in the OperatorHub image/catalog: https://github.com/k8s-operatorhub/bundle-validator

### Risks and Mitigations

The same risk of any conventional or standard not being adopted or getting obsolete.

However, for [operator-framework/API][of-api]
the risk is minimal since the ideas/solutions proposed here are implemented and in use already.

## Design Details

### Test Plan

Any criteria ought to be ensured with unit tests
under the projects which owns the rules.

All validators implemented in the [operator-framework/API][of-api] or the external ones added here
are covered with the unit tests.

#### Examples

**Examples of usage over standard criteria via SDK:**

To check a bundle to be distributed via OLM:

a) Using Operator Framework suite to check the bundle against all possible common validators:

```
$ operator-sdk bundle validate ./bundle --select-optional suite=operatorframework
```

b) Calling option validators to ensure the specific purposes, such to provide nice features
like reports and comments in PRs to advise its users, e.g.:

```
$ operator-sdk bundle validate ./bundle --select-optional name=best-practices
```

**Examples of usage over common criteria by extending operator-framework/api:**

```go
import (
   ...
    apimanifests "github.com/operator-framework/api/pkg/manifests"
    apivalidation "github.com/operator-framework/api/pkg/validation"
    "github.com/operator-framework/api/pkg/validation/errors"
   ...
)

// Load the bundle
bundle, err := apimanifests.GetBundleFromDir(path)
if err != nil {
   ...
   return nil
}

// Call all default validators and the AlphaDeprecateApi optional one
validators := apivalidation.DefaultBundleValidators
validators = validators.WithValidators(apivalidation.AlphaDeprecatedAPIsValidator)

objs := bundle.ObjectsToValidate()

results := validators.Validate(objs...)
nonEmptyResults := []errors.ManifestResult{}

for _, result := range results {
    if result.HasError() || result.HasWarn() {
        nonEmptyResults = append(nonEmptyResults, result)
    }
}
// return the results
return nonEmptyResults
```

**Examples of usage with external validators:**

If we have the intention to publish on Openshift (like as CVP, RedHat Community and all images used by default to build the OCP catalog):

Pre-requirement: have the SDK and [OCP external validator][ocp-validator] binaries locally

1) Ensure the common criteria:

```
$ operator-sdk bundle validate ./bundle --select-optional suite=operatorframework --optional-values=k8s-version=<k8s-version>
```

2) Ensure the OCP specific criteria:

```
$ ocp-olm-catalog-validator <bundle-path> --optional-values="range==<OCP labels>" --output=json-alpha1
```

##### Dev Preview -> Tech Preview

N/A

##### Tech Preview -> GA

N/A

##### Removing a deprecated feature

Following the details over move from operator-framework/api and SDK specific vendor/catalog checks
to the external validator.

**Announce deprecation to move the specific checks for the external validator:**
- SDK: PR: https://github.com/operator-framework/operator-sdk/pull/5414
- Operator-Framework/api: PR: https://github.com/operator-framework/api/pull/199

**Plan for removal**
- Issue: https://github.com/operator-framework/operator-sdk/issues/5422

### Upgrade / Downgrade Strategy

Validators consumers must only bump the dependency tags versions or define what binary version
ought to use to upgrade or downgrade.

### Version Skew Strategy

External validators ought to have implemented a GitHub Action to perform releases automatically and with low effort
such as it was implemented for [ocp olm catalog validator][ocp-validator], see: [here](https://github.com/redhat-openshift-ecosystem/ocp-olm-catalog-validator/blob/main/.github/workflows/release.yaml).

## Implementation History

- All tests executed by [Operator Courier](https://github.com/operator-framework/operator-courier) were moved to the [OperatorHubValidator](https://github.com/operator-framework/api/blob/master/pkg/validation/internal/operatorhub.go)

## Drawbacks

OLM accepts contributions to implement and ship any possible checks indeed vendor catalog specific in
operator-framework/api and SDK call all them via its options selectors and suites.
That would remove the need for external validators such as [ocp olm catalog validator][ocp-validator].

However, it shows goes against the desires over what ought to be kept and maintained upstream since it would
mean allowing contributions with specific checks that might only be valid for particular vendors or catalogs.

## Alternatives

operator-framework/api accepting mainly any specific criteria and SDK shipping all via the CLI and have then
categorized via new suites, such as:

```
# To check the bundle against Openshift specific criteria
$ operator-sdk bundle validate ./bundle --select-optional suite=openshift

# To check the bundle against RedHatConnect specific criteria
$ operator-sdk bundle validate ./bundle --select-optional suite=redhat-connect

# To check the bundle against K8s Community Operator specific criteria
$ operator-sdk bundle validate ./bundle --select-optional suite=k8s-community
```

**pros**
- Any rule/criteria available centralize in one project
- All options and validators are available and shipped with SDK binary by default
- No need for any additional binaries or projects
- Operator Framework users are able to check all options easily
- Easy to ensure that the inputs and output for any possible acceptable validator
  will have the same input and output format/options

**cons**
- Have in upstream implemented and supported vendor and catalog specific criteria
- SDK needs still control and ship the changes, which might cause some disturbing with asks to push
  new releases outside of its calendar to address some bug fix not to block users distributions
- officially support vendor and catalog specific criteria in upstream (SDK and operator-framework/api)

[of-api]: https://github.com/operator-framework/api
[ocp-validator]: https://github.com/redhat-openshift-ecosystem/ocp-olm-catalog-validator
