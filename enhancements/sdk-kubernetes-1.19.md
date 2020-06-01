---
title: Support Kubernetes 1.19 in Operator SDK
authors:
  - "@joelanford"
reviewers:
  - "@estroz"
  - "@camilamacedo86"
  - "@jmrodri"
approvers:
  - "@hasbro17"
  - "@jmccormick2001"
  - "@jmrodri"
creation-date: 2020-05-21
last-updated: 2020-05-21
status: provisional
see-also:
  - "https://github.com/kubernetes/sig-release/tree/master/releases/release-1.19"
replaces: []
superseded-by: []
---

# Support Kubernetes 1.19 in Operator SDK

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] ~~Graduation criteria for dev preview, tech preview, GA~~ (N/A)

## Open Questions

1. controller-runtime currently uses a released kubebuilder version to get the necessary tools to
   run envtest (kube-apiserver and etcd). This causes a chicken-and-egg problem because kubebuilder
   must be updated after controller-runtime, but yet controller-runtime needs Kubebuilder’s tool
   binaries to be the latest version of Kubernetes to test controller-runtime’s changes against.

   The current situation is that controller-runtime is running its e2e tests against an older
   version of Kubernetes than it actually depends on and supports.

   This is likely out-of-scope for this enhancement, but we should follow up on this in parallel.

2. In the same timeframe of this enhancement, we are also considering splitting the SDK repository
   into separate repositories. How do these changes impact each other? Does one block the other?

## Summary

Operator SDK consists of many components (project scaffolding, OLM integrations, scorecard, helm and
ansible operators, and a Go library for operator authors) that have interactions with Kubernetes
clusters. The goal of this enhancement is to upgrade ALL components of Operator SDK to be compatible
with Kubernetes 1.19.

Kubernetes 1.19 has some key changes relevant to Operator development:

- `apiextensions.k8s.io/v1beta1` is deprecated (and scheduled for removal in 1.22) in favor of
  `apiextensions.k8s.io/v1` ([#90673][90673]) [SIG API Machinery]

[90673]:https://github.com/kubernetes/kubernetes/pull/90673

## Motivation

Supporting Kubernetes 1.19 in operator-sdk is necessary to enable our community to continue using
the latest and greatest features of Kubernetes.

### Goals

- Operator SDK’s dependencies are upgraded to use Kubernetes 1.19.
- All e2e tests run during Operator SDK CI are upgraded to test against Kubernetes 1.19.
- New projects scaffolded with Operator SDK are upgraded to use Kubernetes 1.19 dependencies.
- New projects scaffolded with Operator SDK should generate v1 CRDs by default.

### Non-Goals

- Feature development and bug fixes that are not directly related to Kubernetes 1.19 support. The
  rubric here is, if after bumping `go.mod` dependency versions:
  - Will the test suite fail CI without this change?
  - Will ignoring this change cause a regression?

  If the answer to these questions is no, then it is out of scope for this enhancement.
  - Unrelated bug fixes can be done before or after this enhancement is implemented.
  - Integration of new features introduced in 1.19 will be done in follow-on PRs.

## Proposal

### User Stories

#### Upgrade controller-runtime to support Kubernetes 1.19

_GitHub: [kubernetes-sigs/controller-runtime][gh-controller-runtime]_

This story consists of:

- Bumping the Kubernetes dependency versions in go.mod from 1.18 to 1.19.
- Upgrading any relevant documentation to note support for and any breaking changes introduced by
  the bump to Kubernetes 1.19.
- Making any necessary changes to get CI to pass.

#### Upgrade controller-tools to support Kubernetes 1.19

_GitHub: [kubernetes-sigs/controller-tools][gh-controller-tools]_

This story consists of:

- Bumping the Kubernetes dependency versions in go.mod from 1.18 to 1.19.
- Changing the default generated CRD version in controller-gen from v1beta1 to v1.
- Upgrading any relevant documentation to note support for and any breaking changes introduced by
  the bump to Kubernetes 1.19.
- Making any necessary changes to get CI to pass.

#### Upgrade kubebuilder to support Kubernetes 1.19

_GitHub: [kubernetes-sigs/kubebuilder][gh-kubebuilder]_

This story consists of:

- Bumping the controller-runtime dependency version in go.mod file scaffolds to the version of
  controller-runtime that supports Kubernetes 1.19.
- Bumping the controller-gen dependency version in the Makefile scaffolds to the version of
  controller-gen that supports Kubernete 1.19.
- Upgrading any relevant documentation to note support for and any breaking changes introduced by
  the bump to Kubernetes 1.19.
- Making any necessary changes to get CI to pass.
- Potentially bump the default Webhook and CRD version to `v1` in the project Makefile. See
  [kubernetes-sigs/kubebuilder#1065](https://github.com/kubernetes-sigs/kubebuilder/issues/1065) for
  details.
- Potentially bump `kubernetes-sigs/kubebuilder-declarative-pattern` to use Kubernetes 1.19 (only if
  necessary to make CI pass. Refer to non-goals to decide whether this is in or out of scope.).

#### Upgrade helm to support Kubernetes 1.19

_GitHub: [helm/helm][gh-helm]_

This story consists of:

- Bumping the Kubernetes dependency versions in go.mod from 1.18 to 1.19.
- Making any necessary changes to get CI to pass.

#### Upgrade prometheus-operator to support Kubernetes 1.19

_GitHub: [coreos/prometheus-operator][gh-prometheus-operator]_

This story consists of:

- Bumping the Kubernetes dependency versions in go.mod from 1.18 to 1.19.
- Making any necessary changes to get CI to pass.

#### Upgrade operator-sdk to support Kubernetes 1.19

_GitHub: [operator-framework/operator-sdk][gh-operator-sdk]_

This story consists of:

- Bumping the controller-runtime, controller-tools, helm, prometheus-operator, OLM, and Kubernetes
  dependency versions in go.mod to versions that support Kubernetes 1.19.
- Bumping the controller-runtime version in go.mod scaffolds to the controller-runtime version that
  supports Kubernetes 1.19.
- Fix unmarshalling code in SDK that assumes CRDs are `v1beta1`.
- Updating e2e tests to use Kubernetes 1.19 where possible and create follow-on tasks for tests that
  cannot be updated (this often happens when we update before kind images we use have been updated)
- Making any necessary changes to get CI to pass.

[gh-controller-runtime]: https://github.com/kubernetes-sigs/controller-runtime
[gh-controller-tools]: https://github.com/kubernetes-sigs/controller-tools
[gh-kubebuilder]: https://github.com/kubernetes-sigs/kubebuilder
[gh-helm]: https://github.com/helm/helm
[gh-prometheus-operator]: https://github.com/coreos/prometheus-operator
[gh-operator-sdk]: https://github.com/operator-framework/operator-sdk

### Implementation Details/Notes/Constraints [optional]

N/A

### Risks and Mitigations

There are no major risks to users. While Kubernetes 1.19 does deprecate
`apiextensions.k8s.io/v1beta1` CRDs, it does not remove them. So users have plenty of notice to
migrate their CRDs to `apiextensions.k8s.io/v1`.

As always there is risk to the actual development and delivery timing of this feature, due to the
many upstream changes that are required to be completed before Operator SDK can support Kubernetes
1.19.

In particular, as the Operator SDK release relates to downstream integrations (e.g. OpenShift), the
current OpenShift release timelines have feature freeze landing _BEFORE_ Kubernetes 1.19 GA.
Therefore barring changes to the OpenShift release dates, there is a near-certain risk that
OpenShift 4.6 will not be able to support Kubernetes 1.19, and consumers of this enhancement will
have to wait until 4.7 to use it.

## Design Details

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything that would count as
tricky in the implementation and anything particularly challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage expectations).

### Graduation Criteria

N/A

### Upgrade / Downgrade Strategy

N/A

### Version Skew Strategy

As with all Kubernetes version upgrades, the question inevitably arises, “Can I use this new version
of Operator SDK with my existing Kubernetes cluster?”

The answer is that it depends. In general, the Kubernetes API server protocols are backwards
compatible, so the primary concerns are:

- Client library compatibility - Since controller-runtime and operator-sdk are still both pre-1.0,
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

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

None

## Alternatives

None

## Infrastructure Needed

None
