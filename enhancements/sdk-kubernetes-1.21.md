---
title: Support Kubernetes 1.21 in Operator SDK
authors:
  - "@varshaprasad96"
reviewers:
  - "@estroz"
  - "@jmrodri"
approvers:
  - "@estroz"
  - "@jmrodri"
creation-date: 2021-06-10
last-updated: 2020-06-10
status: provisional
see-also:
  - "https://github.com/kubernetes/sig-release/tree/master/releases/release-1.21"
replaces: []
superseded-by: []
---

# Support Kubernetes 1.21 in Operator SDK

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] ~~Graduation criteria for dev preview, tech preview, GA~~ (N/A)

## Summary

Operator SDK consists of many components (project scaffolding, OLM integrations, scorecard, helm, ansible and java operators, and a Go library for operator authors) that have interactions with Kubernetes clusters. The goal of this enhancement is to upgrade ALL components of Operator SDK to be compatible with Kubernetes 1.21.

## Motivation

Operator SDK needs to support the latest features of Kubernetes since that operator's developers expect to be able to use them shortly after the release is made.

## Dependencies

- [external] Require controller-runtime release that supports 1.21+
- [external] Require Helm release that supports 1.21+
- [external] Require kubebuilder release that scaffolds projects using 1.21+
- [internal] Require operator-framework/api release that supports 1.21+
- [internal] Require operator-framework/operator-lib release that supports 1.21+
- [internal] Require OLM release that supports 1.21+

### Goals

- Operator SDK’s dependencies are upgraded to use Kubernetes 1.21.
- All e2e tests run during Operator SDK CI are upgraded to test against Kubernetes 1.21.
- New projects scaffolded with Operator SDK are upgraded to use Kubernetes 1.21 dependencies.
- New projects scaffolded with Operator SDK should generate v1 CRDs by default.
    - Though k8s 1.21 supports v1beta1 CRDs, they would be removed in the next k8s release. This is not a requirement, but can be done now to prepare for future releases to support k8s 1.22.

### Non-Goals

- Feature development and bug fixes that are not directly related to Kubernetes 1.21 support. The rubric here is, if after bumping `go.mod` dependency versions:
    - Will the test suite fail CI without this change?
    - Will ignoring this change cause a regression?

If the answer to these questions is no, then it is out of scope for this enhancement.
    - Unrelated bug fixes can be done before or after this enhancement is implemented.
    - Integration of new features introduced in 1.21 will be done in follow-on PRs.

## Proposal

### User Stories

#### Upgrade controller-runtime to support Kubernetes 1.21

_GitHub: [kubernetes-sigs/controller-runtime][gh-controller-runtime]_

##### Graduation Criteria

- Bumping the Kubernetes dependency versions in go.mod from 1.20+ to 1.21+.
- Upgrading any relevant documentation to note support for and breaking changes introduced by the bump to Kubernetes 1.21.
- Making any necessary changes to get CI to pass.

**Note**
Controller-runtime [0.9.0](https://github.com/kubernetes-sigs/controller-runtime/releases/tag/v0.9.0) has been released which supports k8s 1.21.

#### Upgrade controller-tools to support Kubernetes 1.21

_GitHub: [kubernetes-sigs/controller-tools][gh-controller-tools]_

##### Graduation Criteria

- Bumping the Kubernetes dependency versions in go.mod from 1.20+ to 1.21.
- Upgrading any relevant documentation to note support for and any breaking changes introduced by
  the bump to Kubernetes 1.21.
- Making any necessary changes to get CI to pass.

**Note**
Controller-tools [0.6.0](https://github.com/kubernetes-sigs/controller-tools/releases/tag/v0.6.0) has been released which supports k8s 1.21.

#### Upgrade kubebuilder to support Kubernetes 1.21

_GitHub: [kubernetes-sigs/kubebuilder][gh-kubebuilder]_

##### Graduation Criteria

- Bumping the controller-runtime dependency version in `go.mod` file scaffolds to the version of controller-runtime that supports Kubernetes 1.21.
- Bumping the controller-gen dependency version in the `Makefile` scaffolds to the version of controller-gen that supports Kubernete 1.21.
- Bumping kubebuilder-declarative-pattern dependency to 1.21 to support kubebuilder.  
- Upgrading any relevant documentation to note support for and any breaking changes introduced by the bump to Kubernetes 1.21.
- Making any necessary changes to get CI to pass.

#### Upgrade operator-sdk to support Kubernetes 1.21

_GitHub: [operator-framework/operator-sdk][gh-operator-sdk]_

##### Graduation Criteria

- Bumping the controller-runtime, controller-tools, OLM, and Kubernetes dependency versions in `go.mod` to versions that support Kubernetes 1.21.
- Bumping the controller-runtime version in go.mod scaffolds to the controller-runtime version that supports Kubernetes 1.21.
- Updating e2e tests to use Kubernetes 1.21 where possible and create follow-on tasks for tests that cannot be updated (this often happens when we update before kind images we use have been updated)
- Making any necessary changes to get CI to pass.

#### Upgrade operator-framework/operator-lib to support 1.21

_GitHub: [operator-framework/operator-lib][gh-operator-lib]_

##### Graduation Criteria

- Bumping the controller-runtime version in go.mod scaffolds to the controller-runtime version that supports Kubernetes 1.21.
- Updating e2e tests to use Kubernetes 1.21 where possible and create follow-on tasks for tests that cannot be updated (this often happens when we update before kind images we use have been updated)
- Making any necessary changes to get CI to pass.

#### Upgrade operator-framework/api to support 1.21

_GitHub: [operator-framework/api][gh-api]_

##### Graduation Criteria

- Bumping the controller-runtime version in go.mod scaffolds to the controller-runtime version that supports Kubernetes 1.21.
- Updating tests to use Kubernetes 1.21 where possible and create follow-on tasks for tests that cannot be updated (this often happens when we update before kind images we use have been updated)
- Making any necessary changes to get CI to pass.

#### Upgrade operator-framework/java-operator-plugins to support 1.21

_GitHub: [operator-framework/java-operator-plugins][gh-java]_

##### Graduation Criteria

- Bumping the apimachinery and kubebuilder version in `go.mod` to a version that supports Kubernetes 1.21.
- Updating e2e tests to use Kubernetes 1.21 where possible and create follow-on tasks for tests that cannot be updated (this often happens when we update before kind images we use have been updated)
- Making any necessary changes to get CI to pass.

[gh-controller-runtime]: https://github.com/kubernetes-sigs/controller-runtime
[gh-controller-tools]: https://github.com/kubernetes-sigs/controller-tools
[gh-kubebuilder]: https://github.com/kubernetes-sigs/kubebuilder
[gh-operator-sdk]: https://github.com/operator-framework/operator-sdk
[gh-operator-lib]: https://github.com/operator-framework/operator-lib
[gh-api]: https://github.com/operator-framework/api
[gh-java]: https://github.com/operator-framework/java-operator-plugins

### Risks and Mitigations

- There are no major risks to users with the update in Kubernetes bump. The breaking changes and required migration steps will be documented in Operator SDK [webiste](https://sdk.operatorframework.io). 

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

As with all Kubernetes version upgrades, the question inevitably arises, “Can I use this new version of Operator SDK with my existing Kubernetes cluster?”

The answer is that it depends. In general, the Kubernetes API server protocols are backwards compatible, so the primary concerns are:

- Client library compatibility - Since controller-runtime  is still  pre-1.0, any release can cause a breaking change to client APIs that may require developer intervention to fix. The Operator SDK community documents any known breaking changes in its version migration guides.

- Server API support - As Kubernetes versions are released, old APIs are removed and new APIs are added. Therefore it is not possible for Operator SDK to make a definitive statement about the support for individual operators. As an example, operator developers using `k8s.io/api` may find that an API they use is no longer present in newer versions of Kubernetes. If this occurs, operator developers must either:

    - Remain on an older version of `k8s.io/api` (and therefore operator-sdk), OR
    - Update their operator to use a new version of the API. Note that this is not without risk. Using a newer version of an API may result in losing support for older Kubernetes versions in which that API did not yet exist.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

None

## Alternatives

None

## Infrastructure Needed

None