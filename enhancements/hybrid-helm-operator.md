---
title: enable-hybrid-helm-operators
authors:
  - "@fabianvf"
  - "@varshaprasad96
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2021-05-10
last-updated: 2021-05-12
status: implementable
see-also:
replaces:
superseded-by:
---

# enable-hybrid-helm-operators

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

1. How do we handle migrations for existing Helm-based Operators? For the first iteration
   requiring users to manually migrate charts and watches.yaml changes should be enough,
   but longer term we may want to support a command to automatically migrate this over,
   perhaps as an initialization flag to `operator-sdk init`, similar to how the `--helm-chart`
   argument works for the helm plugin.
1. Where should the helm library code live? Likely it should either go in `operator-frameowkr/operator-lib`,
   but it may make sense for it to have its own repository.
1. How would interaction between Helm and Go apis work? Is it possible to generate a `types.go` for the Helm
   apis that would allow the Go code to interact with them easily?

## Summary

Allow helm-operator users to extend their operators using Golang. We would accomplish
this by exposing all the functionality that the helm-operator has via a library
and scaffolding out code that is equivalent to the default behavior of the helm-operator.
The user could then modify that code as they see fit, effectively turning their helm-based
operator into a Golang one, but leaving a few utilities specific to helm-operators available
for convenience.


## Motivation

Helm is quite limited compared to Ansible or Golang, the other two technologies that
the Operator SDK currently supports. Once a helm-operator author reaches the limits
of what they can accomplish with Helm and begins to need additional functionality, the
only choice they have is to throw away what they have and duplicate it with another
technology. Even then, maintaining a compatible API is non-trivial as the helm-operator
provides functionality that the user would need to reimplement. Rather than starting over,
a hybrid operator approach would allow them to implement advanced functionality in
Golang while leaving the existing helm functionality in place.

### Goals

- Provide a library containing some of the helper logic for interacting with Helm and
  controller-runtime
- Scaffold out whatever code does not fit easily with the library
- If the user changes nothing in the code, their operator should behave exactly as it
  did when it was a plain helm-operator.
- The user should be able to scaffold APIs in either Golang or Helm
- In the end, we are trying to keep the ease of use of Helm-based operators, while adding
  the extensibility of Golang-based operators.

### Non-Goals

- We will not continue to manage the hybrid operator in any way, once it is converted
  it is essentially a Go-based operator and so will not make use of a base image with
  dependencies, automatic upgrades of helm versions, or any other conveniences provided
  by the base image. The user will need to manage and upgrade their code by hand.

## Proposal

There are 4 stages to this proposal:

1. Joe Lanford has made significant progress on this already, in the
   [joelanford/helm-operator][helm-operator-repo] repository. As a first step, we will
   migrate this code to a new repository, `operator-framework/helm-operator-plugins`.
1. We will write a new SDK plugin in this repo that can scaffold out a hybrid operator. As a first
   iteration we can just scaffold out most of the code that is currently in the
   [helm-operator][helm-operator-repo] (though the exact structure of the code will likely change)
   and use that to generate what we ship in the base image.
1. We will break as much code as we reasonably can out of the scaffolding and into a
   utility library, to minimize the amount of code the user will need to maintain. This
   library could either be the existing operator-lib package, or a new, helm-operator specific
   package.
1. We will move the current helm-operator plugin from the `operator-sdk` to the new
   `operator-framework/helm-operator-plugins` repository.

### User Stories

#### Story 1

As a Helm-based operator author, I want to add additional programmatic backup and restore
functionality to my operator, without starting over with a completely different type of operator.

#### Story 2

As a Helm-based operator author, I want to change core behavior of the reconciliation loop
that is not currently exposed in the watches.yaml

#### Story 3

As a prospective author of a Helm-based operator, I want to know that if I invest time and
effort creating a Helm-based operator, I won't need to start over in order to create a higher
level operator.

#### Story 4

As a Golang-based operator author, I would like to leverage existing helm charts for deploying
pieces of my application, without sacrificing the flexibility of Go.


### Implementation Details/Notes/Constraints

#### Scaffolding plugin for the SDK

The basic idea behind the hybrid Helm/Go operator plugin is that we will scaffold out a Go
operator project structure, with all of the code bits from the helm-operator project filled in,
such that building the project without adding any APIs or modifying any code will result in the
same container as what is shipped in the helm-operator base image.

Because the driving need behind this functionality is the ability to customize any piece of
the Helm-based operator, we will scaffold out every bit of code that is not accessible via
a library, and err on the side of scaffolding more than we need to.

For the first iteration of the hybrid Helm/Go operator plugin, we will scaffold out most of the
configuration and core reconciliation logic that is currently in the
[helm-operator repository][helm-operator-repo], but
with a `main.go` that is more in line with a typical Go-based operator, including the kubebuilder
scaffolding markers that make it easy to add pure Go-based apis. The major difference between the
scaffolded `main.go` for a hybrid operator vs a pure Go-based operator is that it will include the
logic for loading/parsing the `watches.yaml` and handling the reconciliation of resources defined there.
This will provide maximal
flexibility for early adopters of the pre-stable plugin. This also matches the functionality that
current users of the [helm-operator repository][helm-operator-repo] have, as the way they have
customized the behavior of the `helm-operator` binary is by forking the repository entirely and
making the modifications in their fork.

Examples of code that will likely have to remain scaffolded in the long-term are:
1. `pkg/watches/watches.go`: Based on current usage of the [helm-operator repository][helm-operator-repo],
    being able to modify the internal structure of the `watches.yaml` configuration file by adding options
    or changing their handling is likely to be a long-term need.
1. `internal/cmd/run/cmd.go`: Similarly to the `watches.go` file, users will need to have the
   ability to freely modify the arguments the `helm-operator` binary takes.
1. `pkg/reconcile/reconciler.go`: There are pieces here that could be moved to the library, but
   the bulk of the core logic of the reconcile loop should be exposed so that the users can
   modify it as they see fit without having to copy/reimplement all of it.

#### Library for helm functionality

Over time, as we gather feedback and observe the actual usage of the hybrid plugin, we should
slim down the scaffolded code and move library-like functionality that does not commonly require
modification into a library. The goal here is to maximize convenience for the users, but that
convenience cannot come at the cost of flexibility.

Examples of code that will likely be broken out into a library are:
1. `pkg/client`: The clients defined here are largely made of code that would be generally useful
   for any operator author that is interested in using Helm as part of their reconciliation loop,
   not just hybrid operator authors.
1. `pkg/annotation`, `pkg/values`, `pkg/manifestutil`, `pkg/hook`: These packages are mostly
   small, exported interfaces. Effectively these are already formatted and prepared for usage as
   a library.

#### The `helm-operator` binary and base image

The hybrid Helm/Go operator is a Go-based operator with scaffolded/library code that makes its
default configuration identical to a Helm-based operator. Rather than relying on a base image,
as a Helm-based operator does, the code for the `helm-operator` binary will be scaffolded and
the scaffolded Dockerfile will build it.

For a time, there will be some code duplication between the operator-sdk and the
helm-operator-plugins repository. Eventually, we will move the helm-operator plugin, binary, and
base image out of the operator-sdk and into helm-operator-plugins, at which point the plugin will
just be imported by the SDK.

#### The Makefile

The Makefile scaffolded for the hybrid Helm/Go operator would ideally be the same as the Makefile
for a standard Go-based operator. It is possible slight modifications may need to be made to
accomodate specific to Helm-based operators. The modifications should become apparent during the
implementation of this proposal and should not require significant time or effort.


#### Using both Go and Helm implementations of `operator-sdk create`

The operator-sdk already supports running commands that do not match the PROJECT layout, by passing the
`--plugins` flag directly. We would not need any additional functionality, we simply need to ensure
that the directory structure for the hybrid Helm/Go operator project is compatible with both the Golang
and Helm plugins' implementation of the `create` subcommands, and that all relevant files have the markers
necessary for both the Helm and Go plugins to insert scaffolded code for new APIs.

Example usage of both plugins to create APIs might look like:

##### init

```bash
$ operator-sdk init --plugins hybrid-helm-go.sdk.operatorframework.io/v1
```

This would initialize a Go project with a `main.go` that contains the markers for extending with
Go-based APIs, but also all of the logic necessary to parse and use a `watches.yaml`. It would also
scaffold the base `watches.yaml` and `helm-charts` that the Helm SDK plugin would initialize.

##### Create Go API
```bash
$ operator-sdk create api --plugins go.kubebuilder.io/v3 --group ship --version v1beta1 --kind Frigate
```

This would behave exactly as normal, scaffolding out code under `api` and `controllers`, as well as
modifying the deployment manifests in `config`.

##### Create Helm API
```bash
$ operator-sdk create api --plugins helm.sdk.operatorframework.io/v1 --group ship --version v1beta1 --kind Trireme
```

This would add the proper entry to `watches.yaml`, update the RBAC rules in `config`, and initialize a new helm
chart under `helm-charts`. Due to the logic included in the `main.go`, additional scaffolded code, and helm library
imports, building and running the operator at this point would automatically set up reconciliation for this resource,
exactly as it would for a standard Helm-based operator.

### Risks and Mitigations

One major downside to this approach is that it moves all of the code for Helm-based operators into scaffolding.
This makes contributing/understanding where code is coming from much more difficult. As time goes on and
more functionality is moved from scaffolding to libraries the impact will be lessened. Scaffolding out
the base helm-operator code and leaving it in-tree could make it less confusing to track down where certain
behaviors are coming from for users, but will not impact the ease of contribution.

Additionally, we will need to ensure that the integration testing in place for Helm-based operators is
thorough enough that we can avoid unknowingly introducing breaking changes for existing users of Helm-based
operators as we change how the code is built and extend it to support this use-case.

We will also be maintaining two implementations of the helm-operator binary, base image, and plugin initially.
We should prioritize moving to the new repository as quickly as possible to prevent code drift between
helm-operator-plugins and operator-sdk.

## Design Details

### Test Plan

Much of the code already has testing in place, but we will still need to add tests to ensure that the
`create` subcommands for Helm and Go based operators work side-by-side.

### Graduation Criteria

In phase 1 and 2 of this plan (moving the code from the [helm-operator repo][helm-operator-repo] to an
operator-sdk scaffolding plugin), the hybrid Helm/Go operator plugin should be considered a highly
unstable dev preview. As functionality is broken out and the helm library fills out and churn decreases,
we will consider moving to alpha status.

## Implementation History

## Drawbacks

This adds significant complexity to the way the helm-operator binary and base image are built.

## Alternatives

