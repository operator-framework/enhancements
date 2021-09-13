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
* Creates a smaller size bundle to ship
* Prevents including testing resources in shipped bundle
  * Stop potential elevated RBAC's in the test bundle from being included in production environments

### Goals

The goal is to create a solution which allows developers to continue to access and utilize the testdata while still having the option to ship an operator without said testdata in the bundle.

### Non-Goals

The goal of this EP is not to change the contents of the bundle nor the testdata, but just to reorganize them, either with multiple bundles or by allowing the tools to modify a bundle to achieve the desired state.

## Proposal

Please note that this proposed solution was elevated from the Alternatives selection after discussions, and the previous implementation has been moved down.

Hard Split
* Split existing bundle in a testdata only bundle paired with a operator essentials only bundle
  * Allows the bundle to save significant space/reduce its size
  * Allows operator developers to securely ship with including potentially sensitive testdata
    * Users can use elevated permissions in testing without concerns about shipping those configs
* Primary impact lies within scorecard internals, as well as scaffolding
  * Scorecard needs logic to understand whether or not there are two bundle or one
    * Recombine bundles at scorecard runtime
      * Scorecard runs continue to behave the same, all this forces is a pre-processing step
    * Allows scorecard test developers to contiue to use `/bundle` context
* Allow discovery of test bundle using Annotation path
  * Scorecard currently assumes testdata in the in current bundle, but can read the annotation to find another bundle
    * This bundle can be configured as the seperate test bundle
  * Seamless tranisition for users, exisiting CLI commands will continue to work in scipts if annotations are correctly configured
  * This method does require manual user updates for configuration
* Scorecard CLI extension
  * Add ability to process one OR MORE bundles in the command line args
  * If multiple args exist combine bundles and carry on
* Opt In design
  * By default bundles will continue to be scaffolded as is
  * Users can always re-scaffold the bundle to toggle between types
* Minimizes User Impact
  * Current operators will still work
  * New single bundle opertaors will not require any changes in the development or test process
    * Those who opt in will have minimal manual adjustments needed for the 2 bundles to work together
  * Scorecard test developers can expect the same scorecard API's and bundle mounting
 

## Design Details

### Test Plan

For testing, ideally all new paths created in the CLI should be verfied in integration testing. Creating a full bundle, slim bundle, and then adding and/or removing testdata scaffolds from both. 

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

Implementation and testing of this feature should be sufficient for graduation. The intended implementation will maintain current user behavior as default.

## Alternatives

Flexible Bundle (Previous Implementation Choice)
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

Dual bundle
* Maintain the current bundle for daily use and provide the option to scaffold a slim bundle at any time
* This would make manual changes to the bundle very difficult to replicate unless the slim bundle generation was based on copying/deleting

