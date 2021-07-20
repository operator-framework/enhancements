---
title: testdata-separation
authors:
  - "@theishshah"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2021-07-12
last-updated: 2021-07-20
status: TBD
---

# testdata-separation


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

The current bundle includes both an operator as well as its testdata. This results in inflated binaries and scenarios where users may want to release an operator bundle sans testdata but are unable to do so. The solution to this issue is to provide users the ability to use and generate a slim bundle which is decoupled from the testing necessities. Due to the nature of the testdata, it needs the rest of the bundle so it has limited value alone.

## Motivation

The motivation for this EP is to provide users with the option to deliver a slim bundle without testdata.

### Goals

The goal is to create a solution which allows developers to continue to access and utilize the testdata while still having the option to ship an operator without said testdata in the bundle.

### Non-Goals

The goal of this EP is not to change the contents of the bundle nor the testdata, but just to reorganize them, either with multiple bundles or by allowing the tools to modify a bundle to achieve the desired state.

## Proposal

Flexible Bundle
* Allow the bundle to be scaffolded with or without test data initially
  * By default, the current structure of bundle + testdata will be scaffolded
  * Add ability to scaffold test data at any time
  * The SDK should allow changing between bundle types at any time, although changes made manually aren’t guaranteed to be preserved
* Allow users the option to strip test data from the bundle post development
  * This would let users maintain their current development flow and axe testdata from the bundle when the time is right to ship
  * Ideally this is done through the osdk CLI, however it is reasonable to have this in the makefiles of operators
* Creates the path of least resistance for OSDK development time
  * This also maintains the current default behavior for existing sdk developers
  * Only changes to CLI and scaffolding need to be made
    * Need to add flags to scaffold slim (no testdata), and to strip or add testdata from an existing scaffold
  * Scorecard, testing, and general operator bundle UI and behavior remains the same for seasoned users as well as contributors
    * The bundle will continue to generate as is under the current workflow

## Design Details

### Test Plan

For testing, ideally all new paths created in the CLI should be verfied in integration testing. Creating a full bundle, slim bundle, and then adding and/or removing testdata scaffolds from both. 

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

Implementation and testing of this feature should be sufficient for graduation. The intended implementation will maintain current user behavior as default.

## Alternatives

Hard Split
* Split existing bundle in a testdata only bundle paired with a operator essentials only bundle
* Would require significant rework in testing to accommodate a test data bundle being strictly separate from the rest of the bundle
  * This will impact how scorecard finds it’s config and test images
    * Primarily scorecard internals, external apis should remain the same
    * Scorecard users would need to learn to work in the new context with their test image and config in a different bundle location
* Still saddles users with the testdata bundle upon scaffolding, but can trivially be removed

Dual bundle
* Maintain the current bundle for daily use and provide the option to scaffold a slim bundle at any time
* This would make manual changes to the bundle very difficult to replicate unless the slim bundle generation was based on copying/deleting

