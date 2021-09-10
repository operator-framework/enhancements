---
title: Cluster Runtime Constraint
authors:
  - "@dinhxuanvu"
reviewers:
  - "@kevinrizza"
approvers:
  - "@kevinrizza"
creation-date: 2021-08-26
last-updated: 2020-09-06
status: provisional
---

# cluster-runtime-constraint

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

This proposal presents the specification and usage to enable runtime constraints to be considered during operator dependency resolution. This feature will ensure Operator Lifecycle Manage (OLM) to install operators that satisfy dependency requirements and runtime constraints if present.

## Motivation

At the moment, OLM resolves a set of operators which seem that they will work together based on dependency requirements that are specified by operator bundles. However, it is possible that those resolved operators will not work on the clusters due to cluster/runtime constraints. For example, if the cluster is running a specific Kubernetes version that is not compatible with the operators, the installed operators may fail to work properly on that given cluster.

The cluster runtime constraints are often unknown by the operator authors and should be specified by cluster admins. However, currently, there is no mechanism for cluster admins to specify runtime constraints that OLM can understand and take under consideration during installation.

### Goals
- Provide mechanism (including specification and guideline) for cluster admins to specify cluster runtime constraints
- Provide specification and guideline for author operators to specify runtime constraints/requirements if needed
- Block the updates/installations of operators based on cluster runtime constraints
- Allow the cluster runtime constraints to be generic and configurable by cluster admins

### Non-Goals

- Provide mechanism to retrieve/populate cluster runtime constraints automatically
- Allow cluster runtime constraints to be scoped
- Allow cluster runtime constraints to come from multiple/different sources
- Allow optional cluster runtime constraints


## Proposal

To provide a generic solution for cluster runtime constraints, a library named [cel-go](https://github.com/google/cel-go) that provides expression evaluation capabilities is utilized in this proposal.

### User Stories

#### Constraints specified by cluster admin

As a cluster admin, I would like to specify a list of cluster runtime constraints that are associated with a given cluster and all installable operators will meet these requirements.

#### Constraint properties specified by operator author

As an operator author, I would like to provide a list of runtime properties that may be needed to be considered before installation to ensure the operator working properly on the cluster.

### Implementation Details

#### Cluster Runtime Constraint Configmap

Cluster admins can specified cluster runtime constraints in a configmap (named `olm-runtime-constraints`) in `olm` namespace using this format:

```yaml=
apiVersion: v1
kind: ConfigMap
metadata:
  name: olm-runtime-constraints
  namespace: olm
data:
    properties: <list-of-expression-constraints-in-json>
```

Each constraint is specified using the following syntax:

```json=
{
	"type": "olm.constraint",
	"value": {
		"evaluator": {
			"id": "cel"
		},
		"source": "properties.exists(p, p.type == \"certified\")",
		"action": {
			"id": "require"
		}
	}
}
```

The constraint `type` is `olm.constraint` and the `value` is information associated with constraint. The `value` struct has 3 fields: `evaluator`, `source` and `action`.

The `evaluator` is a struct with `id` field to represent the language library that will be used to evaluate the expression. At the moment, only `cel-go` library is supported using `cel` as the identifier. More fields are potentially added to expand the supported expression languages/libraries in the future.

The `source` field is the string of expression that will be evaluated during resolution. Currently, only `cel-go` expressions are supported.

Finally, the `action` field is a struct with the `id` field to indicate the resolution action that this constraint will behave. For example, for `require` action, there must be at least one candidate that satisfies this constraint. At the moment, `require` and `conflict` actions are supported. More fields may be added into `action` struct to support more resolution actions in the future.

#### Runtime Dependencies in Bundle

The operator authors specify the cluster runtime requirements in `dependencies.yaml` using the following format:

```yaml=
dependencies:
  -
    type: olm.constraint
    value:
      action:
        id: require
      evaluator:
        id: cel
      source: "properties.exists(p, p.type == \"certified\")"
```

#### Runtime Constraint Resolution

The constraints that are specified in `olm-runtime-constraints` Configmap are converted into properties of a global existing virtual node in the cluster. All operators/candidates must satisfy those properties in order to be installed.

If constraints are added to bundles, they will be evaluated and must be satisfied similarly to `olm.gvk.required` and `olm.package.required` properties.

#### Compound Constraint

It is possible to write `cel-go` expression to evaluate multiple different properties in one single expression that is specified in one constraint. For example:

```json=
{
  "type": "olm.constraint",
  "value": {
    "evaluator": {
        "id": "cel",
    }
    "source": 'properties.exists(p, p.type == "certified") && properties.exists(p, p.type == "new")',
    "action": {
        "id": "require"
    }
  }
}
```

The expression in `source` has two requirements that are linked in an AND (&&) logical operator. In order for that expression to be true, both requirements must be satisfied. This concept represents the compound constraint.

### Risks and Mitigations

### Prototype

TBD

## Drawbacks

The [cel-go](https://github.com/google/cel-go) library is rather new and still under development. Even though cel-go has been used in other opensource projects such as Tekton, it has not yet reached stable release (v1). Breaking changes can be introduced to this library and potentially OLM behavior in the future.

The complexity of the library is not yet fully evaluated which could lead to a poor user experience if the library is too difficult to use and utilized.

One of the popular use cases is semver comparison which is not a supported value type in cel-go. Custom functions are required to support semver.

## Alternatives
