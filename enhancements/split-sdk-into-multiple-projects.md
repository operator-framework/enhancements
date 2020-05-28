---
title: split-sdk-into-multiple-projects
authors:
  - "@jesusr"
reviewers:
  - "@estroz"
  - "@hasbro17"
  - "@joelanford"
  - "@fabianvf"
  - "@shawn-hurley"
approvers:
  - "@estroz"
  - "@hasbro17"
  - "@joelanford"
  - "@fabianvf"
  - "@shawn-hurley"
creation-date: 2020-05-18
last-updated: 2020-05-27
status: implementable
see-also:
replaces:
superseded-by:
---

# Split SDK into multiple projects

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [X] Design details are appropriately documented from clear requirements
- [X] Test plan is defined
- [X] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

1. How will separate repos affect the SDK website and docs in general?
   * What if each individual repo maintains their own docs? Then have a
     pipeline sort of thing that would regenerate the entire doc if there are
     any changes
     * Could this be an intern project?
   * How do versions work? netlify handles it
   * Only other thought is to have a single website docs repo
1. What do we do with other shared tooling, CI, test helpers, etc. (e.g.
   changelog generator)? Do we need a separate repo for that?

## Summary

This is the overarching enhancement that will plan how to split functionality
out of the current operator-sdk project into separate projects. The initial
thought on component breakdown mirrors the kubebuilder and controller-runtime
projects:

1. Common libraries and operator building blocks
1. Project scaffolding and the main SDK CLI
1. Helm operator library and scaffolding plugin
1. Ansible operator library and scaffolding plugin

## Motivation

The motivation is to avoid a breaking change in one component causing a major
version bump for all other unrelated components. Another motivation is to make
the projects consumable by other projects as libraries. A benefit would be
better testing and easier to contribute to the smaller projects.

While there are a few complexities with having separate repos, like PR
coordination between repos, or just the working complexity of having to point
at different repos to pick up your dep changes, these are obstacles we can
overcome. They also aren’t that much different from other dependencies we
work with.

### Goals

Split the operator-sdk repo into 4 separate repos:

1. CLI and Go Operator
1. SDK library
1. Helm operator
1. Ansible operator

This SDK library split would be a requirement for a 1.0 release. The Ansible
and Helm operators could go afterwards if need be.

### Non-Goals

* We will not solve the downstream SDK in this enhancement.
* We will not solve all of the items required for a 1.0 release

## Proposal

Split the operator-sdk repo into 4 separate repos as outlined in the Goals
section above.

1. CLI and Go Operator
   * contains the Go operator code
   * contains the CLI code
   * related scaffolding
   * lives in operator-framework/operator-sdk
   * will have a deprecation policy
     * we can deprecate in minor versions; removal becomes a major version bump
1. SDK library
   * contains utility code like package status, status condition, annotated
     watcher, etc.
   * this is something that operator projects (including Helm and Ansible
     operators) would import
   * depends on controller-runtime
   * lives in operator-framework/sdk-lib (or some other name TBD)
   * this library should probably *not* be 1.0; it's most likely to change often
   * use Go apidiff tool to determine major version bumps
1. Helm operator
   * contains Helm operator code
   * related scaffolding
   * lives in operator-framework/helm-operator
1. Ansible operator
   * contains Ansible operator code
   * related scaffolding
   * lives in operator-framework/ansible-operator

![Image of project dependencies](split-sdk-into-multiple-projects-deps.png)

### Tasks

1. Triage pkg directory to determine what goes to which repo
1. Use git commands to extract directories or files with history into these
   new repos. Exact commands will be TBD.
1. Evaluate if making a separate repo for the hack directory is necessary or
   if we just copy it to each of the 4 repos.
1. CLI and Go Operator
   * keep operator-framework/operator-sdk
   * cmd, internal directories remain here
   * prior to 1.0 release, review entire CLI; axe all legacy support for
     existing project layout
   * consider making `generate csv` separate like `controller-gen` to support
     OLM's plan for CSVless bundles (could be done later)
1. SDK library
   * create new repo operator-framework/sdk-lib
   * export the directories / files based on task 1 triage.
1. Helm operator and plugin
   * create new repo operator-framework/helm-operator
   * export the pkg/helm tree to the new repo at the top level maintaining
     history
     * Deprecate pkg/helm and replace with [Joe's helm repo](https://github.com/joelanford/helm-operator)
1. Ansible operator and plugin
   * create new repo operator-framework/ansible-operator
   * export the pkg/ansible tree to the new repo at the top level
     maintaining history

### Implementation Details/Notes/Constraints [optional]

See above

### Risks and Mitigations

The biggest risk is the instability that will occur while the projects are
split apart. Once all the projects are in place things will come back together.

## Design Details

### Test Plan

The ultimate test is that e2e tests and manual testing of the SDK binary will
be used to determine it remains functional. Each repo will contain their own
unit test suites and where appropriate e2e tests.

### Graduation Criteria

N/A

### Upgrade / Downgrade Strategy

N/A

### Version Skew Strategy

N/A

## Implementation History

20200527 - This document is created.
20200518 - Initial Google Document created to do initial set of review.

## Drawbacks

N/A

## Alternatives

N/A
