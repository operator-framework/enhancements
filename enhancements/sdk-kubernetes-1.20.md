---
title: Support Kubernetes 1.20 in Operator SDK
authors:
  - "@bharathi-tenneti"
reviewers:
  - "@estroz"
  - "@camilamacedo86"
  - "@jmrodri"
approvers:
  - "@joelanford"
  - "@camilamacedo86"
  - "@jmrodri"
creation-date: 2020-09-25
last-updated: 2020-09-25
status: provisional
see-also:
  - "https://github.com/kubernetes/sig-release/tree/master/releases/release-1.20"
replaces: []
superseded-by: []
---

# Support Kubernetes 1.20 in Operator SDK

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] ~~Graduation criteria for dev preview, tech preview, GA~~ (N/A)

## Open Questions


## Summary

Operator SDK consists of many components (project scaffolding, OLM integrations, scorecard, helm and
ansible operators, and a Go library for operator authors) that have interactions with Kubernetes
clusters. The goal of this enhancement is to upgrade ALL components of Operator SDK to be compatible
with Kubernetes 1.20.


## Motivation

Operator SDK needs to support the latest features of Kubernetes since that operator developers expect to be able to use them shortly after the release is made.

## Dependencies

- [external] Require controller-runtime release that supports 1.20
- [external] Require Helm release that supports 1.20
- [external] Require kubebuilder release that scaffolds projects using 1.20
- [internal] Require operator-framework/api release that supports 1.20


### Goals

- Operator SDK’s dependencies are upgraded to use Kubernetes 1.20.
- All e2e tests run during Operator SDK CI are upgraded to test against Kubernetes 1.20.
- New projects scaffolded with Operator SDK are upgraded to use Kubernetes 1.20 dependencies.
- New projects scaffolded with Operator SDK should generate v1 CRDs by default.


### Non-Goals

- Feature development and bug fixes that are not directly related to Kubernetes 1.20 support. The
  rubric here is, if after bumping `go.mod` dependency versions:
  - Will the test suite fail CI without this change?
  - Will ignoring this change cause a regression?

  If the answer to these questions is no, then it is out of scope for this enhancement.
  - Unrelated bug fixes can be done before or after this enhancement is implemented.
  - Integration of new features introduced in 1.20 will be done in follow-on PRs.

- SDK Go version bump to 1.15
  - Risk: users are often slow to upgrade go versions, but upstream deps might use go 1.15 features.
  - Mitigation: we will only bump go version to 1.15 if there are breaking changes (go 1.15 features) introduced by Kubernetes 1.20.

## Proposal

### User Stories

#### Upgrade controller-runtime to support Kubernetes 1.20

_GitHub: [kubernetes-sigs/controller-runtime][gh-controller-runtime]_

##### Graduation Criteria

- Bumping the Kubernetes dependency versions in go.mod from 1.18+ or 1.19+ to 1.20.
- Upgrading any relevant documentation to note support for and any breaking changes introduced by
  the bump to Kubernetes 1.20.
- Making any necessary changes to get CI to pass.

#### Upgrade controller-tools to support Kubernetes 1.20

_GitHub: [kubernetes-sigs/controller-tools][gh-controller-tools]_

##### Graduation Criteria

- Bumping the Kubernetes dependency versions in go.mod from 1.18+ or 1.19+ to 1.20.
- Upgrading any relevant documentation to note support for and any breaking changes introduced by
  the bump to Kubernetes 1.20.
- Making any necessary changes to get CI to pass.

#### Upgrade kubebuilder to support Kubernetes 1.20

_GitHub: [kubernetes-sigs/kubebuilder][gh-kubebuilder]_

##### Graduation Criteria

- Bumping the controller-runtime dependency version in go.mod file scaffolds to the version of
  controller-runtime that supports Kubernetes 1.20.
- Bumping the controller-gen dependency version in the Makefile scaffolds to the version of
  controller-gen that supports Kubernete 1.20.
- Bumping kubebuilder-declarative-pattern dependency to 1.20 to support kubebuilder.  
- Upgrading any relevant documentation to note support for and any breaking changes introduced by
  the bump to Kubernetes 1.20.
- Making any necessary changes to get CI to pass.


#### Upgrade operator-sdk to support Kubernetes 1.20

_GitHub: [operator-framework/operator-sdk][gh-operator-sdk]_

##### Graduation Criteria

- Bumping the controller-runtime, controller-tools, OLM, and Kubernetes
  dependency versions in go.mod to versions that support Kubernetes 1.20.
- Bumping the controller-runtime version in go.mod scaffolds to the controller-runtime version that
  supports Kubernetes 1.20.
- Updating e2e tests to use Kubernetes 1.20 where possible and create follow-on tasks for tests that
  cannot be updated (this often happens when we update before kind images we use have been updated)
- Making any necessary changes to get CI to pass.

#### Upgrade operator-framework/operator-lib to support 1.20

_GitHub: [operator-framework/operator-lib][gh-operator-lib]_

##### Graduation Criteria

- Bumping the controller-runtime version in go.mod scaffolds to the controller-runtime version that supports Kubernetes 1.20.
- Updating e2e tests to use Kubernetes 1.20 where possible and create follow-on tasks for tests that cannot be updated (this often happens when we update before kind images we use have been updated)
- Making any necessary changes to get CI to pass.

#### Upgrade operator-framework/operator-helm-lib to support 1.20

#### Upgrade operator-framework/operator-ansible-lib to support 1.20


[gh-controller-runtime]: https://github.com/kubernetes-sigs/controller-runtime
[gh-controller-tools]: https://github.com/kubernetes-sigs/controller-tools
[gh-kubebuilder]: https://github.com/kubernetes-sigs/kubebuilder
[gh-operator-sdk]: https://github.com/operator-framework/operator-sdk
[gh-operator-lib]: https://github.com/operator-framework/operator-lib


### Risks and Mitigations

- There are no major risks to users. While Kubernetes 1.20 does deprecate apiextensions.k8s.io/v1beta1 CRDs, it does not remove them. So users have plenty of notice to migrate their CRDs to apiextensions.k8s.io/v1.

- As always there is risk to the actual development and delivery timing of this feature, due to the
many upstream changes that are required to be completed before Operator SDK can support Kubernetes
1.20.

- In particular, as the Operator SDK release relates to downstream integrations (e.g. OpenShift), the
current OpenShift release timelines have feature freeze landing _RIGHTAFTER_ Kubernetes 1.20 GA.
Therefore barring changes to the OpenShift release dates, there is a near-certain risk that
OpenShift 4.7 will not be able to support Kubernetes 1.20, and consumers of this enhancement will
have to wait.

- Helm latest release 3.3.x, is only compatible with 1.15.x - 1.18.x versions with kubernetes.
  -  Risk: breaking code/chart API changes
      The current Helm version the SDK uses is v3.2.4, which also supports 1.15.x-1.18.x, so support is the same and isn’t a breaking change
  - Mitigation: do not bump Helm dep version

- The k8s 1.20 dep bump may occur in a new alpha plugin if any breaking changes to project scaffolds occur.
  - Risk: this plugin version will not be the SDK’s default plugin; users will have to enable an alpha plugin to get 1.20 support
  - Mitigation: None
  - Note: This is not a risk if no breaking changes occur between 1.19 and 1.20

## Design Details

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything that would count as
tricky in the implementation and anything particularly challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage expectations).


### Version Skew Strategy

As with all Kubernetes version upgrades, the question inevitably arises, “Can I use this new version
of Operator SDK with my existing Kubernetes cluster?”

The answer is that it depends. In general, the Kubernetes API server protocols are backwards
compatible, so the primary concerns are:

- Client library compatibility - Since controller-runtime  is still  pre-1.0,
  any release can cause a breaking change to client APIs that may require developer intervention to
  fix. The Operator SDK community documents any known breaking changes in its version migration
  guides.

- Server API support - As Kubernetes versions are released, old APIs are removed and new APIs are
  added. Therefore it is not possible for Operator SDK to make a definitive statement about the
  support for individual operators. As an example, operator developers using `k8s.io/api` may find
  that an API they use is no longer present in newer versions of Kubernetes. If this occurs,
  operator developers must either:

  - Remain on an older version of `k8s.io/api` (and therefore operator-sdk), OR
  - Update their operator to use a new version of the API. Note that this is not without risk. Using
    a newer version of an API may result in losing support for older Kubernetes versions in which
    that API did not yet exist.

The only other aspect of this enhancement that involves upgrading and downgrading is the choice to
use v1beta1 or v1 CRDs. controller-gen and operator-sdk already support configurable CRD generation
that gives users the ability to choose the version of CRD to generate.

