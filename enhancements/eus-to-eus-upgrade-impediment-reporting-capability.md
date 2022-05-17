---
title: eus-to-eus-upgrade-impediment-reporting-capability

authors:
  - "@bentito"
reviewers:
  - TBD
  - "@camilamacedo86"
  - "@gallettilance"
approvers:
  - TBD
creation-date: 2022-05-17
last-updated: 2022-05-17
status: provisional
see-also:
  - "/enhancements/none.md"
replaces:
  - "/enhancements/none.md"
superseded-by:
  - "/enhancements/none.md"
---

# eus-to-eus-upgrade-impediment-reporting-capability


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

A cluster upgrade between two versions (for instance from one EUS version to
a later EUS version) may be hampered by critical, operator-delivered, services
not being able to be upgraded.

## Motivation

Customers using X and Y and Z Red Hat Operators need to know that if they are using
specific versions of each operator starting at certain OCP version, can they do a 
successful EUS to EUS upgrade with no manual steps.


### Goals

A report showing impediments we want to determine for each operator and display:

1. [Green] Can the operator remain the same version while OCP upgrades around it
min, max OCP version is encompassed by the proposed OCP version upgrades.
maxOCPVersion isn’t violated by the proposed upgrades.
2. [Green] Can the operator be updated (auto-update by OLM) along the same channel as OCP progress through each version.
3. [Orange] Can the operator be updated to a version for each of OCP, but one, or more, 
versions is on a different channel thus requiring manual intervention.
4. [Red] Operator won’t be able to work on the proposed upgrade path
Operator deprecated along proposed OCP upgrade path (not released, etc.)


### Non-Goals

[None for now]

## Proposal

Using operator metadata such as OCP version label, max OCP version label and presence
of the operator in various versions of OCP determine report values. Render the report
values for viewing by an audience of users and operator authors.

---
Please only comment on this RFE above this line, below is simply template values for now
---

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
be a good place to talk about core concepts and how they relate.

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

