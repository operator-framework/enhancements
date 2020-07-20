---
title: kubectl plugin for Operator management
authors:
  - "@dmesser"
reviewers:
  - "@joelanford"
  - "@ecordell"
  - "@krizza"
approvers:
  - "@joelanford"
  - "@ecordell"
creation-date: 2020-07-20
last-updated: 2020-07-20
status: implementable
see-also:
  - "https://github.com/joelanford/kubectl-operator"  
replaces:
  - ""
superseded-by:
  - ""
---

# kubectl plugin for Operator Lifecycle Manager

A `kubectl` plugin for managing Operators and Operator catalogs with an OLM enabled cluster.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

- Can `oc` simply use `kubectl` plugins?
- What are the distribution vectors of this plugin?
- How are downstream users of OpenShift going to get the plugin installed?
- What is the minimum OLM version the plugin should support?

## Summary

A `kubectl` plugin called `operator` allows for a low entrance barrier into Operator management
with OLM. It provides the convenience of a CLI utility with integrated validation, specific
switches, contextual help while relying on OLM on cluster to actually fulfill the requested operation.

It serves as an alternative option to API-driven or manifest-centric ways to interact with OLM.

## Motivation

In the light of reducing the API surface of OLM a more interactive, CLI-driven
interaction with OLM could further lower the learning curve for cluster administrators new to OLM.
By compartmentalizing many of the common use cases for OLM into distinct sub-commands perceived
complexity is reduced and human errors are minimized. Next to interactive use by a cluster administrator
the CLI could be used to ease construction of pipelines that interact with OLM.

### Goals

- the plugin covers the main uses cases of OLM to manage Operators and Operator catalogs
- the plugin provides command completion for common administrative shells
- the plugin aids in constructing pipelines for testing Operators
- the plugin can be downloaded from the [`krew`](https://github.com/kubernetes-sigs/krew-index/blob/master/plugins.md) index

### Non-Goals

- replace `opm`
- replace `operator-sdk` functionality for testing, packaging and release operators

## Proposal

The proposed functionality of the `operator` plugin for `kubectl` is outlined in the fictional man page below
which adheres to the [tldr](https://tldr.sh/) format:

### kubectl operator man page


`kubectl operator`

Plugin to manage Operators and Catalogs of Operators with OLM
Requires OLM to be installed.

  - Add a catalog to the cluster globally

    `kubectl operator catalog add my-catalog quay.io/my-org/my-catalog:v1.0`

  - Add a namespaced catalog to the cluster

    `kubectl operator catalog add -n my-namespace my-catalog quay.io/my-org/my-catalog:v1.0`

  - List all globally installed catalogs in the cluster

    `kubectl operator catalog list`

  - List all catalogs in a particular namespace

    `kubectl operator catalog list -n my-namespace`

  - Remove a catalog

    `kubectl operator catalog remove my-catalog`

  - Install an Operator globally

    `kubectl operator install my-operator`

    `kubectl operator install -m allnamespaces my-operator`

  - Install an Operator in a namespace

    `kubectl operator install -n my-namespace my-operator`

  - Install an Operator from my-catalog

    `kubectl operator install -c my-catalog my-operator`

  - Install an Operator watching a namespace

    `kubectl operator install -w my-namespace my-operator`

    `kubectl operator install -m singlenamespace -n my-namespace my-operator`

    `kubectl operator install -m singlenamespace -n my-namespace -w my-namespace my-operator`
    
  - Install an Operator in a namespace watching another namespace

    `kubectl operator install -m singlenamespace -n my-namespace -w another-namespace my-operator`

  - Install an Operator in a particular version

    `kubectl operator install my-operator.v1.1.2`

  - Install an Operator with automatic updates enabled

    `kubectl operator install -p auto my-operator`
  
  - Update an Operator to the next available version

    `kubectl operator update my-operator`

  - Update an Operator to the next available version

    `kubectl operator update my-operator`

  - Update an Operator to a particular version

    `kubectl operator update my-operator my-operator.v.1.2.0`

  - Uninstall an Operator

    `kubectl operator uninstall my-operator`

  - Uninstall an Operator and all its deployed workloads and CRDs

    `kubectl operator uninstall -r --yes-i-am-really-really-really-sure my-operator`

  - List installed Operators

    `kubectl operator list`
    `kubectl operator list installed`

  - List installed Operators in a namespace
    
    `kubectl operator list -n my-namespace`

   - List installed Operators from a particular catalog
    
    `kubectl operator list -c my-catalog`

  - List Operators available to install

    `kubectl operator list available`

  - List all available versions of a particular Operators available to install

    `kubectl operator list available my-operator`

  - List Operators available from a particular catalog

    `kubectl operator list available -c my-catalog`

  - List installed Operators watching this particular namespace

    `kubectl operator list serving -n my-namespace`

### Implementation Details/Notes/Constraints [optional]

tbd

#### Constraints

- the plugin does not expose the concept of `OperatorGroup` but handles as much as possible of that in the background
- the plugin 

#### Notes

The output format of all commands should adhere to `kubectl` behavior as closely as possible.
ASCII tables are the default with the option to provide YAML and JSON equivalents.

### Risks and Mitigations

1) Confusion with the `Operator` API

The upcoming Operator API pursues many of the same goals as this proposal but serves additional needs 
like driving UIs, GitOps workflows and other API-driven use cases.

2) Pending changes due to Operator scope discussions potentially eliminating some of the above subcommand switches.

3) Too much abstraction from underlying concepts, e.g. `OperatorGroup` or automatic updates and approvals.

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

