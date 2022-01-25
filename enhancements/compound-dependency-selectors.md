---
title: compound-dependency-selectors
authors:
  - "@estroz"
reviewers:
  - "@joelanford"
  - "@kevinrizza"
  - "@dinhxuanvu"
approvers:
  - "@joelanford"
  - "@kevinrizza"
  - "@dinhxuanvu"
creation-date: 2021-10-14
last-updated: 2021-11-30
status: implementable
see-also:
  - "/enhancements/operator-dependency-resolution.md"
replaces:
superseded-by:
---

# compound-dependency-selectors

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

None

## Summary

An [operator bundle][bundles] is a declaration of a Kubernetes operator in image format.
There are two components to a bundle: its Kubernetes manifests, and metadata about those
manifests such as package name, version, and what group-version-kind/APIs's the manifests provide.
These metadata are referred to as bundle [properties][properties], and are expressed as [declarative configuration][dc].
Operators can depend on other operators for providing APIs, ex. those created by CRDs,
to interact with; a great example is an operator requiring HA config storage and service discovery
could use Etcd's set of APIs, which are provided by an Etcd operator. These dependencies
are specified as "constraint" bundle properties as well, ex. `olm.gvk.required`.

When [OLM][olm] installs a bundle with constraint properties (constraints), the [resolver][olm-resolver] will evaluate them,
reduced down to a boolean expression, over the non-constraint properties of all currently installed bundles.
These constraints are either `olm.gvk.required`, `olm.package.required`,
or an [`olm.constraint` clause][arb-constraints]. After the resolver completes evaluation,
it either returns a non-satisfiability error or the "best" choice of constraint bundle to install.
Any properties not recognized by the resolver are ignored, whether constraints or otherwise.

## Motivation

The existing constraints `olm.package.required` and `olm.gvk.required` are the only constraints
required for most use cases. They are specified as follows:

```yaml
---
schema: olm.bundle
name: foo.v1.0.0
package: foo
properties:
- type: olm.package.required
  value:
    name: bar
    versionRange: '>=1.0.0'
- type: olm.gvk.required
  value:
    group: etcd.database.coreos.com
    kind: EtcdBackup
    version: v1beta2
```

The above bundle `foo.v1.0.0` requires that both a bundle in package `bar` that is of at least
version 1.0.0 _and_ a bundle that provides the `EtcdBackup` API in group `etcd.database.coreos.com`
and at version `v1beta2`. The resolver will translate those constraints into a boolean expression
that is evaluated over all installed bundles's properties in the cluster; if both `bar` and `etcd`
bundles with properties satisfying group and/or version constraints are installed, then
`foo.v1.0.0` installation can proceed.

However, what if:
- `foo.v1.0.0` can use multiple versions of the same API, but only one of that group-kind?
- `foo.v1.0.0` can use multiple versions of the same API, but only if another package version is present?
- `foo.v1.0.0` cannot use a particular API version, but can use all others?

These constraints are logical disjunctions and negations in character; the existing
top-level constraints are together a logical conjunction, although not explicit.
Currently the spec does not support specifying such "compound" dependency constraints.

### Goals

- Define logical conjunction, disjunction, and negation properties as dependency constraints, termed "compound constraints".
- Define nesting of compound constraints.
- Describe how compound constraints are evaluated.
- Describe what needs to change in OLM's resolver to support compound constraints.

### Non-Goals

- Change the existing constraint definitions.
- Break bundle spec or OLM backwards-compatibility in any way.

## Proposal

### User Stories

#### Story 1

As an operator author, I should be able to specify "any of these APIs"
or "none of these APIs" dependencies as bundle properties, and "all of these APIs" explicitly.

#### Story 2

As an operator author, I should be able to nest any compound constraint(s)
within a parent compound constraint.

#### Story 3

As an operator author, I should be able to build operator bundles that "work"
with both older and newer versions of OLM. I understand that "work" might mean
different things depending on OLM version; obviously older OLM versions cannot
compound constraints.

#### Story 4

As a cluster administrator using OLM, I should receive the same information
about successful and failed resolution as with current "simple" constraints.


### Implementation Details/Notes/Constraints

**Note**: see ["Implementation History"](#implementation-history) for proof-of-concept info.

I propose the new [`olm.constraint` property][arb-constraints] fields `all`, `any`, and `not`
to specify conjunction, disjunction, and negation compound constraints, respectively. Their value 
type is a struct containing a list of constraints such as GVK, package, or any compound constraint 
listed above under a `constraints` key. This new `olm.constraint` key should be defined in 
`dependencies.yaml` but can also be defined in `properties.yaml`.

I also propose that `olm.gvk.required` and `olm.package.required` are redefined as
`gvk` and `package` fields to align with other properties under `olm.constraint`:

```yaml
- type: olm.constraint
  value:
    failureMessage: Package bar v1.0.0+ is needed for...
    package:
      name: bar
      versionRange: '>=1.0.0'
- type: olm.constraint
  value:
    failureMessage: GVK Buf/v1 is needed for...
    gvk:
      group: bufs.example.com
      version: v1
      kind: Buf
```

These new schemas are equivalent to the existing schemas, and can be used interchangeably
at the top level; the old schema keys cannot be used within a compound constraint.

The implementation of compound constraints will primarily live in the OLM codebase,
since they are properties that the resolver understands. Some parsing details
will be implemented in operator-registry for registry components that need to know
about dependencies. The resolver will continue to function as it does now,
with the new compound constraints being atomic predicates that are evaluated in whole
against an installable object's properties.

#### Conjunction and disjunction

These compound constraint types are evaluated following their logical definitions. 

**Note**: Before proceeding, it is important to keep in mind that each constraint will be solved
by a single operator, not multiple. In order to define multiple operators you must specify 
multiple `olm.constraint` values within the dependencies.yaml. 

This is an example of a conjunctive constraint of two packages and one GVK,
i.e. they must all be satisfied by installed bundles:

```yaml
type: olm.constraint
value:
  failureMessage: All are required for Baz because...
  all:
    constraints:
    - failureMessage: Package bar is needed for...
      package:
        name: bar
        versionRange: '>=1.0.0'
    - failureMessage: GVK Buf/v1 is needed for...
      gvk:
        group: bufs.example.com
        version: v1
        kind: Buf
```

This is an example of a disjunctive constraint of three versions of the same GVK,
i.e. at least one must be satisfied by installed bundles:

```yaml
type: olm.constraint
value:
  failureMessage: Any are required for Baz because...
  any:
    constraints:
    - gvk:
        group: foos.example.com
        version: v1beta1
        kind: Foo
    - gvk:
        group: foos.example.com
        version: v1beta2
        kind: Foo
    - gvk:
        group: foos.example.com
        version: v1
        kind: Foo
```

#### Nested compound constraints

A nested compound constraint, one that contains at least one child compound constraint
along with zero or more simple constraints, is evaluated from the bottom up following
the procedures described for each above.

This is an example of a disjunction of conjunctions, where one, the other, or both
can be satisfy the constraint.

```yaml
type: olm.constraint
value:
  failureMessage: Required for Baz because...
  any:
    constraints:
    - all:
        constraints:
        - package:
            name: foo
            versionRange: '>=1.0.0'
        - gvk:
            group: foos.example.com
            version: v1
            kind: Foo
    - all:
        constraints:
        - package:
            name: foo
            versionRange: '<1.0.0'
        - gvk:
            group: foos.example.com
            version: v1beta1
            kind: Foo
```

The maximum raw size of an `olm.constraint` is 64KB to limit resource exhaustion attacks.
See [this issue][json-limit-issue] for details on why size is limited and not depth.
This limit can be changed at a later date if necessary.

#### Negation

Negation is worth further explanation, since at first glance its semantics
are unclear in this context. The negation instructs the resolver to remove 
any possible solution that includes a particular GVK, package at a version, 
or satisfies some child compound constraint from the result set.

If you define a negation constraint on a resource, then that resource cannot 
be used as a solution. This can be useful if you want to introduce a constraint 
that cannot be solved by a specific GVK or bundle. In almost all instances, the
negation constraint should be used as a nested constraint as it will not restrict 
anything when placed in the root. 

**Remember, negation is only used to say "this package cannot be used as a 
solution". It does not mean that the bundle is restricted from successfully 
installing if the negated package is already installed on the cluster**

The most common usecase for negation is excluding a certain package/version of a package
from providing a GVK that your operator depends on. In the below example, if the only package
providing the `bufs.example.com/v1` GVK was named `foo` with version `>=v1.0.0` it would
be removed from the solution and the constraints would be considered unsolvable.

```yaml   
type: olm.constraint
value:
  message: All are required because...
  all:
    constraints:
    - all:
        constraints:
        - failureMessage: GVK Buf/v1 required for...
          gvk:
            group: bufs.example.com
            version: v1
            kind: Buf
    - not:
        constraints:
        - failureMessage: Package foo version >=1.0.0 cannot be required for...
          package:
            name: foo
            versionRange: '>=1.0.0'
```

While less useful, here is an example of a negation constraint of one version of 
a GVK under the root. Again, this says that this GVK cannot be provided by any bundle 
in the result set. Since there are no other constraints, anything will solve this
constraint and thus it will be installable:

```yaml
type: olm.constraint
value:
  failureMessage: Cannot be required for Baz because...
  not:
    constraints:
    - gvk:
        group: foos.example.com
        version: v1alpha1
        kind: Foo
```

#### Changes to OLM's resolver

The [resolver package][olm-resolver-primitives] supplies the primitives "and" and "or" already,
and uses them to combine simple GVK and package constraints into a predicate expression.
Compound constraints would simply extend the usage of these primitives in the nested
case, and add the "not" primitive.

### Risks and Mitigations

- With the possibility for nesting comes the increased possibility that two or more nestings
will logically conflict and result in an unsatisfiable expression.
  - There is no quick way to determine whether an expression is unsatisfiable
  other than by using a SAT solver, which the resolver uses internally.
  Compound constraints must be vetted by hand.
- Compound constraint nesting exposes an exponentiation attack vector, where a malicious
bundle author adds deep nesting to a bundle resulting in long resolver pauses.
  - Tools that work with bundle constraints _must_ limit the depth of nesting of 10 levels.
  - Typically bundles are vetted by catalog maintainers, so deep nesting can be caught
  by automation prior to inclusion in a catalog.


## Design Details

### Test Plan

- operator-registry unit and e2e tests.
- operator-lifecycle-manager unit and e2e tests.
  - Both API and resolver unit tests.

### Graduation Criteria

This feature is in GA upon release.

#### Examples

See ["Implementation Details/Notes/Constraints"](#implementation-details-notes-constraints).

##### Removing a deprecated feature

- Announce deprecation and support policy of the existing feature
- Deprecate the feature

### Upgrade / Downgrade Strategy

Upgrading OLM will result in the same resolution results for bundles with top-level
GVK and package constraints.

Downgrading OLM on a cluster with catalogs containing compound constraints will
not cause bundle installation errors, but the resolver will ignore compound
constraint properties. See [version skew strategy](#version-skew-strategy) for details.

### Version Skew Strategy

Older versions of OLM will ignore unknown bundle properties, i.e. `olm.constraint`'s,
so compound constraints will be ignored.
Therefore the same bundle will "work" between OLM versions, even though `olm.constraint`'s
containing compound constraints will not be interpreted on bundles.

## Implementation History

Proof of concept:
- operator-registry: https://github.com/operator-framework/operator-registry/pull/814
- operator-lifecycle-manager: https://github.com/operator-framework/operator-lifecycle-manager/pull/2418

## Drawbacks

- Runtime penalty for complex boolean expressions in the resolver.

## Alternatives

### Arbitrary constraints

Compound constraints could be expressed via an [arbitrary constraint][arb-constraints] expression language,
depending on its implementation and what data gets passed to it. It could look something like

```yaml
- type: olm.constraint
  value:
    cel:
      rule: 'properties.exists(p, p.gvk.provides("v1", "example.com", "Foo")) && properties.exists(p, p.package.provides("bar", ">=1.0.0"))'
```

This syntax is more verbose. It also may have greater runtime penalties because of arbitrary constraint
parsing overhead. This overhead is unnecessary because GVKs and packages (the only two currently available
compound sub-constraints) are well-defined properties on bundles already. Compounding arbitrary constraints
will be possible once implemented as well, so constraints other than those for GVKs/packages are compound-able.

### OLM chooses to interpret old or new constraint schema

Instead of OLM interpreting both old (`olm.{gvk,package}.required`) and new (`olm.constraint`)
schemas simultaneously, OLM could make a choice:

1. If at least one `olm.constraint` is present, interpret only `olm.constraint` properties
and ignore all other constraint properties.
1. Else, interpret top-level constraints as currently defined.

Operator authors could then write bundles that both older and newer versions of OLM support.
I only propose giving OLM the ability to choose in this case because older OLM
versions ignore unknown properties like `olm.constraint`, behavior that should not
be modified; however newer versions could be built to smoothly transition constraint types.
I am not advocating that future OLM versions should ignore unknown `olm.constraint` value schemas,
which by design must result in error emission.

This approach has a UX flaw: old constraint schemas no longer work declaratively with
new OLM versions, which is confusing and goes against the effort to make operator distribution declarative.


[bundles]:https://olm.operatorframework.io/docs/tasks/creating-operator-bundle/
[properties]:/enhancements/properties.md
[olm]:https://olm.operatorframework.io
[dc]:/enhancements/declarative-index-config.md
[olm-resolver]:https://github.com/operator-framework/operator-lifecycle-manager/tree/5812e0b/pkg/controller/registry/resolver
[olm-resolver-primitives]:https://github.com/operator-framework/operator-lifecycle-manager/tree/5812e0b/pkg/controller/registry/resolver/cache/predicates.go
[arb-constraints]:https://github.com/operator-framework/enhancements/pull/91
[json-limit-issue]:https://github.com/golang/go/issues/31789#issuecomment-538134396
