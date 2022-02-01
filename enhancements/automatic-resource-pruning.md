---
title: automatic-resource-pruning
authors:
  - '@ryantking'
reviewers:
  - '@jmrodri'
  - '@gallettilance'
  - '@fgiloux'
  - '@joelanford'
approvers:
  - '@jmrodri'
creation-date: 2022-01-28
last-updated: 2022-02-01
status: implementable
---

# Automatic Resource Pruning

## Summary

This EP will provide a way for operator authors to limit the number of unbounded resources that may exist on the
cluster. In the context of this EP, we define an "unbounded resource" as any resource that has the following two
properties:

1. The resource is continuously created as the operator runs.
2. The resource is not removed as the operator runs.

When those two properties exist for a resource, the total number of that resource will grow unbounded. The functionality
introduced by this EP will make it possible for operator authors to easily control the entire life cycle of a resource
by automatically removing resources that meet specific criteria. These criteria can include properties of individual
resources, the entire set of resources, or a combination of both.

## Motivation

Often, operators will create unbounded resources during execution. For example, if we imagine an operator that
implements Kubernetes' builtin `CronJob` functionality, every time the operator reconciles and finds a `CronJobs` to
run, it creates a new `Job` type to represent a single execution of the `CronJob`. Users will often want to have access
to theses unbounded resources in order to view historical data, but want to limit the number that can exist on the
system. Looking again at Kubernetes out-of-the-box functionalities, users can configure the retention policy for
resources such as `ReplicaSets` and `Jobs` to maintain a certain amount of historical data. Operator authors should have
a defined path for implementing the same functionality for their operators.

### Goals

- Add a library to [operator-lib](https://github.com/operator-framework/operator-lib) that houses this functionality
  with a user-friendly API.
- Add a happy path for what we determine to be common use cases, such as removing operator-owned `Pods` that are both in
  a finished state and older than a certain age.
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

An an operator author, I want to limit the number of operator-owned `Jobs` in a completed state on the cluster so that
the long term storage cost of my operator has a cap.

#### Story 2

As an operator author, I want to limit the number of operator-owned `Pods` in a completed state on a the cluster and
preserve the `Pods` in an error state so that the long term storage cost of my operator has a soft cap and the user can
see status information about failed `Pods` before manually removing.

#### Story 3

As an operator author, I want to limit the number of operator-owned `Pods`with a custom heuristic based on the creation
timestamp so that the long term storage cost of my operator has a cap based on my operator's logic.

#### Story 4

As an operator author, I want to prune a custom resource with specific state information when there is a certain number
so that the long term storage cost of my operator has a cap.

#### Story 5

As an operator author, I want to prune both operator-owned `Jobs` and `Pods` of a certain age so that the long term
storage cost of my operator has a cap and there are no orphaned resources.

#### Story 6

As an operator author, I want to prune a custom resource with specific state information with a custom strategy so that
the long term storage cost of my operator has a cap.

### Implementation Details

- A strategy is a function that takes in a collection of resources and returns a collection of resources to remove.
- The identifier for a resource types will be `GroupVersionKind` value.
- An is-pruneable function takes in one instance of a resource and returns an error value that indicates if it is safe
  to prune or has other issues.
- The library will provide built-in is-pruneable functions for `Pods` and `Jobs` that can be overwritten.
- A registry will hold a mapping of resource types (`GVKs`) to is-pruneable functions.

There are two proposed go APIis in [Appendix A](#appendix-a) and [Appendix B](#appendix-b).

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
- Should we mandate that an author must register any resource type that they wish to prune?

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

// Error returns a string reprenstation of an `ErrUnpruneable` error.
func (e *ErrUnpruneable) Error() string { return "" }

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

// NewPruner returns a pruner that uses the given strategy to prune objects.
func NewPruner(client dynamic.Interface, opts ...func(p *Pruner)) Pruner { return Pruner{} }

// Prune runs the pruner.
func (p Pruner) Prune(ctx context.Context) error { return nil }
```

## Appendix B

The following is a more modular Go API that can support more custom resource functionality in the future:

```go
// ErrUnpruneable indicates that it is not allowed to prune a specific object.
type ErrUnpruneable struct {
  Obj *runtime.Object
  Reason string
}

// Error returns a string reprenstation of an `ErrUnpruneable` error.
func (e *ErrUnpruneable) Error() string { return "" }

// Registration holds information about a resource with how it should be
type Registration struct {
  // IsPruneable is a function that checks the data of an object to check whether or not it is eligible for pruning.
  // It should return `nil` if it is eligible to prune, `ErrUnpruneable` if it is unsafe, or another error.
  // It should safely assert the object is the expected type, otherwise it might panic.
  IsPruneable func(obj *runtime.Object) error
}

// Registry holds configuration about specific resource types.
type Registry struct {
 // ...
}

// RegisterResource adds a resource to the registry.
func (r *Registry) RegisterResource(gvk *schema.GroupVersionKind, ...func(r *Registration)) { /* ... */ }

// GetRegistration returns a resource registered with the given GVK.
func (r *Registry) GetRegistration(gvk *schema.GroupVersionKind) (*Registration, bool) { return nil, false }

// Register adds a resource to the default registry.
func Register(gvk *schema.GroupVersionKind, ...func(r *Registration)) { /* ... */ }

// Get returns a resource registered in the default registry with the given GVK.
func Get(gvk *schema.GroupVersionKind) (*Registration, bool) { return nil, false }

// WithIsPruneable adds a function to the resource registration.
func WithIsPruneable(func (obj *runtime.Object) error) func (*Registration) { return nil }

// StrategyFunc takes a list of resources and returns the subset to prune.
type StrategyFunc func(ctx context.Context, objs []runtime.Object) ([]runtime.Object, error)

// Pruner is an object that runs a prune job.
type Pruner struct {
  // ...
}

// NewPruner returns a pruner that uses the given strategy to prune objects.
func NewPruner(client dynamic.Interface, strategy StrategyFunc, opts ...func(p *Pruner)) Pruner { return Pruner{} }

// Prune runs the pruner.
func (p Pruner) Prune(ctx context.Context) error { return nil }
```
