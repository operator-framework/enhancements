---
title: Substitutes-For-Graph-Replacements
authors:
  - "@bparees"
reviewers:
  - "@ecordell"
  - "@kevinrizza"
  - "@lgalleti"
approvers:
  - TBD
creation-date: 2020-12-17
last-updated: 2020-12-17
status: provisional
---

# Bundle Substitution

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

1. Can we get the operator pipeline using a single opm binary that is always the latest
version (for both DB building and including in catalogs)?  If not, why not? (it means we 
must backport this feature to all opm versions the operator pipeline uses today)

Answer: no, it's unreasonably constraining to try to force this level of cross-version compatibility on the opm/DB/resolver interactions.  We will backport this feature back to the 4.5 opm binary so that it can be used by the operator pipeline. [See also](https://hackmd.io/8pb60u3KQtiamveuyBYasA?view)


## Summary

It is currently very difficult to inject a new version of a bundle into the middle of an upgrade graph.  This
is because existing nodes in the graph may contain explicit "replaces" or "skips" references to other nodes.
If a new bundle is added to the middle of the graph, replaces/skips references must be updated to include
the new node.

There is a need to rebuild old versions of bundles to pick up CVE fixes.  Doing so requires that the
upgrade graph is preserved such that users who update to the CVE-patched bundle can still continue to
upgrade to newer bundles, just as if they were still on the un-patched bundle.

For example, suppose the original upgrade graph, as specified via "replaces" references, was:

1.0.0->1.0.1

If the patched version is 1.0.1-patched (1.0.1-patched is the 1.0.0 bundle, rebuilt with CVE fixes.  It is not 
the v1.0.1 version of the operator.  Due to semver semantics this is ordered before 1.0.1, which is what we want), 
then ideally we want the new graph to be:

[1.0.0 or 1.0.1-patched]->1.0.1

However achieving such a graph would require updating the 1.0.1 bundle to add a "replaces 1.0.1-patched" reference.
In turn, 1.0.1 can't be updated without changing its version, which means we end up needing something like:

[1.0.0 or 1.0.1-patched or 1.0.1]->1.0.2

(Where the only difference between 1.0.1 and 1.0.2 is the replaces references).


To avoid having to update and publish new versions of subsequent bundles in the upgrade graph, we need a way to
effectively stitch 1.0.1-patched into the graph without changing 1.0.1 (or any other existing bundles that need
to be able to upgrade from 1.0.1-patched).

This enhancement proposes we do so by allowing a bundle to specify which other bundles it "substitutes for".  This
will effectively act as a search/replace in the existing bundles for any references to the bundles this bundle 
substitutes for.  

Continuing the example above, if 1.0.1-patch is annotated as substituting for 1.0.0, then the effective new upgrade 
graph would be:

1.0.0->1.0.1-patched->1.0.1
and
1.0.0->1.0.1


The implications of this for opm add + the catalog db are that the substitutes-for metadata must be stored in the DB associated w/ each bundle.

The expected flow is:

1) opm add inserts the new bundle following all existing logic, but also adding the substitutes-for metadata
2) during insert, bundles that reference the now substituted bundle (bundles that skip, replace, or skipRange the bundle that's being
substituted) are updated to replace/skip/skipRange the replacement bundle (as well as still skipping/skipRanging the substituted
bundle)


Complications:

1) the CSV itself contains other version references(beyond skips/replaces), but it should be ok not to perform updates on them because those references are not used by anything making upgrade decisions (i.e. they are not consumed by the resolver)

2) Substitutions must be applied transitively.  If C substitutes B and B substituted A, it should be assumed that C substitutes A.  The db metadata and substitution processing needs to reflect this.  This also means that all bundles within a package must be reprocessed for substitution updates anytime a new substitution is added to the database.


## Motivation

The primary motivation of this feature is to provide a way for the tooling to automatically rebuild an 
operator bundle in response to CVEs in the operator's dependencies and then add the new bundle to the 
operator catalog in such a way that clusters can upgrade to the new bundle from the unpatched bundle, 
as well as upgrade from the patched bundle to newer bundles.

### Goals

* Patched bundles can be published that result in an intact upgrade graph, without having to modify
other previously published bundles. (e.g. users do not end up stranded because they upgrade to a rebuilt
bundle which cannot be upgraded to the head of another channel).
* Future bundles that reference the unpatched version of a bundle, when a patched version exists in the
catalog, have their references updated to include the patched versions.
* Clusters running a version of the bundle that predates the vulnerable bundle are not forced to
upgrade into the vulnerable bundle before being able to upgrade to the patched bundle.
* The code changes need to be contained within the catalog and opm binary so that it does not require
OCP cluster changes.  It must work across existing 4.5+ versions of OCP clusters.

### Non-Goals

* Fully refactoring the upgrade graph management system out of bundle metadata so that upgrade graphs
can be modified independently of bundle modification/publication.  (This is a future goal for OLM, however)


## Proposal

The proposal is to add a new field (tentatively named "substitutes-for") to CSVs, the value of which is one or
more csv names that this bundle can be plugged in place of.  This will allow existing CSVs that indicate
they "replace" or "skip" some other CSV, to also replace/skip the new CSV that substitutes for the one
they skip/replace.  The net result of this is the ability to publish new CSVs that can still be upgraded
to the latest CSV, without having to update the latest CSV to add new replaces/skips references.

Because the set of substitutions may grow over time, and are not guaranteed to be added in any particular 
order, the substitutions must be applied in an order independent way.  This means each time a new 
substitution rule is added, additional substitution rules must be generated to account for transitive
substitutions.

For example, suppose we have the following set of relationships we need to add to the catalog
(In this example, A has a CVE, but B has already shipped as a replacement for A.  We want to patch
A as it is still the head of some channel.  We produce A' to patch A, and then find another issue in
A' and produce A'' to patch A'):

B replaces A (original CSV relationship)
A' substitutes for A
A'' substitutes for A'

We would first add the rule:
A' -> A

Since this is the first entry, there are no additional rules to generate.

Next we add:
A'' -> A'

Which generates the following transitive substitution(A''->A'->A):
A'' -> A

So the total set of substitution rules is:
A' -> A
A'' -> A'
A'' -> A

Which means that ultimately B will be able to replace A, A', or A'', as desired.



If we work the example in a different order we would first add the rule:
A'' -> A'

Since this is the first entry, there are no additional rules to generate.

Next we add:
A' -> A

Which generates the following transitive substitution (A''->A'->A):
A'' -> A

Giving us the same total set of rules:
A' -> A
A'' -> A'
A'' -> A

Note that each time a new rule is added, the substitutions must be re-processed for additional
new rules, until no new rules are generated.

Given the set of substitution relationships:
A' -> A
A'' -> A'
A'' -> A

As well as the replaces definition in the CSV (B replaces A), it remains to
to produce a catalog DB that will allow OLM to upgrade versions A'', A', and A' of the operator, 
to version B.

To achieve this, the existing bundle metadata in the catalog must be mutated as part of the opm 
add processing.  Since "replaces" can only have a single entry, we will also need to add additional 
"skips" entries to the bundle to account for the other valid upgrade paths.  Specifically, the mutated
bundle for "B" will indicate that it replaces A''(the final substitution for A), and
skips A and A' (the other values in the substitution relationship)

In addition, the mutated version of A'' will add a skip for A.  (A'' will already skip A' as 
provided by the CVE rebuilder).  Finally, A' will also skip A, as defined by the CVE rebuilder.

This means the bundle metadata in the catalog DB will reflect that:

B: replaces A''(originally was A), skips A, A'(result of substitution of A)
A: whatever the original CSV listed
A': skips A(added by the CVE rebuilder), plus original CSV
A'': skips A'(added by the CVE rebuilder)+A(carried from CSV for A' when A' was rebuilt)+original CSV

This in turn will be reported via ListBundles, ensuring that OLM can resolve the upgrade
correctly.

### User Stories [optional]

Detail the things that people will be able to do if this is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.

#### The CVE rebuilder rebuilds a bundle that is the head of the channel, the bundle
does not exist in any other channels.

In this case, the CVE rebuilder simply takes the CSV it is rebuilding and adds a
"skips" reference naming the bundle it is rebuilding.  A substitutes-for
reference must also be added to the CSV since future bundles might reference
the original bundle rather than the rebuild.

#### Bundle B replaces bundle A, bundle A is rebuilt as A' to fix a CVE.

The CVE rebuilder will rebuild A as A', adding a "skips: A" reference.  In addition
it will add a "substitutes-for: A" reference.

When A' is added to the catalog, the result of substitution processing will be that
B will include the following references:
1) replaces A' (substitution for A from the original CSV)
2) skips A (because of replaces A plus the A'->A substitution rule).  Ideally
this would also be a "replaces" reference, but we cannot have multiple replaces
references.

The result of this is that users on A can update to A' or B(depending which
channel they are on).  Users on A' can update to B.

**Note: This is not ideal since it means a user can be upgraded from something older
than A/A', directly to B, without going through A/A', which may mean they
skip a necessary migration.  For example, suppose that A skips some bundle Z.
We will now have a skips chain of B skips A skips Z, which means upgrades
directly from Z to B are possible.  Prior to the rebuild/substitution logic,
the upgrade graph was B replaces A skips Z, which would have ensured A
was installed as part of upgrading to B.  Need to discuss this scenario further.**

#### Bundle B skips bundle A, bundle A is rebuilt as A' to fix a CVE.

The CVE rebuilder will rebuild A as A', adding a "skips: A" reference.  In addition
it will add a "substitutes-for: A" reference.

The catalog DB entry for B will include the following references:
1) skips A (from the original CSV)
2) skips A' (because of skips A plus the A->A' substitution rule).

The result of this is that users on A can update to A' or B(depending which
channel they are on).  Users on A' can update to B.


#### Bundle A is rebuilt as A', bundle A' is rebuilt as A''

Substitution rules are transitive, so opm add must mutate any references
(replaces/skips) to A such that they also include skips of A' and A''.

Again ideally an existing `replaces A` would be updated to `replaces: A,A',A''`
but since multiple replaces are not valid, we must settle for `replaces: A''`
plus `skips: A,A''`.


### Implementation Details/Notes/Constraints [optional]

A new column will be added to the catalog DB's operatorbundle table.  The
column will hold the "substitues-for" value, indicating what bundle
this bundle is a rebuild/substitution for.  The column will hold a single
value (it will not be valid for a bundle to substitute for multiple bundles).

The column value will be populated from a new substitutes-for field on the CSV
(There is some debate about whether this should be part of a generic properties
map on the CSV.  For now I am inclined towards a first class field).

Any time a bundle is added to a package, the following steps must occur:

1) substitutes-for metadata from the bundle is added to the DB
2) substitution chains are established (e.g. if D substitutes C and C
substitutes B, the chain is D->C->B) for transitivity purposes.
2) For all bundles in the package, any replaces reference is updated to
the head of the substitutes-for chain (e.g. given a substitution chain 
of D->C->B->A and a replaces reference to B, the replaces reference
will be updated to reference D).
3) The prior replaces reference becomes an additional skips reference
4) Additional skips references are added such that the bundle will now
skip anything that substitutes for anything it previously skipped. 
(E.g. given a subsitutes chain of D->C->B->A and a skips reference of B,
the bundle will be updated to also skip C and D).
5) Given a bundle with a skipRange, for all bundles that fall within the 
skipRange of another bundle, skips must be added to the bundle with the
skipRange, such that the bundle now skips anything that substitutes for
things it previously skipped.  For example, given a bundle Z which 
has a skipRange that encompasses a bundle A, and given a bundle A' which
substitutes for bundle A, bundle Z will gain an additional "skips A'"
because A' substitutes for A, and A falls within Z's skipRange.


One of the more subtle pieces of the substitution logic is ensuring that
the existing "replaces" reference in a CSV is substituted with the "latest"
substitution.  That is, if a CSV "replaces A" and A has multiple substitutions
available (A', A'', etc), we want to mutate the reference to state "replaces A''"
(where A'' is the latest substitution).  We can enforce this ordering
by requiring that only one substitution relationship can be defined for a given
CSV.  E.g. if "A substitutes-for B", then nothing else can also declare that
it substitutes-for B.  This ensures we can always identify the "final"
substitution to use in the replaces field.

Once we have that assertion, it's trivial to ensure all substitutions are 
done using the "final" substitution.  E.g. if:
A -> A'
A' -> A''
A'' -> A'''

Then any "replace" reference to A, A', or A'' will be mutated to A'''.

This will ensure that operators are upgraded through the latest version of a 
given "substitution equivalent" operator, hopefully thereby using the latest
migration logic.


### Risks and Mitigations

The primary risk is that in mutating the CSVs we break the upgrade path in some
way, either by not adding skips/replaces that we should, adding incorrect ones,
or changing the upgrade behavior (such as the aforementioned cases where
we go from a "replaces" upgrade path to a "skips" upgrade path, and some operator
migration logic does not get executed).

Not aware of any security concerns.


## Design Details

### Test Plan

**Note:** *Section not required until targeted at a release.*

Need to test the user scenarios described above.  Generally:

1) rebuilding a bundle multiple times (creating transitive substitution
relationships)
2) rebuilding bundles that are referenced by other bundles
3) adding bundles to the catalog in different orders to ensure
replaces/skips references are always updated correctly.

And ensuring in all cases that the CSV content served by ListBundles
contains all the necessary/correct skips/replaces references to ensure
a correct upgrade graph for all channels.  (ListBundles itself is not
changing, but it renders the content which was mutated in the database)

### Graduation Criteria

This will be put directly into production once implemented, as it will be
consumed by the CVE rebuilder.  So substitutes-for will need to be a GA api with
a GA implementation in OPM.

### Upgrade / Downgrade Strategy

All logic related to this behavior will be tied to the opm binary used to
generate the catalog, and the opm binary in the catalog image that is used
to serve the content.  Therefore it should not require any changes to 
OLM on the target cluster.  Catalogs produced that use this feature must
work with any existing supported OCP cluster.

Users who want to take advantage of this feature will need to use a current
opm binary both to build their catalog DB as well as packaged in their 
catalog image.

Note:  Today the operator pipeline uses the latest opm binary to build catalog DBs
(good) but matches the opm binary in the catalog to the ocp version.  We will
need to backport the substitutes-for support in opm to all opm versions needed for
the operator pipeline (today that means back to 4.5 at least).

If an existing catalog DB is being modified by this new opm binary, it will need
to add the new table/column tracking the substitutes-for metadata.


### Version Skew Strategy

See upgrade/downgrade strategy for discussion of these issues.


## Drawbacks

This is bolting on yet another piece of CSV metadata that influences the upgrade graph.  The upgrade
graph (when using skips/replaces/skiprange) is already often misunderstood by operator authors
and this adds another piece to reason about.  

Furthermore, the actual replaces/skips metadata can't be seen by just looking at the bundle itself since 
the references are mutated only in the DB based on what substitutes-for relationships exist, making it even 
harder to validate the upgrade graph reflects what is desired.  Unfortunately that is a necessary 
consequence of the fact that we are trying to achieve a way to modify what bundles an already shipped 
bundle can upgrade from, without re-shipping the bundle.  So post-publication mutation is a requirement.



## Alternatives

### Mutate in ListBundles

Rather than modifying the metadata in the catalog DB as bundles are added, we could mutate
the content as it is served by ListBundles.  This has the advantage of only processing the
substitution relationships when they are definitely needed and the complete set is known,
however it also means processing that logic every single time the api is invoked, which is
ultimately more costly than doing it once on each bundle add.

### Declarative Config

We can abandon skips/replaces entirely and moving to an explicitly defined upgrade graph.  
The CVE rebuilder could then stitch new content into that upgrade graph as it is produced.
The primary arguments against this approach, for now, are:

1) we need something sooner than that will be ready
2) even if we had it, using it would require operator teams to migrate to the declarative config
meaning yet another operator migration(though not as big as the migration to bundles)
3) the operator pipeline would also need to be modified to consume the declarative config (and where
it would be stored)
4) we'd still have to sort out how the declarative config can be modified by both operator authors
and the CVE rebuilder without conflicting with each other or reverting each other's changes.

Note that ultimately we will have to solve these problems if we deliver declarative config (as
we currently intend to do) but it doesn't seem like we have a short runway to providing such
a solution.
