---
title: Catalog-Data-Transfer-Reduction
authors:
  - "@estroz"
reviewers:
  - "@ecordell"
  - "@dmesser"
  - "@kevinrizza"
approvers:
  - "@ecordell"
  - "@dmesser"
  - "@kevinrizza"
creation-date: 2021-06-10
last-updated: 2021-07-12
status: implementable
see-also:
  - "/enhancements/declarative-index-config.md"
replaces:
  - https://github.com/operator-framework/operator-registry/pull/243
---

# catalog-data-transfer-reduction

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

1. Is this work targeting older / existing catalogs?
    - Since there is no effort to migrate older catalogs to DC format, no.
1. What happens when two indices behind two separate CatalogSources serve the same bundle? Is this allowed?
    - This is allowed, and the [resolver][resolver] will pick the higher priority catalog’s bundle at runtime.
    Catalog priority is encoded as an integer, and should be treated opaquely here.

## Summary

Disconnected environments suffer from a data transfer bottleneck: a human
must physically transfer image data transfer between connected and disconnected mount points.
This directly affects operator catalogs, which contain hundreds of packages
which contain at least one bundle image reference that also references at least
one operator image. These packages are updated with new versions constantly;
transferring the entire catalog each time may require multiple storage media,
ex. DVDs, being filled at each transfer event.

Optimally, the only data that requires transfer is all "new" data, i.e. images published
since the last transfer. The initial transfer can also be reduced to the packages
required at cluster creation time, and the latest versions of packages not installed
for real-time discoverability.

## Motivation

### Goals

#### Reduce the amount of data to download to initially mirror disconnected catalogs

I should be able to prune down a catalog containing one or more
package channels to only their head bundles.

#### Select a specific set of bundles to include in the mirror-able payload

I should be able to include any bundles (packages at a particlar version)
I need to serve existing workloads.

#### Reduce the amount of data to keep a mirrored catalog updated on regular basis

Given an old catalog and an updated version of that catalog,
I should be able to produce a diff containing only new/changed catalog bundles.

#### Provide catalog manipulation functionality as both a command and library

I want to have the option to import functionality described here as a library
to wrap in some program, or as a CLI tool that calls the library (`opm`).

#### Define the on disk representation of a differential data set (catalog)

I should be able to configure the tool(s) described here in a canonical manner,
and receive an output commonly understood by catalog manipulation tools.

#### Support a simple and controllable approach to updating the disconnected side’s catalog

Once a diff is produced and mirrored, I should be able to add or merge the diff
with an existing catalog seamlessly. This addition/merge operation should be verified
to work on the connected side prior to mirroring the diff.

### Non-Goals

1. Perform the mirroring itself.
1. Resolve dependencies using OLM’s dep [resolver][resolver] on the connected side.
    - This is a candidate for follow-up work.
1. Decide which architectures to mirror.

## Proposal

### User Stories [optional]

<!-- TODO maybe -->

### Implementation Details/Notes/Constraints [optional]

This section delves into [declarative configs][declcfg-ep] (DC) heavily so it is recommended you read that EP first.

Some terminology:
- *Bundle*: a collection of manifests that define an operator at a particular state (ex. version).
- *Replaces*: a [reference][replaces] to another bundle that can be upgraded from to the referent.
- *Skips*: a [reference][skips] to another bundle that can be skipped when upgrading to the referent.
- *Dependency*: a package or group-version-kind (GVK) requirement by a bundle.
  - There are three ways to specify dependencies*: by full package, by package version range, and by GVK.
  The first way is effectively the same as the second, with version range being the entire set of published versions.
- *Upgrade graph*: the DAG of replaces from one bundle to another, skipping bundles referenced in skips.
- *Replaces chain*: upgrade graph, ignoring skips.
- *Channel*: a named upgrade graph.
- *Channel head*: a bundle in a channel that is not referenced by any replaces or skips.
- *Package*: a collection of channels.
- *Index*: a collection of packages, each of which may or may not contain all published bundles for that package.
  - Typically built into an image that can serve packages via gRPC when run to OLM via a CatalogSource
- *Catalog*: one or more indices serving content.

Catalog invariants:
- A package name + bundle name is a unique identifier within a catalog, hence is an ID.
- Channels and packages have unique names within their parent type
(bundle->channel->package->catalog).
- A CSV name is unique to a set of catalogs. That is, a catalog is invalid if
two packages contain CSVs with the same name.
    - This may be removed in favor of package + bundle name only.

An aside on dependencies: an operator can specify package or GVK dependencies
that must be satisfied by some catalog in a cluster. Otherwise the operator
will fail to install at runtime. This is not an enforced catalog invariant
because a dependency may be satisfied by any available catalog,
a fact that cannot be known until runtime.

The scenario this EP attempts to solve involves two remotely available catalogs:
an old catalog that is either empty or contains operator bundles,
and a new catalog that is not empty and contains most if not all of
the old catalog's bundles (if not empty), plus newly published bundles.
Effectively the second catalog is the first at a newer state.
To service the above [goals](#goals), a process that calculates the difference
between all bundles of these two catalog is needed. This "diff" is a partial
catalog, in that some packages may have valid upgrade paths to a channel head
within this catalog, and some may only have valid upgrade graphs
in the presence of the old catalog (explained below).
It is helpful to think about such a "diff" operation in terms
of the smallest catalog unit, the bundle, because a diff would
ultimately operate on a per-bundle granularity to produce a partial catalog.

A new `opm diff` command will be introduced, with an accompanying library, to calculate a diff.
The output will be a declarative config. There are two modes to `opm diff`:
- *headsOnly*: remove everything but channel heads from a new catalog
and dependencies not satisfied by the old catalog. This is effectively a prune operation.
in the diff, since a cluster admin may already have a set of requirements in mind.
- *latest*: keep only the bundles either not in the old catalog or that have
changed somehow. This can be thought of as a traditional diff.

Both modes will accept a list of packages, channels, and/or bundles to include
that may not be in the initial or subsequent diffs.

The `headsOnly` mode removes old catalog data that will likely not be used by cluster tenants;
if specific bundles are required, they can be requested during this or later diffs.
This mode can be used to produce an initial catalog for a disconnected cluster,
for which having all package history is extraneous.
The `latest` mode creates a slim set of bundles to be mirrored that, when
[served alongside or merged](#merging-catalogs) with the existing old catalog,
presents as a superset of the new catalog. The inputs can be any catalog artifacts
understood by [`opm render`][opm-render] (sqlite databases, images, etc.),
and the output is a [DC][declcfg-ep] that can be either built into
an index image or mirrored directly with a command that understands the DC format.

#### Diff algorithm

Since package + bundle name is unique to the catalog,
bundle lookup between two catalogs and comparison is possible.
Two bundles are equal if all data and metadata of each bundle are semantically equivalent.
If any datum semantically differs between the two, the bundles are unequal.
Practically speaking, these data are bundle properties like channels,
objects, and upgrade-related information.

Given this primitive, two catalogs `A` and `A'` can be compared, where `A'` is considered
a newer version of `A`, and unequal bundles or bundles not in `A` can be added to some output
catalog `B`. Then, every bundle `a'` in catalog `A'` either has a counterpart `a` in `A`
that is either equal, or newer and therefore has a more desirable state,
or no counterpart and therefore is of the newest state:
- If `a` exists and `a == a'`, neither `a` nor `a'` are added to `B` because `A` has newest bundle state `a`.
- If `a` exists and `a != a'`, `a'` is added to `B` because `A` does not have the newest bundle state `a'`.
- If `a` does not exist, `a'` is added to `B` because `A` does not have the newest bundle state `a'`.
- If `a'` does not exist, `a` is not added to `B` because `A` has the newest bundle state `a`.

The `headsOnly` mode is a special case of the above algorithm, because
it is obviously undesirable to add all of `A'` to `B` given the set of goals;
therefore, the condition that if `A` is empty, then only add the `a'`'s that
are channel heads to `B`.

Since all `a` and `a'` that were added to `B` either fit into an existing
upgrade graph from `A` or `A'` or have had the upgrade graph updated
to include more constituents by the algorithm, it follows that every bundle's
newest state is now satisfied by `A` + `B`.

##### Dependencies

Given that any bundle may specify some package at a version range (effectively a set of
bundles) or GVK dependency that must be satisfied by some catalog in a cluster,
the algorithm must expand such that dependencies are included. Finding the exact
set of dependencies that satisfy a catalog is a job for [OLM's resolver][resolver]
due to the problem's intractability; however, the latest versions of bundles
that satisfy one or more dependencies can be found and included in the partial
catalog such that entire packages of satisfying bundles do not need to be included.

For example, a bundle selected for the partial catalog `foo.v0.1.0` has
a GVK dependency of kind `Bar`, apiVersion `example.com/v1`
that is satisfied by bundles `bar.v0.1.0` and `bar.v0.2.0`, where
`bar.v0.2.0` replaces `bar.v0.1.0` and neither are in the partial catalog initially.
Naively both bundles could be selected for the partial catalog, but because
only one is needed, the latest bundle `bar.v0.2.0` can be solely chosen.
If another bundle `baz.v0.1.0` is selected for the partial catalog
and depends on kind `Buf`, apiVersion `example.com/v1alpha1` that is satisfied
only by `bar.v0.1.0`, then the algorithm must select both bundles.
This example extends to package-versions by replacing GVK with
some version range, selecting the latest bundle satsifying that range
for each range, then deduplicating by package + bundle ID.

#### Merging catalogs

Now that the diff (catalog `B`) has been created, some mirroring command
like `oc adm catalog mirror` can write the DC and all referenced images to a disk mount.
The mount's data is then transferred by hand to the disconnected side,
and images are pushed to a catalog registry, making the partial catalog image available.

If there is no existing catalog in the cluster (fresh cluster, mode `headsOnly`),
then a CatalogSource and set of Subscriptions are created and the process is complete.

If one or more catalogs do exist in the cluster (existing cluster, mode `latest`),
then there are two options for merging `A` and `B`:
1. A new CatalogSource is be created for `B` to live alongside that of `A`.
OLM's resolver will [prioritize][cs-priority] bundles from `B` over counterparts in `A`.
All replaces will point to bundles in either `A` or `B` by the algorithm.
2. Pull the existing index image for `A` (retrieved from a CatalogSource field)
and merge it with index image to form a new catalog:
    ```sh
    opm render mirror/catalog-A:tag mirror/catalog-B:tag > index.yaml
    docker build -f index.Dockerfile -t mirror/catalog-merged:tag . # Dockerfile created from catalog A's as a tempalte.
    docker push mirror/catalog-merged:tag
    kubectl apply -f merged-catalog-source.yaml # A CatalogSource referencing mirror/catalog-merged:tag
    ```
    1. This option is desirable for long-lived clusters (i.e. most cases) because
    only one catalog is present as the ultimate outcome of the proposed workflow,
    and Subscriptions can be easily written against one CatalogSource whereas having
    N CatalogSources for N diff operations (option 1) is not scalable.

### Risks and Mitigations

- Some undefined behavior may occur when running `opm render` on two refs containing
one or more of the same package + bundle IDs that have different data, dependent on ref order.
This case needs to be tested and fixed/worked around if undefined behavior occurs.

## Design Details

### Test Plan

Integration testing on a robust catalog test set with OLM's resolver is necessary to make sure
`opm diff` produces a valid partial catalog.

## Implementation History

- The `opm <index|registry> prune` command [exists][opm-prune], which can only remove
full packages from an sqlite-backed index store.

## Drawbacks

- Unless the full disconnected catalog state is provided, there may be duplicate
transfer of dependencies.

## Alternatives



[declcfg-ep]:https://github.com/operator-framework/enhancements/blob/master/enhancements/declarative-index-config.md
[resolver]:https://github.com/operator-framework/operator-lifecycle-manager/tree/master/pkg/controller/registry/resolver
[replaces]:https://github.com/operator-framework/community-operators/blob/642d49ff5180e6caee9e874eb191ce8ce3f1bd49/docs/operator-ci-yaml.md#replaces-mode
[skips]:https://github.com/operator-framework/api/blob/v0.10.0/pkg/operators/v1alpha1/clusterserviceversion_types.go#L311-L315
[opm-prune]:https://github.com/operator-framework/operator-registry/pull/243
[render]:https://github.com/operator-framework/operator-registry/pull/661
[cs-priority]:https://github.com/operator-framework/api/blob/da186270e3db3fed9cc1b1d36708114e683c5308/pkg/operators/v1alpha1/catalogsource_types.go#L45-L52
