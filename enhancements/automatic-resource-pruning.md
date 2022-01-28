---
title: Automatic Resource Pruning
authors:
  - "@ryantking"
reviewers:
  - "@jmrodri"
  - "@gallettilance"
approvers:
  - "@jmrodri"
creation-date:2022-01-28
last-updated: 2022-01-28
status: implementable
---

## Summary

This EP aims to provide an easy way for users to operator authors to limit the number of ephemeral resources that may
exist on the cluster at any point in time. In this context, we define an ephemeral resource as a short-lived resource
that is continuously created as the operator runs. Short-lived does not refer to a specific amount of time, rather, the
fact that the resource has a defined life span. For example, a web server running in a `Pod` is not ephemeral because it
will run until an external force such as a cluster administrator or CD pipeline acts on it while a log rotating script
running in a `Pod` is ephemeral since it will run for until it finishes its defined work. Operator authors will be able
to employ different strategies to limit the number of ephemeral resources such as adding a numerical limit, an age
limit, or something custom to the author's use case.

## Motivation

Often, operators will create ephemeral resources during execution. For example, if we imagine an operator that
implements Kubernetes' builtin `CronJob` functionality, every time the operator reconciles and finds a `CronJos` to run,
it creates a new `Job` type to represent a single execution of the `CronJob`. Users will often want to have access to
theses ephemeral resources in order to view historical data, but want to limit the number that can exist on the system.
Looking again at Kubernetes out-of-the-box functionalities, users can configure the retention policy for resources such
as `ReplicaSets` and `Jobs` to maintain a certain amount of historical data. Operator authors should have a defined path
for implementing the same functionality for their operators.

### Goals

- Add a library to [operator-lib](https://github.com/operator-framework/operator-lib) that houses this functionality
  with a user-friendly API.
- Add a happy path for what we determine to be common use cases, such as removing `Pods` in a finished state if they are
  older than a set age.
- Provide an easy way for operator authors to plug in custom logic for their custom resource types and use cases.

### Non-Goals

- Add auto-pruning to any of the templates or scaffolding functionality.
- Adding auto-pruning support for Helm or Ansible operators.

## Proposal

The proposed implementation is adding a package, `prune`, to
[operator-lib](https://github.com/operator-framework/operator-lib) that exposes this functionality. There will be a
primary entry point function that takes in configuration and prunes resources accordingly. The configuration will accept
one or many resource types, a pruning strategy, namespaces, label selectors, and other common settings such as a dry run
mode, hooks, and logging configuration.

Another aspect of the library will be determining when it can and cannot prune a resource. For example, the prune
functionality should not remove a running `Pod` until it has completed, even if it meets the strategy criteria. The
library will expose a way for operator authors to specify what criteria makes a resource of a specific type safe to
prune; this can include checking annotations, a status, or any other data available on the resource. An important
distinction to draw is the difference between a strategy and checking whether it is safe to prune a resource (henceforth
called the "is-pruneable" functionality). Strategies look generically at collections of resources and decide which
resources in the collection to prune, if any. They should only take criteria common to all Kubernetes resources into
account such as the count of resources and creation timestamp. The is-pruneable functionality conversely looks at one
and only one resource at a given point in time to determine whether or not the current prune task can remove a resource
based on its specific data.

Note that stategies will not be programmatically limited to being resource agnostic, but it will be a defined best
practice to write strategies in such a way. One exception to this recommendation will be when the operator author wants
to prune based on a cumulative value such as the summation of a field across multiple resources. The operator author
must then be sure to add safe guards to the strategy to avoid unexpected behavior if used with an incompatible resource.

### User Stories

#### Story 1

An an operator author, I want to limit the number of `Jobs` in a completed state on the cluster so that the long term
storage cost of my operator has a cap.

#### Story 2

As an operator author, I want to limit the number of `Pods` in a completed state on a the cluster and preserve the
`Pods` in an error state so that the long term storage cost of my operator has a soft cap and the user can see status
information about failed `Pods` before manually removing.

#### Story 3

As an operator author, I want to limit the number of `Pods`with a custom heuristic based on the creation timestamp so
that the long term storage cost of my operator has a cap based on my operator's logic.

#### Story 4

As an operator author, I want to prune a custom resources with specific status information when there is a certain
number so that the long term storage cost of my operator has a cap.

#### Story 5

As an operator author, I want to prune both `Jobs` and `Pods` of a certain age so that the long term storage cost of my
operator has a cap and there are no orphaned resources.

### Implementation Details

- A strategy is a function that takes in a collection of resources and returns a collection of resources to remove.
- The identifier for a resource types will be `GroupVersionKind` value.
- An is-pruneable function takes in one instance of a resource and returns an error value that indicates if it is safe
  to prune or has other issues.
- The library will provide built-in is-pruneable functions for `Pods` and `Jobs` that can be overwritten.
- A registry will hold a mapping of resource types (`GVKs`) to is-pruneable functions.

A proposed go API is in [Appendix A](#appendix-a).

### Risks and Mitigations

The primary risk associated with this EP is exposing too many knobs and features in the day 1 implementation. We can
mitigate this by only exposing functionality that is absolutely needed. APIs are easy to grow, but near-impossible to
shrink.

## Design Details

### Test Plan

The following components will be unit tested:

- The builtin strategies.
- The builtin is-prunable functions.
- The main prune routine.

The feature author will add an integration test suite that runs the prune routine in the use cases defined in the user
stories.

## Implementation History

[operator-framework/operator-lib#75](https://github.com/operator-framework/operator-lib/pull/75): Implements a first
pass of the prune package with only support for `Jobs` and `Pods`. The API is also slightly different than the one
proposed in this EP.

## Drawbacks

The user will need to manually integrate this functionality into their operator since it is a library.

## Alternatives

An alternative approach would be adding this logic to the core SDK and scaffolding it optionally during operation
generation. The primary drawbacks with this approach are the increased complexity to the implementation and adding it to
existing operators.

## Open Questions

- What are the predefined use cases that we want to support? Currently we support pruning completed `Jobs` and `Pods` by
  age and max count.

### Implementation-specific

- What type of Kubernetes object should we generically work with? E.g. `metav1.Object`or `runtime.Object`?
- How do we specify which Kubernetes objects to delete? Pass back another list of objects? We just need name, namespace,
  and `GVK`.
- Which Kubernetes client should we work with? Dynamic client due to custom resource types?
- Should we register `IsPruneable` functions or a `ResourceConfig` structure that will hold that function and
  potentially additional configuration.

## Appendix A

The following is the proposed Go API:

```go
// StrategyFunc takes a list of resources and returns the subset to prune.
type StrategyFunc func(ctx context.Context, objs []runtime.Object) ([]runtime.Object, error)

// ErrUnpruneable indicates that it is not allowed to prune a specific object.
type ErrUnpruneable struct {
  Obj *runtime.Object
  Reason string
}

// IsPruneableFunc is a function that checks a the data of an object to see whether or not it is safe to prune it.
// It should return `nil` if it is safe to prune, `ErrUnpruneable` if it is unsafe, or another error.
// It should safely assert the object is the expected type, otherwise it might panic.
type IsPruneableFunc func(obj *runtime.Object) error

// RegisterIsPruneableFunc registers a function to check whether it is safe to prune a resources of a certain type.
func RegisterIsPrunableFunc(gvk schema.GroupVersionKind, isPruneable IsPruneableFunc) { /* ... */ }

// Pruner is an object that runs a prune job.
type Pruner struct {
  // ...
}

// PrunerOption configures the pruner.
type PrunerOption func(p *Pruner)

// NewPruner returns a pruner that uses the given startegy to prune objects.
func NewPruner(client dynamic.Interface, opts ...PrunerOption) Pruner { return Pruner{} }

// Prune runs the pruner.
func (p Pruner) Prune(ctx Context) error { return nil }
```
