---
title: olm-optional-dependency-declaration
authors:
  - "@cdjohnson"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2021-09-22
last-updated: 2021-09-22
status: provisional
see-also:
  - "/enhancements/operator-dependency-resolution.md"
  - "/enhancements/operator-dependency-provisioning.md"
replaces:
  - ""
superseded-by:
  - ""
---

# OLM Optional Dependency Declaration

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]
- Who are the actual consumers of the dependency.  Do I have them right?  Are there more to consider?
- Should the depenendency intent be on all dependency types?  I was thinking olm.package only because this would give us a way to more easily identify the "thing" to provision from the sister provisinioning spec.


## Summary
All OLM-Enabled Operator dependencies today are described statically in 
both the ClusterServiceVersion (CSV) as a CRD or API dependency, or in the 
Operator bundle `metadata/dependencies.yaml` as an `olm.gvk` or `olm.package` dependency.  
OLM then uses this information to attempt to resolve to an existing operator or 
provision a new operator that satisfies the requirements.  When OLM privisions operators, it attempts to create a `Subscription` from an available `CatalogSource` (prioritized) and a `Channel` (also prioritized).  It therefore translates a `olm.gvk` and `olm.package` dependency to `CatalogSource`, `Package` and `Channel`.

There are many use cases, that it would be beneficial to describe dependencies that are non-required
in nature and provide a way to provision those dependencies on-demand.  Many other package managers have similar capabilities:

- **[apt](https://www.debian.org/doc/debian-policy/ch-relationships.html)**: 
  - Required: `Depends`:  Automatically Provisions.
  - Optional: `Recommends`, `Suggests`, `Enhances`
- **[npm](https://betterprogramming.pub/what-are-npms-optional-dependencies-and-when-should-we-use-them-796a6a964e73)**: 
  - Required: `dependencies`:  Automatically provisions. 
  - Optional: `optionalDependencies`
- **[rpm & yum](http://ftp.rpm.org/max-rpm/s1-rpm-depend-manual-dependencies.html)**:
  - Required: `Requires`, `BuildRequires`, `PreReq`: Yum has CLI options to introspect and auto-provision.
  - Optional: Not available.
- **[Homebrew](https://docs.brew.sh/Formula-Cookbook#specifying-other-formulae-as-dependencies)**:
  - Required: The default behavior requires the depedency and auto-provisions it.
  - Optional: `optional`, `recommended` (as hash modifiers to the dependency)
- **[Snap](https://snapcraft.io/docs/build-and-staging-dependencies)**:
  - Required: The only behavior.  Automatically provisions.  Includes a concept of a channel (track/risk/branch).
  - Optional: Not available
- **[pip](https://packaging.python.org/discussions/install-requires-vs-requirements/#install-requires-vs-requirements-files)**:
  - Required: `install_requires`.  Automatically provisions.
  - Optional: `extras`  

By nature, Optional dependencies cannot themselves be provisioned automatically.  The nature of Optional means the policy or condition can be ambiguous.  The best we can do, therefore, is to describe the potential dependency and allow the Operator to provision the operator on-demand in a consistent way.

Because Optional dependency provisioning is a Runtime decision, it’s difficult to model the intent of the consumer when mirroring the required images to the air-gapped (disconnected) registry.  With this in mind, Optional dependencies will be treated as Static dependencies by the mirroring toolign, and the Dynamic capabilitity will be limited to the resolution of an Operator’s dependencies and the Provisioning of those dependencies.


## Motivation

This specification simply extends the existing `dependencies` structure to allow adding additional information about the intent of the dependency, where the default is `required`, `auto-provision` and `mirror`.

This specification further describes how the current `dependencies` consumers implement the intent of the optional dependency.

### Goals
- Preserve default behavior and interoperability with older clients of the metadata.
- Simple and obvious configuration
- Clear descriptions of how the current consuming clients should behave which include:
  - Dependency Resolver
  - Provisioner (Installer)
  - Curated catalog collectors (e.g. `opm alpha diff`)
  - Related images collectors (e.g. `oc adm catalog mirror`)


### Non-Goals
- Methods for provisioning optional dependencies.

## Proposal
Add an `intent` property to the `olm.package` dependency object.

```yaml
dependencies:
- type: olm.package
  value:     
    packageName: prometheus
    version: >0.27.0
    intent: required|recommended|optional
- type: olm.gvk
  value:
    group: etcd.database.coreos.com
    kind: EtcdCluster
    version: v1beta2
```

### Intent Values

Summary:
-	`required`:  The default that requires the dependency to be present and auto-provisioned.
  - Same behavior as today.
  - OLM requires the dependency
  - OLM provisions the dependency if available.
  - OLM blocks provisioning of Operators if dependencies are not satisfied.
-	`recommended`:  The dependency is not required, but if available, will be provisioned by default.  Clients can override the default behavior.
-	`optional`: The dependency is not required, and if available, will not be provisioned by default.  Clients can override the default behavior.

#### required
The required dependency is the default.  When the `intent` is omitted or `required` is specified, the following behaviors are expected:
- Dependency Resolver: 
  - Dependency MUST be satisfied 
- Provisioner:
- `opm alpha diff`
- Image Collectors:

### User Stories [optional]

Detail the things that people will be able to do if this is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.

#### Story 1

#### Story 2

### Implementation Details/Notes/Constraints [optional]

What are the caveats to the implementation? What are some important details that
didn't come across above. Go in to as much detail as necessary here. This might
be a good place to talk about core concepts and how they releate.

### Risks and Mitigations

What are the risks of this proposal and how do we mitigate. Think broadly. For
example, consider both security and how this will impact the larger Operator Framework
ecosystem.

How will security be reviewed and by whom? How will UX be reviewed and by whom?

Consider including folks that also work outside your immediate sub-project.

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

Similar to the `Drawbacks` section the `Alternatives` section is used to
highlight and record other possible approaches to delivering the value proposed
by an enhancement.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources
started right away.

