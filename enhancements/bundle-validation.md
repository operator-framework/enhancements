---
title: bundle-validation
authors:
  - "@bowenislandsong"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-03-09
last-updated: 2020-03-09
status: implementable
---

# bundle-validation

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

This enhancement proposes extending the existing [operator-framework/api](https://github.com/operator-framework/api) validation library to include support for static analysis of individual
[Operator bundles](https://github.com/openshift/enhancements/blob/master/enhancements/olm/operator-bundle.md).
This library could be imported by tools interested in validating Operator bundles, such as [scorecard](https://github.com/operator-framework/operator-sdk/blob/master/doc/test-framework/scorecard.md) and [opm](https://github.com/operator-framework/operator-registry/blob/master/docs/design/opm-tooling.md), providing users with consistent definitions of valid bundles.

The library provided as a part of this enhancement will include a core set of tests that can be ran against any bundle. This library will also provide users with an easy way to extend these tests for pipeline specific validations.

## Motivation

The primary motivation for this enhancement is to provide a library that statically validates a [bundle](https://github.com/openshift/enhancements/blob/master/enhancements/olm/operator-bundle.md). This library can be used by external projects and tools, such as [Operator-SDK](https://github.com/operator-framework/operator-sdk), to perform a static check for the validity of a bundle before testing the operator on clusters.

Providing users with the tools to statically analyze bundles is a hard requirement from operator developers and pipeline owners due to the high cost of testing bundles on fresh clusters. Even though there is no replacement solution to testing operators on a running cluster, we would still like to provide static validations to help catch as many issues as possible. Currently, people rely on [operator-courier](https://github.com/operator-framework/operator-courier), a python project, for static analysis of bundle data. There are primarily two shortcomings that exist with the operator-courier project:

- **Issue 1:** The fact that the objects validated by the library are defined in OLM and written in Golang mismatches with the courier project language. The structs defined in go are updated frequently, causing courier to be out-of-sync and in need of constant updates.

- **Issue 2:**  Courier was designed to interact with [App-Registries](https://github.com/operator-framework/operator-courier/blob/master/docs/how-to-upload-artifact-deprecated.md), which are being replaced by [Catalog Images](https://docs.openshift.com/container-platform/4.3/operators/olm-restricted-networks.html#olm-understanding-operator-catalog-images_olm-restricted-networks).

The new validation library can address each of the aforementioned shortcomings by the following:

- **Issue 1:** The library will be written in Golang and maintained as part of [operator-framework/api](https://github.com/operator-framework/api). The bundle structs will be moved to _operator-framework/api_ and act as the source of truth for the structure of these objects by the [OLM](https://github.com/operator-framework/operator-lifecycle-manager), [Operator-SDK](https://github.com/operator-framework/operator-sdk), and this validation library. This change will synchronize the state of the bundle structure across all projects making it easier to maintain, import, and serialize.

- **Issue 2:** Rather than rewriting the courier project to provide static analysis for bundles provided as Catalog Images, the new library can be written and then imported by tools like _scorecard_ to perform validation.

### Goals

- Define a set of rules/validations so that bundles with their specific set of configurations can be validated by checking the required information and components.

### Non-Goals

- The validation compares different versions of bundles from the same package.
 
- The validation considers the validity of a bundle in the context of an index, which should be part of [operator-registry](https://github.com/operator-framework/operator-registry).

- The validation guarantees the successful installation of a bundle (an operator).

## Proposal

### User Stories

As an Operator developer or maintainer, I would like to discover my missing vital metadata and ensure that my Operator bundle is well-formed.

As a cluster administrator or release pipeline owner, I would want to be aware of malformed Operator bundles so that I can prevent them from being released into a catalog for cluster tenants to install.

As a community contributor to the bundle validation library, I would want to understand the workflow of contributing to the core validation strategies or customized validations as part of an extended package.

### Implementation Details/Notes/Constraints

- The bundle validation library should list out errors based on static scanning of a bundle directory and locate their root causes if applicable. 

- Much like project [flak8](https://pypi.org/project/flake8/) where a [list of validations](https://www.flake8rules.com/) can be individually enabled/disabled, individual bundle validating rule should also be separable. Ultimately, tools that are interested in importing the validation library should be able to cherry-pick validations used for different scenarios and configurations. 

- The library should provide all validations currently available within the operator-courier.

- The bundle validation library will be used to validate OLM/registry manifests and expose an interface to add validators for any scheme. 

- Errors and alerting messages should support both standardized human readable format and `JSON` output.

- The structure of the validation library needs to be well-organized and defined in a README.md to encourage community contribution.

- The library should allow customized validation strategies as part of an extended package.

- The library should allow grouping different sets of validations to support specific bundle types or targeted cluster settings, ie: "validations for Kube 1.18", etc.

- In the future, we want to support [CSVless bundle type](https://github.com/operator-framework/enhancements/pull/8/files) by both/either creating more validations for various media types or statically synthesizing CSVs from these bundles to validate with existing rules.

#### Initial set of validations

There exist validation strategies available in _operator-courier_ and _api_ repositories which would serve as an initial set of validations:

- Validate the metadata for every CRD that exists in a bundle.
    - E1: crd metadata not defined
    - E2: crd metadata.name not defined
    - E3: crd apiVersion not defined
    - E4: crd spec not defined
    - E5: crd spec.names not defined
    - E6: crd spec.names.kind not defined
    - E7: crd spec.names.plural not defined
    - E8: crd spec.group not defined
    - E9: crd spec.version pr spec.versions not defined
    - E10: error converting versioned crd to unversioned crd
    - E11: invalid CRD based on "k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/validation"


- Ensure the CSV and its metadata contain required fields, installable as deployments, and all its CRDs has specified owner(s).
    - E12: csv metadata not defined
    - E13: csv metadata.name not defined
    - E14: csv name must have format: {operator name}.(v)X.Y.Z
    - E15: csv name contains an invalid semver
    - W1: csv metadata.annotations item is not defined if is not ["categories", "description", "containerImage", "createdAt", "support"] **\***
    - W2: csv metadata.annotations.certified not defined **\***
    - E16: metadata.annotations.certified is not of type string **\***
    - W3: both alm-examples and olm.examples are present, checking only alm-examples **\***
    - E17: metadata.annotations.alm-examples contains invalid json string **\***
    - E18: csv.Spec.APIServiceDefinitions.Owned is not empty (example provided API)
    - W4: each example provided API should have an example annotation **\***
    - E19: csv apiVersion not defined
    - E20: csv spec not defined
    - E21: csv spec item is not defined if is not ["displayName", "description", "icon", "version", "provider", "maturity"] **\***
    - E22: csv spec.installModes not defined
    - E23: duplicate installModes present
    - E24: none of InstallModeTypes are supported
    - E25: csv spec.install not defined
    - E26: csv spec.install.strategy must be deployment
    - E27: csv spec.install.strategy not defined
    - E28: csv spec.install.spec not defined
    - E29: csv spec.install.spec.deployments not defined
    - E30: csv spec.install.spec.deployments should be a list
    - E31: csv spec.install.spec.permissions should be a list
    - E32: csv spec.install.spec.clusterPermissions should be a list
    - E33: csv spec.customresourcedefinitions.owned not defined
    - E34: csv spec.customresourcedefinitions.name not defined
    - E35: crd referenced in csv not defined in root list of crds
    - E36: owned CRD is present in bundle but not defined in CSV
    - E37: csv spec.customresourcedefinitions.kind not defined
    - E38: csv spec.customresourcedefinitions.version not defined
    - E39: CRD.spec.names.kind does not match CSV.spec.crd.owned.kind
    - E40: CSV.spec.crd.owned.version is not in CRD.spec.versions list
    - E41: CRD.spec.version does not match CSV.spec.crd.owned.version
    - E42: CRD.spec.names.plural and CRD.spec.group does not match CSV.spec.crd.owned.name


- Validating packages and manifests.
    - E43: Only 1 package is expected to exist per bundle
    - E44: packageName not defined
    - E45: no package channels defined
    - E46: default channel is empty but more than one channel exists
    - E47: package channel.name not defined
    - E48: duplicate package manifest channel name
    - E49: package channel.currentCSV not defined
    - E50: channel.currentCSV is not included in list of csvs
    - E51: The packageName in bundle does not match repository name
    - E52: duplicate CSV in bundle set
    - E53: error getting spec.replaces from bundle CSV
    - E54: spec.replaces is referenced by more than one CSV
    - E55: spec.replaces field cannot match its own metadata.name
    - E56: spec.replaces CSV is not present in manifests
    - E57: spec.skips cannot match spec.replaces
    - E58: metadata.annotations.skipPackageAnnotationKey range contains the current CSV's version
    - E59: metadata.annotations.skipPackageAnnotationKey is an invalid semantic version range
    - E60: spec.replaces references a parent in CSV replace chain which causes repalce cycle
    - E61: spec.replaces does not map to any CSV in bundles
    - E62: currentCSV for channel name not found in bundle

- Validating cluster service versions for operatorhub.io UI. **\***
    - W5: CRD does not have an example CR in alm-examples
    - W6: every owned CRD should have alm-examples
    - E63: csv.spec.provider should be a singleton list
    - E64: csv.spec.provider element should contain name as a single field
    - E65: csv.spec.maintainers element should contain both name and email
    - E66: csv.spec.maintainers element email is not valid
    - E67: csv.spec.maintainers must be a list of name & email pairs
    - E68: csv.spec.links element should contain both name and url
    - E69: csv.spec.links element url is not valid
    - E70: csv.spec.links must be a list of name & url pairs
    - E71: spec.version is not a valid semver
    - E72: metadata.annotations.capabilities is not a valid capabilities level if it is not ["Basic Install", "Seamless Upgrades", "Full Lifecycle", "Deep Insights", "Auto Pilot"]
    - E73: spec.icon.mediatype %s is not a valid mediatype if it is not [\"image/gif\", \"image/jpeg\", \"image/png\", \"image/svg+xml\"]
    - E74: spec.icon can only contain two fields: \"base64data\" and \"mediatype\"
    - E75: spec.icon should be a singleton list
    - E76: category is not valid if it is not [
                "AI/Machine Learning",
                "Application Runtime",
                "Big Data",
                "Cloud Provider",
                "Developer Tools",
                "Database",
                "Integration & Delivery",
                "Logging & Tracing",
                "Monitoring",
                "Networking",
                "OpenShift Optional",
                "Security",
                "Storage",
                "Streaming & Messaging"
            ]

**\*** Validation may be subjected to relocation or included as part of extended package.

#### Public validation output struct

```go
type alert struct {
  source    string
  code      string
  message   string
  severity  string
}
```

### Risks and Mitigations

- If any set of rules is outdated or ill-formed that reports false-negative alerts, valid Operators may be blocked from being released or installed. Since we allow explicit enabling/disabling individual alert and error messages, we can always have a work around as temporary solutions.

- We are mostly concerned with false-negative reports that block valid Operators. We can resort to alert and report only critical mal-formed bundles or missing information. Any none-critical alerts should be categorized as such.

## Design Details

### Test Plan

Tests including unit and e2e should collaborate with bundle definitions. Any change in bundle semantics and outlooks should be tested with existing validations and vice versa.

### Version Skew Strategy

The bundle definitions and format may evolve in the future, which may be separable by bundle spec versions. An operator bundle may specify a supported version and, thereby, examined by its corresponding validations. 

## Drawbacks

Operator developers/users might expect and depend on the effectiveness of validation to cover all possible combinations of configurations, whereas not all edge cases could be perfectly covered. In addition, the alerts could also misinform the actual cause of an invalid Operator.

## Alternatives

An alternative to validate bundles library is to use [operator-courier](https://github.com/operator-framework/operator-courier) to perform a static validation on Operator manifest format.
