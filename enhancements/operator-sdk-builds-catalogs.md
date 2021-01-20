---
title: operator-sdk-builds-catalogs
authors:
  - "@estroz"
reviewers:
  - "@jmrordi"
  - "@tlwu2013"
approvers:
  - "@jmrordi"
  - "@tlwu2013"
creation-date: 2021-01-20
last-updated: 2021-01-20
status: provisional
---

# operator-sdk-builds-catalogs

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

Operator authors using `operator-sdk` may opt to build and host their own index images (catalogs)
containing their Operator's bundles instead of using a productization pipeline,
or at least would like to test their bundles collectively as a catalog.
`operator-sdk` already facilitates building and testing bundles, so it should
facilitate building catalogs from these bundles.

## Motivation

Canonically, `opm index add` is how a catalog is created and bundles are added to a catalog.
In fact, it is the only entrypoint into the world of managing catalogs, likely for adoption reasons and
because these processes are largely opaque and complicated, even though the spec is not.
While `opm` is intended for "power users" (read: those who need to manipulate catalog images),
certain subcommands like `index` and its related libraries can be used fairly easily by anyone
`operator-sdk` should expose functionality of `opm` in a limited and sanely-defaulted way
to enable building catalog images.

### Goals

- Expose `opm index add` functionality to `operator-sdk` users for all supported Operator types.
- Use existing libraries/`opm` itself to expose the desired functionality.
- Increase understanding of the Operator packaging lifecycle.

### Non-Goals

- Re-implement any part of the catalog management process.
- Add any catalog-related code or files to `operator-sdk` projects, ex. `index.Dockerfile`.
- Add any other functionality other than build a catalog locally (`opm index add`).

## Proposal

### User Stories

#### Build a catalog using `opm`

As an Operator developer, I want to easily build a catalog from my Operator's bundles
without having to know the details of _how_ it is built.

#### Understand what a catalog is

As an Operator developer, I should understand what a catalog is and why I would consider
building one. I also should understand how bundling and building a catalog ties into shipping my Operator.

### Implementation Details/Notes/Constraints

There are two ways of adding `opm index add` functionality to `operator-sdk`:
1. Create a `catalog-build` Makefile rule that calls `opm index add`.
1. Add a `operator-sdk catalog build` subcommand (discussed in the [alternatives](#subcommand-approach) section).

The Makefile approach is by far the best for a few reasons:
- Users have full control over which `opm` version to use.
- Users have full control over defaults for their project, including image tags.
- Users get experience using `opm`, but only the relevant part(s).
  - They can easily add more `opm` targets if they want.

The Makefile additions would look like:
```make
VERSION ?= 0.1.0
IMAGE_TAG_BASE ?= quay.io/example/my-operator
BUNDLE_IMG ?= $(IMAGE_TAG_BASE)-bundle:$(VERSION)

bundle-push:
	$(MAKE) docker-push IMG=$(BUNDLE_IMG)

OPM = ./bin/opm
opm:
  # Download `opm` from github.com/operator-framework/operator-registry/releases

BUNDLE_IMGS ?= $(BUNDLE_IMG)
CATALOG_IMG ?= $(IMAGE_TAG_BASE)-catalog:v$(VERSION)
catalog-build: opm
	$(OPM) index add --mode semver --tag $(CATALOG_IMG) --bundles $(BUNDLE_IMGS)

catalog-push:
	$(MAKE) docker-push IMG=$(CATALOG_IMG)
```

In addition, consider these changes to the current Makefile:

```diff
VERSION ?= 0.1.0
+IMAGE_TAG_BASE ?= quay.io/example/my-operator
-BUNDLE_IMG ?= controller-bundle:$(VERSION)
+BUNDLE_IMG ?= $(IMAGE_TAG_BASE)-bundle:v$(VERSION)

+bundle-push:
+	$(MAKE) docker-push IMG=$(BUNDLE_IMG)
```

Coupled with the suggested `opm` additions, these changes would allow target chaining like so:

```sh
export IMAGE_TAG_BASE=quay.io/example/my-operator
make bundle-build bundle-push catalog-build catalog-push
```

Resulting in both `quay.io/example/my-operator-bundle:v0.1.0` and `quay.io/example/my-operator-catalog:v0.1.0`
being built and pushed.

Check out this [PR](https://github.com/operator-framework/operator-sdk/pull/4406) for an implementation reference.

### Risks and Mitigations

The major risk, which has already been encountered, is an issue with an `opm` release.
In particular, problems with building a binary on a particular architecture may result
in no binary being released for that architecture. `operator-sdk` supports
amd64 (for linux and darwin platforms), arm64, ppc64le, and s390x; a binary targeting each
platform must exist for every `opm` release. Mitigation involves
1. Not upgrading `opm` versions scaffolded until an `opm` release is confirmed to support each platform.
1. Improving `opm`'s release process with a host-agnostic multi-platform build system like `docker buildx`.

Another risk, although minor, is confusion around which situations a catalog should be built.
This will be mitigated by documented examples demonstrating when building a catalog is the right
choice, and when the deployment entrypoint for their operator should be an existing catalog (pipeline).

## Design Details

### Test Plan

Add an integration test that builds a catalog using a bundle that `operator-sdk run bundle`
will use as a base index image.

### Upgrade / Downgrade Strategy

Users can change the `opm` version in their Makefile.

## Implementation History

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

### Subcommand approach

Adding a `catalog build` subcommand is possible by vendoring `operator-registry`'s index image building
library, used by `opm index add`. The current `opm index add` CLI flags could be copied verbatim
or slimmed down to the most useful subset of flags.

Benefits:
- `operator-sdk` can tailor CLI flags and help text to suit non-power users.
- Catalog building would be self-contained within `operator-sdk`.

Drawbacks:
- Vendoring the `opm` library would require `operator-sdk` be compiled with CGo enabled,
meaning a rewrite of our release process and parts of our CI pipeline (again).
  - New architectures (ex. Apple Silicon) would be non-trivial to add.
- While `operator-sdk` already vendors `operator-registry` for its bundle code generation library,
using the index image library increases the bug surface area.
  - Users would need to wait for `operator-sdk` to release patches before they can get bug fixes
- Wrapping `opm` in a subcommand further confuses users: should they be using `opm` or `operator-sdk catalog build`?
What are the differences? What if they graduate to "power user" and have to change their tooling setup? etc.
  - If more `opm` functionality was desired, new subcommands would need to be added, further confusing users.
