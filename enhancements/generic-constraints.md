---
title: generic constraint
authors:
  - "@dinhxuanvu"
reviewers:
  - "@kevinrizza"
  - "@joelandford"
approvers:
  - "@kevinrizza"
creation-date: 2021-08-26
last-updated: 2020-11-17
status: implementable
---

# generic-constraint

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA

## Summary

The dependency resolution in OLM needs to support constraints that are based on arbitrary properties from either bundles or clusters beyond the existing GVK or package properties.

This enhancement proposes the adoption of [Common Expression Language
(CEL)](https://github.com/google/cel-go) to present the specification and usage of generic constraint that OLM can evaluate for dependency resolution at runtime.

CEL is a sufficiently lightweight expression language that has been adopted in multiple opensource projects. It allows users to express operator constraints in both straight-forward and advanced cases without OLM having to dictate the evaluation methodology.

## Motivation

At the moment, OLM only supports 2 built-in types of constraint:
- GVK constraint (`olm.gvk.required`)* which depends on a specific GroupVersionKind information of a CRD/API.
- Package constraint (`olm.package.required`)* which depends a specific package name and version (or a range of version)

Both types are specifically handled by the resolver in OLM and cannot be changed or expanded to support further use cases. In order to support more types of constraints, it is generically expected for OLM to add more specific built-in types to allow users to express those new constraints in the bundles. Over the time, it may become cumbersome and impractical to continue to add more specific types to the resolver. The need for OLM to support a generic constraint model becomes more apparent. With a generic constraint model, OLM no longer needs to carry any specific evaluation methodology besides the understanding of how to parse the constraint syntax. The users should be able to express declaratively the rules for a constraint that can be evaluated against the dataset of arbitrary properties.

* Note: The current GVK and package constraints are specified in `dependencies.yaml` using `olm.gvk` and `olm.package` type and they are converted to `olm.gvk.required` or `olm.package.required` property (which is required constraint) to differentiate from the `olm.package` and `olm.gvk` property in the bundle.

### Goals

- Provide specification and guideline to specify a constraint that contains declarative requirements/rules.
- Support the compound constraint (the constraint with multiple requirements that is resolved into a single operator).

### Non-Goals

- Eliminate the existing built-in constraints.
- Support multiple expression languages other than CEL (though the syntax may allow the support for other languages for future use).
- Provide mechanism to provide cluster constraints. This feature can be addressed in a separate enhancement.


## Proposal

The [Common Expression Language (CEL)](https://github.com/google/cel-go) is a great inline expression language that can be used to express multiple rules within a single expression. The rules are fully customizable and constructed specifically for users' needs and requirements without OLM needing to know the design methodology. It is important to recognize the dependent relationship between constraints and properties. As a constraint is evaluated against a dataset of properties from a bundle or a given source, in order to design a constraint, there must be an understanding on what properties are available in a given a bundle or source and what information they present in order for a constraint to be evaluated properly.

The CEL library also supports [extension functions](https://github.com/google/cel-spec/blob/master/doc/langdef.md#extension-functions) which allows OLM to implement additional functions that may be necessary for certain operations. These functions then can be used within the CEL expression. For example, [semver](https://semver.org/) comparison is not built-in in CEL and it is often used in OLM as bundles are versioned in semver format. The semver functions can be added in the initial release and allow users to express semver evaluations in the constraint. However, extension functions should be treated and designed with care as they can be difficult to change in the future and potentially create version skew or breaking changes.

All CEL expressions are [parsed and typechecked](https://github.com/google/cel-go#parse-and-check) before evaluation. The insulting programs can be cached for later evaluation to reduce unnecessary computation. Also, the CEL evaluation is [thread-safe and side-effect free](https://github.com/google/cel-go#evaluate).

### User Stories

#### Generic Constraint

As an operator author, I would like to specify a constraint that has my own rules/requirements that are appliable to a set of known properties are from other bundles or from the cluster itself without having to tell OLM how to evaluate those rules. For example, if I know there is a property named certified in some bundles, I can express a constraint with a rule that says my operator to only be installed along with those certified bundles without having to tell OLM to support a new type of constraint for certified property.

#### Compound Constraint

As an operator author, I would like to specify a constraint that has multiple rules and requirements that must be resolved into a single dependent operator. For example, I would like to have a constraint for a dependent operator that belongs to a specific package and provides a specific API/GVK.

### Design Details

#### Generic Constraint Syntax

The constraint syntax is designed to have similarity with the existing constraints that OLM supports which are GVK and package. It is intentional to keep the fundamental syntax with `type` and `value` the same so that it doesn't introduce any dramatic changes to the overall user experiences. The proposed constraint syntax is designed to support other expression languages in the future. Additionally, other goals of this EP to support potential custom constraints which may not use expression language and have additional fields without having to introduce a new constraint type. As a result, each constraint type is designed to be an individual struct with custom fields that are specific to that type. New types can be introduced and added to `value` struct to support new constraint type without needing a new `type` identifier.

At the top level, `type` field is the identifier for the new constraint type named `olm.constraint`. The `value` field is a struct that contains all information related to the constraint. Under `value`, the `failureMessage` field is a place to include string-representation of the constraint failure message that will be surfaced to the users if the constraint is not satisfiable at runtime.

Each constraint is specified using the following syntax:


```yaml=
type: olm.constraint
value:
    failureMessage: 'require to have "certified" and "stable" properties'
    cel:
        rule: 'properties.exists(p, p.type == "certified") && properties.exists(p, p.type == "stable")'
```

```
type Cel struct {
    Rule   string
}
```

The `Cel` struct is specific to CEL constraint type that supports CEL as the expression language. The `Cel` struct has `rule` field which contains the CEL expression string that will be evaluated against bundle properties at the runtime to determine if the bundle satisfies the CEL expression.

If there is a new custom type named `PropertyEquals` introduced, a new go struct will be added along with the existing `Cel` struct:

```
type Cel struct {
    Rule   string
}

type PropertyEquals struct {
    Name  string
    Value string
}
```

Then, the `PropertyEquals` type is added into the `ConstraintValue` struct to support the new type:

```
type ConstraintValue struct {
    FailureMessage        string
    Cel                   *Cel
    PropertyEquals        *PropertyEquals
}
```

The constraint syntax in YAML for the new `PropertyEquals` type is:

```yaml=
type: olm.constraint
value:
    failureMessage: 'require to have "certified" and "stable" properties'
    propertyEquals:
        name: "olm.maxOCPVersion"
        value: "4.9"
```

##### Pros

* Expandable to support new expression languages and custom types by adding new struct
* Each type has its own struct with predefined fields which leads to easier to validate each type

##### Cons

* Require additional validation at build-time and/or run-time to ensure only one type is in use

#### Arbitrary Properties

At the moment, operator authors may declare arbitrary properties in `properties.yaml` file in the bundle metadata. While those properties will be processed and surfaced in the properties table in the sqlite database, OLM resolver simply doesn't have any understanding of those properties due to lacking of custom code to perform evaluation (compare) those properties. As a result, operator authors cannot declare dependencies on those properties. The generic constraint proposal allow operator authors to specify dependencies that rely on any types of properties via CEL expressions.

For example, the bundle properties map (which is an input to the resolver) is:

```yaml
properties:
  - property:
      type: sushi
      value: salmon
  - property:
      type: soup
      value: miso
  - property:
      type: olm.gvk
      value:
        group: olm.coreos.io
        version: v1alpha1
        kind: bento
```

The example constraint is specified to have a dependency for a custom property `sushi` with value `salmon`:

```yaml=
type: olm.constraint
value:
  failureMessage: "require to have the property `sushi` with value `salmon`"
  cel:
    rule: "properties.exists(p, p.type == 'sushi' && p.value == 'salmon')"
```

The cel expression will search for a property with the type `sushi` and the value `salmon` within the bundle property map and return `true` or `false`.

#### Compound Constraint

The CEL [syntax](https://github.com/google/cel-spec/blob/master/doc/langdef.md#syntax) supports a wide range of operators including logic operator such as AND and OR. As a result, a single CEL expression can have multiple rules for multiple conditions that are linked together by logic operators. These rules are evaluated against a dataset of multiple different properties from a bundle or any given source and the output is solved into a single bundle or operator that satisfies all of those rules within a single constraint. For example:

```json=
{
  "type": "olm.constraint",
  "value": {
      "failureMessage": 'require to have "certified" and "stable" properties',
      "cel": {
          "rule": 'properties.exists(p, p.type == "certified") && properties.exists(p, p.type == "stable")'
      }
  }
}
```

The expression in `rule` has two requirements that are linked in an AND (&&) logical operator. In order for that expression to be true, both requirements must be satisfied. Therefore, the resolved operator will be a single bundle that has both certified and stable properties. This concept represents the compound constraint.

### Graduation Criteria

This generic constraint is currently scheduled to be available in the next release with a set of supported functionalities. More functionalities will be added to the feature if needed in the subsequent releases.

Alternatively, it is possible to wrap this feature under feature flag which means it is disabled by default. As a result, the feature can be a dev-preview/tech-preview feature and begins to gather users' feedbacks.

As the feature is being used and evaluated in production, changes can be made to address feedbacks and improve overall usefulness and user experience. Then, the feature flag can be removed and the feature is enabled by default moving forward. At the time, the feature is GA.

### Test Plan

* operator-lifecycle-manager unit and e2e tests with the emphasis on resolver tests to ensure the new constraint type is working properly with existing dependency types and arbitrary properties.
* operator-registry unit and e2e tests to validate the configuration of the new constraint type to ensure it is properly constructed before being added to the index.

### Risks and Mitigations

#### User experience concern

The CEL language is relatively new and requires a learning curve to understand. The complexity of the language may potentially create a difficult user experience (UX) especially when the users intend to write a complex expression for an advanced user case. Also, the current resolver doesn't surface the resolution information very clearly which leads to the difficulty on how the users can identify what goes wrong when the CEL expression is evaluated to `false`.

Mitigation: The feature can be implemented under a feature flag to allow feedbacks to be gathered and then the UX can be evaluated based on those feedbacks in order to improve the UX if needed in future releases.

Also, the constraint syntax can be simplified to reduce the complexity in order to improve the overall UX. (See Alternatives).

For majority of use cases, the CEL expression should be simple and straight-forward especially with well-constructed examples and guidelines where users can make a few plug-in changes and ready to be used.

The `failureMessage` field is introduced to allow specific information from the author who writes the expression to surface as resolution output.

#### CEL language maturity and support

The CEL language is relatively new and not yet at stable release (v1). There are potentially breaking changes that can be introduced.

Migration: The CEL library can be spinned to a specific release to avoid introducing breaking changes if occurs.

The language is being used in another well-known opensource projects including Tekton and Kubernetes ([apiserver/CRD](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/2876-crd-validation-expression-language/README.md)). This should improve the confidence on the usage of CEL language and these teams will potentially influence or impose against any major breaking changes that may come to CEL.

If CEL becomes too unstable in the future due to breaking changes, it is possible to gradually deprecate the support of the language and introduce another language that is more stable. The constraint syntax allows the support of other languages in the future by adding new struct type that can be added under `value` struct.

## Alternatives

### Other languages choices

While CEL is the selected language to be supported in the new OLM constraint type. It is not the intention to limit OLM to only support CEL expression. It possible to support multiple expression languages in the future releases if needed.

Besides CEL, there are other languages that can be used such as:

- [Rego](https://github.com/open-policy-agent/opa/tree/main/rego) (Open Policy Agent)
- [Expr](https://github.com/antonmedv/expr)
- [Starlark](https://github.com/bazelbuild/starlark)
- [Cue](https://github.com/cue-lang/cue)

All of these languages can be supported in constraint type. A new struct with its own configuration can be created to support a new language similarly to `Cel` struct. However, introducing a new language to the constraint type should be evaluated carefully to ensure there is a real need for it. Providing the support for multiple languages can be overwhelming and potentially creates a fragmented user experiences and unnecessary maintenance effort in a long run.

### Alternative constraint syntax

#### Evaluator constraint type syntax

Instead of using individual type/struct under `value` struct, this proposed syntax uses an `evaluator` struct as an identifier to signal to the resolver which constraint type it is and which fields are needed for this type.

Each constraint is specified using the following syntax:

```json=
{
   "type":"olm.constraint",
   "value":{
      "evaluator":{
         "id":"cel"
      },
      "rule": "properties.exists(p, p.type == \"certified\")",
      "failureMessage": "require to have certified property",
      "action":{
         "id":"require"
      }
   }
}
```

The constraint `type` is `olm.constraint` and the `value` is information associated with constraint. The `value` struct has 4 fields: `evaluator`, `rule`, `failureMessage` and `action`.

The `evaluator` is a struct with `id` field to represent the language library that will be used to evaluate the expression. At the moment, only CEL expression is supported using `cel` as the identifier. Given the possibility of supporting other expression languages in the future, the `evaluator` is constructed as a struct so that additional fields can be added without breaking the current syntax. For example, if we support a hypothetical expression language named `lucky` that has ability to dynamically load a package named `bobby` at runtime, a `package` field can be added to `evaluator` struct to support that ability:

```json=
"evaluator":{
    "id":"lucky",
    "package":"bobby"
},
```

The `rule` field is the string that presents a CEL expression that will be evaluated during resolution against . Only CEL expression is supported in the initial release.

The `failureMessage` field is a field that is accommodating the rule field to surface information regarding what this CEL rule is about. The message will be surfaced in resolution output when the rule is evaluated to false.

Finally, the `action` field is a struct with the `id` field to indicate the resolution action that this constraint will behave. For example, for `require` action, there must be at least one candidate that satisfies this constraint. This action can be used to indicate optional constraint in the future adding a new field `optional` such as:

```json=
"action":{
    "id":"require"
    "optional":true
},
```

##### Pros

* Single-entry identifier is "easier" to validate
* Fully expandable as additional fields can be added to `value`/`evaluator`/`action` to accommodate new expression languages and custom resolver behavior.

##### Cons

* Nested structure can be difficult to construct and decipher
* Potential additional fields into multiple struct and sub-struct increase complexity and readability

#### Simplified CEL constraint type

The current proposed constraint syntax has most of information nested under `value` field. The complexity of a struct with nested fields can create a bad user experience. It is possible to introduce a constraint type that is specified for CEL

```json=
{
  "type": "olm.constraint.cel",
  "value": {
    "rule": 'properties.exists(p, p.type == "certified") && properties.exists(p, p.type == "stable")',
    "message": 'require to have "certified" and "stable" properties',
  }
}
```

The identifier `type` for this constraint is `olm.constraint.cel` which means this is a CEL constraint that only supports CEL expression. The `value` field contains the `rule` field which houses the CEL expression and the `failureMessage` field which is the resolution output if the expression is evaluated as `false`.

This simplified version of constraint type still satisfies the goals of having a generic constraint (though to lesser extend due to lacking of expandability that is provided by additional fields) and the compound constraint.

##### Pros

* Flat structure is simple and highly readable

##### Cons

* Lacking of expandability as the type is for cel language exclusively
