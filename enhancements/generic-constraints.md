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
last-updated: 2020-11-04
status: provisional
---

# generic-constraint

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

The dependency resolution in OLM needs to support constraints that are based on arbitrary properties from either bundles or clusters beyond the existing GVK or package properties.

This enhancement proposes the adoption of [Common Expression Language
(CEL)](https://github.com/google/cel-go) to present the specification and usage of generic constraint that OLM can evaluate for dependency resolution at runtime.

CEL is a sufficiently lightweight expression language that has been adopted in multiple opensource projects. It allows users to express operator constraints in both straight-forward and advanced cases without OLM having to dictate the evaluation methodology.

## Motivation

At the moment, OLM only supports 2 built-in types of constraint:
- GVK constraint (`olm.gvk.required`)* which depends on a specific GroupVersionKind information of a CRD/API.
- Package constraint (`olm.package.required`)* which depends a specific package name and version (or a range of version)

Both types are specifically handled by the resolver in OLM and cannot be changed or expanded to support further use cases. In order to support more types of constraints, it is generically expected for OLM to add more specific built-in types to allow users to express those new constraints in the bundles. Over the time, it may become cumbersome and impractical to continue to add more specific types to the resolver. The need for OLM to support a generic constraint model becomes more apparent. With a generic constraint model, OLM no longer needs to carry any specific evaluation methodology besides the understanding of how to parse the constraint syntax. The users should be able to express declaratively the rules for a constraint that can be evaluated againts the dataset of arbitrary properties.

* Note: The current GVK and package constraints are specified in `dependencies.yaml` using `olm.gvk` and `olm.package` type and they are coverted to `olm.gvk.required` or `olm.package.required` property (which is required constraint) to differentiate from the `olm.package` and `olm.gvk` property in the bundle.

### Goals

- Provide specification and guideline to specify a constraint that contains declarative requirements/rules.
- Support the compound constraint (the constraint with mutliple requirements that is resolved into a single operator).

### Non-Goals

- Eliminate the existing built-in constraints.
- Support multiple expression languages other than CEL (though the syntax may allow the support for other languages for future use).
- Provide mechanism to provide cluster constraints. This feature can be addressed in a separate enhancement.


## Proposal

The [Common Expression Language (CEL)](https://github.com/google/cel-go) is a great inline expression language that can be used to express multiple rules within a single expression. The rules are fully customizable and constructed specifically for users' needs and requirements without OLM needing to know the design methodology. It is important to recoginize the dependent relationship between constraints and properties. As a constraint is evaluated against a dataset of properties from a bundle or a given source, in order to design a constraint, there must be an understanding on what properties are available in a given a bundle or source and what information they present in order for a constraint to be evaluated propertly.

The CEL library also supports [extension functions](https://github.com/google/cel-spec/blob/master/doc/langdef.md#extension-functions) which allows OLM to implement additional functions that may be necessary for certain operations. These functions then can be used within the CEL expression. For example, [semver](https://semver.org/) comparison is not built-in in CEL and it is often used in OLM as bundles are versioned in semver format. The semver functions can be added in the initial release and allow users to express semver evaluations in the constraint. However, extension functions should be treated and designed with care as they can be difficult to change in the future and potentially create version skew or breaking changes.

All CEL expressions are [parsed and typechecked](https://github.com/google/cel-go#parse-and-check) before evaluation. The insulting programs can be cached for later evaluation to reduce unnecesary computation. Also, the CEL evaluation is [thread-safe and side-effect free](https://github.com/google/cel-go#evaluate).

### User Stories

#### Generic Constraint

As an operator author, I would like to specify a constraint that has my own rules/requirements that are appliable to a set of known properties are from other bundles or from the cluster itself without having to tell OLM how to evaluate those rules. For example, if I know there is a property named certified in some bundles, I can express a constraint with a rule that says my operator to only be installed along with those certified bundles without having to tell OLM to support a new type of constraint for certified property.

#### Compound Constraint

As an operator author, I would like to specify a constraint that has multiple rules and requirements that must be resolved into a single dependent operator. For example, I would like to have a constraint for a dependent operator that belongs to a specific package and provides a specific API/GVK.

### Design Details

#### Generic Constraint Syntax

The constraint syntax is designed to have similarity with the existing constraints that OLM supports which are GVK and package. It is intentional to keep the fundamental syntax with `type` and `value` the same so that it doesn't introduce any dramatic changes to the overall user experiences.

Each constraint is specified using the following syntax:

```json=
{
   "type":"olm.constraint",
   "value":{
      "evaluator":{
         "id":"cel"
      },
      "rule": "properties.exists(p, p.type == \"certified\")",
      "message": "require to have certified property",
      "action":{
         "id":"require"
      }
   }
}
```

The constraint `type` is `olm.constraint` and the `value` is information associated with constraint. The `value` struct has 4 fields: `evaluator`, `rule`, `message` and `action`.

The `evaluator` is a struct with `id` field to represent the language library that will be used to evaluate the expression. At the moment, only CEL expression is supported using `cel` as the identifier. Given the possibility of supporting other expression languages in the future, the `evaluator` is constructed as a struct so that additional fields can be added without breaking the current syntax. For example, if we support a hypothetical expression language named `lucky` that has ability to dynamically load a package named `bobby` at runtime, a `package` field can be added to `evaluator` struct to support that ability:

```json=
"evaluator":{
    "id":"lucky",
    "package":"bobby"
},
```

The `rule` field is the string that presents a CEL expression that will be evaluated during resolution against . Only CEL expression is supported in the initial release.

The `message` field is an optional field that is accommodating the rule field to surface information regarding what this CEL rule is about. The message will be surfaced in resolution output when the rule evaluates to false.

Finally, the `action` field is a struct with the `id` field to indicate the resolution action that this constraint will behave. For example, for `require` action, there must be at least one candidate that satisfies this constraint. This action can be used to indicate optional constraint in the future adding a new field `optional` such as:

```json=
"action":{
    "id":"require"
    "optional":true
},
```

#### Compound Constraint

The CEL [syntax](https://github.com/google/cel-spec/blob/master/doc/langdef.md#syntax) supports a wide range of operators including logic operator such as AND and OR. As a result, a single CEL expression can have multiple rules for multiple conditions that are linked together by logic operators. These rules are evaluated against a dataset of multiple different properties from a bundle or any given source and the output is solved into a single bundle or operator that sastifies all of those rules within a single constraint. For example:

```json=
{
  "type": "olm.constraint",
  "value": {
    "evaluator": {
        "id": "cel",
    }
    "rule": 'properties.exists(p, p.type == "certified") && properties.exists(p, p.type == "stable")',
    "message": 'require to have "certified" and "stable" properties',
    "action": {
        "id": "require"
    }
  }
}
```

The expression in `rule` has two requirements that are linked in an AND (&&) logical operator. In order for that expression to be true, both requirements must be satisfied. Therefore, the resolved operator will be a single bundle that has both certified and stable properties. This concept represents the compound constraint.

### Risks and Mitigations

#### User experience concern

The CEL language is relatively new and requires a learning curve to understand. The complexity of the language may potentially create a difficult user experience (UX) especially when the users intend to write a complex expression for an advanced user case. Also, the current resolver doesn't surface the resolution information very clearly which leads to the difficulty on how the users can identify what goes wrong when the CEL expression is evaluated to `false`.

Mitigation: The feature can be implemented under a feature flag to allow feedbacks to be gathered and then the UX can be evaluated based on those feedbacks in order to improve the UX if needed in future releases.

Also, the constraint syntax can be simplified to reduce the complexity in order to improve the overall UX. (See Alternatives).

For majority of use cases, the CEL expression should be simple and straight-forward especially with well-constructed examples and guidelines where users can make a few plug-in changes and ready to be used.

The `message` field is introduced to allow specific information from the author who writes the expression to surface as resolution output.

#### CEL language maturity and support

The CEL language is relatively new and not yet at stable release (v1). There are potentially breaking changes that can be introduced.

Migration: The CEL library can be spinned to a specific release to avoid introducing breaking changes if occurs.

The language is being used in another well-known opensource projects including Tekton and Kubernetes ([apiserver/CRD](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/2876-crd-validation-expression-language/README.md)). This should improve the confidence on the usage of CEL language and these teams will potentially influence or impose against any major breaking changes that may come to CEL.

If CEL becomes too instable in the future due to breaking changes, it is possible to gradually deprecate the support of the language and introduce another language that is more stable. The constraint syntax allows the support of other languages in the future via `evaluator` field.

## Alternatives

### Other languages choices

While CEL is the selected language to be supoorted in the new OLM constraint type. It is not the intention to limit OLM to only support CEL expression. It possible to support multiple expression languages in the future releases if needed.

Besides CEL, there are other languages that can be used such as:

- [Rego](https://github.com/open-policy-agent/opa/tree/main/rego) (Open Policy Agent)
- [Expr](https://github.com/antonmedv/expr)
- [Starlark](https://github.com/bazelbuild/starlark)
- [Cue](https://github.com/cue-lang/cue)

All of these languages can be supported in constraint type. The `evaluator` field is designed to support multiple evaluators/languages if needed. However, intrducing a new language to the constraint type should be evaluated carefully to ensure there is a real need for it. Providing the support for multiple languages can be overwhelming and potentially creates a fragmented user experiences and unnecessary maintainance effort in a long run.

### Alternative constraint syntax

#### Custom constraint type syntax

The current proposed constraint syntax is designed to support other expression languages in the future. The current `evaluator` struct is designed to suppport additional fields to falicitate additional information that new expression languague may require. Additionally, other goals of this EP to support potentual custom constraints which may not use expression language and have additional fields without having to introduce a new constraint type. The complexity of using `evaluator` struct as an identifier and depending on identifier, more fields are required in the `value` struct can potentually be confusing as fields become intermingled. The alternative syntax is to shift cel constraint type into its own struct under `value` struct.

```yaml=
type: olm.constraint
value:
    message: 'require to have "certified" and "stable" properties'
    CEL:
        rule: 'properties.exists(p, p.type == "certified") && properties.exists(p, p.type == "stable")'
    action:
        id: "require"
        optional: true
```

```
type CEL struct {
    Rule   string
}
```

If there is a new custom type introduced, a new go struct will be added:

```
type CEL struct {
    Rule   string
    Action string
}

type PropertyEquals struct {
    Name  string
    Value string
}
```

The overall construct struct in go:

```
type Constraint struct {
    Message        string
    Action         Action
    CEL            *CEL
    PropertyEquals *PropertyEquals
}

type Action struct {
    id       string
    optional bool
}
```

The constraint syntax in YAML for the new `PropertyEquals` type is:

```yaml=
type: olm.constraint
value:
    message: 'require to have "certified" and "stable" properties'
    propertyEquals:
        name: "olm.maxOCPVersion"
        value: "4.9"
    action:
        id: "require"
        optional: false
```

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

The identifier `type` for this constraint is `olm.constraint.cel` which means this is a CEL constraint that only supports CEL expression. The `value` field contains the `rule` field which houses the CEL expression and the `message` field which is the resolution output if the expression is evaluated as `false`.

This simplified version of constraint type still satisfies the goals of having a generic constraint (though to lesser extend due to lacking of expandability that is provided by additional fields) and the compound constraint.
