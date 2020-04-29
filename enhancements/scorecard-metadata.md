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
last-updated: 2020-04-29
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
and other test assets to be included into the bundle.  This allows
test frameworks and tools a known place to search for test assets
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
│   └── memcached-operator.v0.0.1.clusterserviceversion.yaml
├── metadata
│   └── annotations.yaml
└── tests
    └── scorecard
        ├── config.yaml
        └── kuttl
            ├── example1
            └── example2
```

In the above proposed layout, the test metadata resides under a new
directory called `tests`.  Below that directory is free form but
in the case of operator-sdk scorecard, it would populate this
directory with a subdirectory named `scorecard` as depicted.

## Proposal

### User Stories 

#### Story 1

As a test developer I would like to be able to include and replace test metadata into 
operator bundles.

#### Story 2

As a test developer I would like to extract test metadata from
operator bundles so as to execute tests against the operator and its bundle contents.

### Risks and Mitigations

You would not want to define locations in the bundle that might
conflict with other metadata or usage of the bundle.

Some sort of check would need to be created when inserting test
metadata within a bundle to allow a maximum size of metadata
that needs to be determined.

