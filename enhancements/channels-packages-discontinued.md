---
title: Neat-Enhancement-Idea
authors:
  - "@camilamacedo86"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2022-04-26
last-updated: 2022-04-26
status: implementable
---

# usage of properties to describe that channels or packages will be deprecated

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [X] Graduation criteria for dev preview, tech preview, GA

## Open Questions 

1. Should the SQLIndex schema or API be changed to address this requirement?

No. We want this option only from FBC and the changes will not be backported.

2. Could we not use the CSV maturity spec to define that the channels or package are deprecated/discontinued?  

Allow the feature via the CSV spec maturity seems 
not a good option since Operator Authors would need to change previous publications or publish 
new versions only to add this info. On top of that, the data is part of the channels/package 
level and try to inform it via bundles does not fit properly.

3. Currently, we have a property `olm.deprecated` valid only for bundles. This property is used to flag that a bundle is deprecated 
when we use the command `opm deprecatetruncate` and in for some specific scenarios was not possible to remove the whole bundle from the index. Therefore:

a) What's the current OLM/console behavior for the `olm.deprecated` property on bundles?

Currently, it shows that bundles with the property `olm.deprecated` are not installed by OLM and
that the console does nothing about it.

b) How the property `olm.deprecated` is migrated to the FBC schema? 

Then, this property will be under the channel, e.g:

```json
{
    "schema": "olm.bundle",
    "name": "multicluster-operators-subscription.v0.2.6",
    "package": "multicluster-operators-subscription",
    "image": "quay.io/openshift-community-operators/multicluster-operators-subscription@sha256:fc017a62871038b4618313aee013eed6300ea9da100ffca807bc13e51f6f96c2",
    "properties": [
        {
            "type": "olm.deprecated",
            "value": {}
        },
```

c) Do we want to extend the behavior of the `olm.deprecated` property on bundles? Could we extend without making a breaking change? Does the existing behavior translate to what we want for channels/packages?

- For bundles, it means that the bundle is deprecated and should not be installed by the OLM. (no longer available already)
- For channels and packages as proposed on this doc it would mean only provide the metadata info to its consumer. (no longer maintained and some point it will no longer be provided)

So, we cannot say that by using `olm.deprecated` property under channels/package we would keep or extend the behaviour
which came from the old SQL index schema format and from the `opm index deprecatetruncate` command.

d) How `opm deprecatetruncate` works with the new FBC format?

This command is deprecated and is valid only with SQL index format.

In FBC, users are able to just remove the bundle from their package.json when they no longer want to provide it on the index.
Also, they are able to manually change the upgrade graph configurations with replaces, skip and skipRange to fix the graph
accordingly after the removal.

In this way, the current behaviour applied to bundles where the property `olm.deprecated` is used is not valid
for the new FBC format from the moment that Operator authors would be able to opt-in and manage their
own catalog in a declarative way.

e) If we can't make that property behave consistently across the FBC schema types, do we need a different name or a `olm.deprecated.v2` property?

From the moment where users would begin to opt-in for the new format and be able to manage their own
index then, seems like we will need to communicate and clarifies that old implementations based on properties
migrated from SQL index might no longer work in the same way in the future. OLM future APIs has
no intention to keep the same behaviour since they will no longer be required. 

In this way, in the future, `olm.deprecated` under bundles would also work at the same way proposed
here for channels and packages. So that would mean, OLM would not handle it differently or have any 
code implementation tightened to it. Its purpose would be only provide metadata info 
and expose it via the GRPC API for its consumers.

## Summary

This doc aims to propose a solution to allow Operator Authors properly define what are channels
and packages which would be discontinued in future releases. 

This EP proposes the extension
of the GRPC API to expose arbitrary properties under the channels and package level which can 
allow Operator Authors provides metadata info to address other requirements and not
only the specific motivations and use cases point out on this EP.

## Motivation

Following the issues raised which motivates this proposal:
- https://github.com/k8s-operatorhub/community-operators/issues/549 
- https://github.com/k8s-operatorhub/community-operators/issues/750

### Goals

- Allow Operator Authors provide the information to the users about channels and packages which should be discontinued.

### Non-Goals

- Backport or provide equivalent features for SQL format and/or schema.

## Proposal

Extend the GRPC API to expose arbitrary package and channel properties to allow
Operator Authors provides metadata info over channels and packages which can be properly consumed
for others components such as the console.

Therefore, to address the requirement which motivates this EP authors would be 
able to use the property `olm.deprecated` under channels and packages to 
describe these changes so that, components such as the console or OperatorHub.io 
would be able to consume and render this information:

```yaml
properties:
- type: 'olm.deprecated'
  value:
    message: 'human-friendly message'
    moreInfoURL: 'url/link for extra info such as docs or release notes'
    replacement:
      name: 'of package/channel replacing this one (blank means there is no replacement)'
      moreInfoURL: 'url/link for extra info such as repo of the packaged that will be replacing'
```

This proposal is valid only for [FBC](https://olm.operatorframework.io/docs/reference/file-based-catalogs/) index format.

### User Stories

#### Story 1

I am as an Operator Author would like to have an option to clarify to the Operator users that
a channel or the whole packages will no longer be maintained in future releases so that, users
are able to plan no longer consume them and be aware of the required changes.

#### Story 2

I am as consumer of the GRCP API would like to have an endpoint to consume properties under
channels and packages so that, I could implement specific features based on the meta info
provided by Operator Authors over their packages and channels.

### Implementation Details/Notes/Constraints

The proposed changes are to:
- extend the GRPC API to support responses with this metadata
- extend the packagemanifests API to include this metadata

> **Notes:** the exposition of the properties is only supported by FBC.
> So, it seems like we're going to have to touch the [FBC spec](https://github.com/operator-framework/operator-registry/blob/master/alpha/declcfg/declcfg.go) and
the [GRPC API](https://github.com/operator-framework/operator-registry/tree/master/pkg/api) itself.
Although a lot of that code is generated from types that are imported and then scaffolded with the makefile.

Then, once the above items be addressed in the OLM/OPM API the OLM console as any other component 
and consumer of its API would be able to have access to these meta info to properly deal with 
the scenarios which motivates this proposal. 

### Risks and Mitigations

The risk here is the same to expose any new endpoints in an API.
That would mean that if properties would not be allowed in the future for channels and packages
then these changes would need to be deprecated and planned to be removed.

## Design Details

### Test Plan

The exposition of the properties under channels and packages should be unit tested.

### Graduation Criteria

This feature is in GA upon release.

#### Examples

Following some examples over the proposed usage to address this requirement. However, note that
by addressing the proposed changes here the mata info under the channels and packages
could also be used to address further requirements.

- For channels:

```yaml
properties:
- type: 'olm.deprecated'
  value:
    message: 'This channel is deprecated and will no longer be provided in future releases. Please, update your subscription to get updates from the channel stable.'
    moreInfoURL: 'https://github.com/myorg/mmy-operator-repo/discussions'
    replacement:
      name: 'stable'
```

- For packages:

```yaml
properties:
- type: 'olm.deprecated'
  value:
    message: 'This package is deprecated and will no longer be provided in future releases. Please, ensure that you use its Backup Service in favour to move forward and began to use the project SQLite Operator.'
    moreInfoURL: 'https://github.com/myorg/my-operator-repo/deprecation.md'
    replacement:
      name: 'SQLite Operator'
      moreInfoURL: 'https://github.com/myorg/sqlite-operator-repo'
```

##### Removing a deprecated feature

For the future APIs we probably will need to communicate that some
properties such as `olm.deprecated` for bundles will
no longer be handled as before and will work differently. 

### Version Skew Strategy

Afterwords the properties to be exposed via the GRPC API other components
such as OperatorHub.io and the OLM console could begin to properly render
the mata info and take advantage of it so that, would also be required 
to define what are exactly the properties and attributes which should 
be supported and acceptable per each component/consumer as keep them
properly documented.

## Implementation History

N/A

## Drawbacks

If we do not want to allow the usage of properties under channels and packages levels for the FBC schema.

## Alternatives

An alternative section would be tries to use the maturity spec provided via the CSV.
However, as described above it hurts some concepts since the mata info that we are 
trying to provide are related to the package and channels levels as bring some complexities and limitations
since the bundle ought to be immutable.

Also, the long term vision would be bundle be CSVless therefore proposing solutions under the CSV 
does not show go along with the OLM maintainers vision for the future. For further info see [here](https://github.com/operator-framework/enhancements/blob/master/enhancements/csvless-bundles.md).
