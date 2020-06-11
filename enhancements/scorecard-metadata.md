---
title: scorecard-metadata
authors:
  - "@jemccorm"
  - "@jlanford"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-04-29
last-updated: 2020-05-19
status: provisional
---

# scorecard-metadata

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

This enhancement is to define a place within the bundle format whereby
scorecard metadata could be added.  This will allow tests to be co-located
with the normal operator content found in a bundle.  This enhancement
suggests naming and paths for the scorecard metadata within the bundle.

## Motivation

The purpose of this enhancement is to define a location for scorecard
and other test assets to be included into the bundle.  This definition
informs test frameworks and tools of a well-known location to search for test assets
within a bundle and reduces confusion to test developers as to 
where test assets will reside.

### Goals

This enhancement will be used by the scorecard feature within the operator-sdk
initially as part of a major redesign of scorecard functionality.  Other
test frameworks apart from operator-sdk or scorecard could leverage this
enhancement as well.

### Non-Goals

This enhancement is scoped to define the location within the bundle for
this sort of metadata and get consensus on names and paths.  Follow on
work would be required to implement tooling that lets a user inject
their test assets into a bundle image as well as extract this metadata.

## Design Details

The bundle format would allow for test metadata to be
included in the following locations within a bundle:

```
bundle/
├── manifests
│   ├── cache.example.com_memcacheds_crd.yaml
│   └── memcached-operator.clusterserviceversion.yaml
├── metadata
│   └── annotations.yaml
└── tests
    └── other-tests
    └── scorecard
        └── config.yaml

```

In the above proposed layout, the test metadata resides under a new
directory called `tests`.  Below that directory is free form but
in the case of operator-sdk scorecard, it would populate this
directory with a subdirectory named `scorecard` as depicted.

The scorecard test label annotations for the bundle would be as follows:
```
annotations:
  operators.operatorframework.io.scorecard.mediatype.v1: "scorecard+v1"
  operators.operatorframework.io.scorecard.config.v1: "/tests/scorecard/config.yaml"
  operators.operatorframework.io.other-tests.mediatype.v1: "other-tests+v1"
  operators.operatorframework.io.other-tests.config.v1: "/tests/other-tests/setup.yaml"
```

For the purpose of this enhancement, scorecard would assume that its test
metadata would be in the same bundle image as the operator image.

This proposal would require the operator-registry github repo to 
contain a golang API that can be used by the SDK to add scorecard
metadata into a bundle image.

The SDK scorecard is just one example of the type of metadata
that might be added into the bundle image using the proposed 
operator-registry golang API.

The proposed operator-registry golang API would also need to 
support updating or removing custom metadata from a bundle image.

## Proposal

### User Stories 

#### Story 1

As a developer, I would like to use the operator-registry github repo
golang API to include, replace, and remove test metadata within operator 
bundles.  The ability to remove test metadata would support 
disconnected environments where running tests is not applicable.

#### Story 2

As a developer I would like to use the operator-registry github repo
golang API to extract test metadata from operator bundles so as to 
execute tests against the operator and its bundle contents.

#### Story 3

As a developer I would like to use the operator-registry github repo
golang API to extract test metadata from operator bundles but also
have the ability to control what bundle contents are extracted.

### Risks and Mitigations

You would not want to define locations in the bundle that might
conflict with other metadata or usage of the bundle.

Some sort of check would need to be created when inserting test
metadata within a bundle to allow a maximum size of metadata
that needs to be determined.
