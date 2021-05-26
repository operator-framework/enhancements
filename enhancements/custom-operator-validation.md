---
title: custom-operator-validation
authors:
  - "@exdx"
reviewers:
  - "@kevinrizza"
  - "@harishsurf"
approvers:
  - "@kevinrizza"
  - "@ecordell"
creation-date: 2020-08-26
last-updated: 2020-09-01
status: provisional
see-also:
  - "https://github.com/operator-framework/enhancements/pull/28"  
replaces:
  - "https://github.com/operator-framework/enhancements/pull/28"
---

# Enable custom validation rules for operators

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

This enhancement adds support for enabling custom sets of validation logic to run against [operator bundles](https://github.com/openshift/enhancements/blob/master/enhancements/olm/operator-bundle.md) as they are being added to an index. Different personas, such as pipeline maintainers and cluster admins, have different validation requirements for operators and the additional kubernetes objects packaged with those operators. This enhancement introduces a flexible and extendible approach to allow varying degrees of validation based on the users needs. As OLM continues to grow and support more objects in the operator bundle, this approach will enable more custom validation to be easily added. 

## Motivation

The primary motivation is to allow more flexible generation and modification of operator bundle validation rules. There currently exists a set of rules in the `operator-framework/api` repository as well as additional rules in the `operator-framework/registry` repository that encode validation logic as a hard-coded sequence of rules written in Go. The desire is to allow additional rules to be created and used during the validation process that are easier to implement and customize. 

Since OLM now supports additional objects in the bundle, such as `PodDisruptionBudgets`, a motivation was to enable validation for these additional objects to prevent them from disrupting operations on a kubernetes cluster. As more arbitrary objects are supported the validation library will be increasingly important in ensuring operator bundles do not have side-effects that impact other workloads on the cluster.

Adding support for per-index level validation can provide a natural link between the index and the type of validation bundles need to pass before being successfully added to the index. Certain indexes can have more validation rules apply compared to other indexes. 

### Goals

- Support validation of various objects in the operator bundle based on template files which specify the rules the objects must adhere to.
- Enable custom validation rules to be easily consumed by various `operator-framework` tooling so that users can add/modify the validation rules as they see fit.
- Provide a set of default validation rules for various objects such as `PodDisruptionBudgets`, `PriorityClasses`, and others. 
- Document and provide examples of how to create custom validation rules for end users. 

### Non-Goals

- Exposing a list of bundles that fail validation from an index. This is considered in a separate enhancement. 
- Rewrite the existing validation rules to follow the new approach. 
- Provide dynamic, on-cluster validation of bundles and existing objects on the cluster. 
- Replace existing alert-based mechanisms for detecting invalid objects and unsafe bundles that may already exist on-cluster. 

## Proposal

### Express validation rules as static configuration files

One approach to validation is using the [CUE Data Constraint Language](https://github.com/cuelang/cue) to encode validation rules as `.cue` files which can then be imported and analyzed by the existing `opm` tool when running the `opm alpha bundle validate` command. 

Consider an operator bundle that includes a `PriorityClass` in its `/manifests` directory. 

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: true
```

Adding a `PriorityClass` to the cluster can impact other workloads if the `globalDefault` setting is set to `true`. See [this document](https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/adding-priority-classes.md#caveats) for a detailed look into the caveats of adding a `PriorityClass`. Therefore, a validation rule could enforce that `globalDefault` is set to `false` for all `PriorityClass` objects in the bundle. This rule can be enforced as a cue file. 

- CUE is a configuration language that allows one to define a `<validation-rule>.cue` template for validating kubernetes manifests like `PriorityClass` objects.
- CUE can act as an extendable library to accept a validation template for a resource and then report whether or not the resource passes that particular validation check.

To validate the above `PriorityClass` its first imported via `cue import priorityclass.yaml` which generates a `priorityclass.cue` file which represents the validation spec for that particular object. The `.cue` file is then edited to reflect the validation rule that `globalDefault` cannot be set to `true` on a provided `PriorityClass`. The following is the `priorityclass.cue` file that encodes this validation rule. 

```yaml
apiVersion: "scheduling.k8s.io/v1"
kind:       "PriorityClass"
globalDefault?: !=true
```

Validating the example `PriorityClass`, located in a file called `prioriyclass.yaml`, against the `prioriyclass.cue` spec:

```
$ cue vet priorityclass.yaml priorityclass.cue
globalDefault: invalid value true (excluded by !=true):
    ./priorityclass.cue:3:17
    ./priorityclass.yaml:6:17
```

The custom validation rule was enforced and the `PriorityClass` object did not pass the validation check. A bundle containing such a `PriorityClass` would not be allowed when using the `opm` tool to run validation on the bundle. 

Another tool that can be used to fulfill the same goal is the Rego-based `conftest` project. The same validation rule expressed via `conftest` would look like the following in the `policy/priorityclass.rego` file. 

```go
package main

deny[msg] {
  input.kind = "PriorityClass"
  not input.spec.globalDefault = true
  msg = "Bundles cannot contain PriorityClass objects with the globalDefault option set to true"
}
```

Both `cue` and `conftest` formats provide the same functionality: an easily readable and extendible format for writing custom operator validation rules. 

### Storing validation rules

The set of rules will be stored within the `operator-framework` project. This enables anyone to open a PR adding additional rules, or forking the repository and generating their own sets of validation. 

By providing a git-based repository for rules, it will be easy to revise and add new rules as required. 

The default rule repository will be located within https://github.com/operator-framework/api. 

It will include documentation, tagged releases, and an intuitive heirarchy of default validation rules. 

### Consuming validation rules

Users can choose whether to run the default set of validations, in the format described above, or pass their own validation scripts which would run and return an exit code that determines whether or not the validation was successful. 

It should be noted that existing bundle validation rules will not change, and if operator authors do not include new objects in their bundle, they will not be impacted. The existing validation rules will still be run for every bundle. 

There will be a configuration for validation, and option for manual overrides of rules, so that users can skip certain rules or mark them as warnings instead of errors. 

### Providing resources for building validation rules

As part of delivering the enhancement, it should be clear to users how to successfully generate their own sets of validation rules and run them against bundles. This documentation will be delivered as part of the feature enhancement. 

### User Stories 

#### A pipeline maintainer wants to add custom validation rules for specific objects

Different pipelines have different requirements for objects that are allowed to be included in the bundle, and should be free to develop their own rules as necessary. This enhancement should enable them to quickly generate their own sets of rules. 

For example, a pipeline can run their own additional custom validation scripts to enforce stricter rules on objects in the pipeline when the bundle is being added to a staging index. 

#### A cluster admin wants to add custom validation rules for specific objects

Cluster admins have different requirements for objects that are allowed to be included in the bundle, and should be free to develop their own rules as necessary. This enhancement should enable them to quickly run validation rules and get back results on whether or not a bundle conforms to those rules.

For example, a cluster admin can choose to run the default validation rules and remove some validation rules that are not relevant to them to enforce less rules on objects in the cluster. 

### Implementation Details/Notes/Constraints

A caveat is that its best not to shell out to the underlying validation binary directly when running validation commands to check objects against the relevant rules. Therefore, tools will import the relevant libraries in Go and run the validation itself, so the resulting logic is encoded into the binary. 

### Risks and Mitigations

`cue` and `conftest` are both fairly young projects (currently pre-1.0) and the api may change. This is mitigated somewhat by vendoring in the dependent code. However, if the format of the rules were to change suddenly, it could impact all those relying on the previous format.

Opening up the validation rules to the broader community could introduce potential security risks. 

## Design Details

### Test Plan

Testing the validation is fairly straightforward. Provide a series of bundles to the new validation logic, and based on the rule templates verify that the bundles either pass the validation rules as expected. A series of table-driven tests with fixtures could generate the necessary test coverage. 

### Version Skew Strategy

There is a risk that as kuberentes releases new versions, certain rules may no longer be valid. For example, a field may have become deprecated as an APIVersion changed. The templates may have to be tagged as relevant to a certain version of kubernetes.  

## Drawbacks

Relying heavily on a preexisting rule format may be a drawback as this is the first time it is being used within OLM. For example, calling into the `cue` library directly, instead of the well-documented CLI tool, may produce some undefined behavior that the CLI handles at a higher level. 

`cue` does not provide robust error messaging for its validation results, which can lead to errors that are hard for end users to fully understand. One way to mitigate this is to provide a custom error handling library that improves error messaging. 

Nothing stops users from side-stepping the validation provided here and using their own custom bundles and images to run on the cluster. 

## Alternatives

The alternative is to continue encoding validation rules as Go code in the `operator-framework/api` repository. This is easier in the short term but does not scale well and does not allow OLM to address the needs of the various personas involved in bundle validation outlined above. 

