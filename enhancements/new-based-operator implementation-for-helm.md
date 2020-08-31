---
title: Neat-Enhancement-Idea
authors:
  - "@camilamacedo86"
reviewers:
  - TBD
  - "@alicedoe"
approvers:
  - TBD
  - "@oscardoe"
creation-date: 2020-08-01
last-updated: 2020-08-01
status: implementable
---

# New based-operator implementation image for Helm

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA

## Open Questions 

1. If we do not prioritize it then, could not all the effort, and these improvements get outdated?  

Answer: Yes, but according to @Joe the possible issues might to be solved in the new implementation and would be possible to try to keep the [joelanford/helm-operator](https://github.com/joelanford/helm-operator) been update with te changes made against the SDK repository.  

## Summary

This proposal is for the new based-operator implementation for Helm to be incorporated to the project by pushing the changes made in [joelanford/helm-operator](https://github.com/joelanford/helm-operator) to the SDK repository. It is proposing replace the implementation in [operator-sdk/internal/helm](https://github.com/operator-framework/operator-sdk/tree/master/internal/helm) for [joelanford/helm-operator/pkg](https://github.com/joelanford/helm-operator/tree/master/pkg) via a PR. 

## Motivation

- Improve the maintainability and reusability
- Adopted properly the [api convention](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties) to work with conditional.status
- Works with hooks and annotations which allow address in a better way feature requested by users and brings more flexibility
- Allow controller customizations with pre and post hooks and annotations to enable/disable hook actions
- Provide cleaned logs 
- Be fully backwards compatible with the existing helm operator implementation
- Re-organize the underlying libraries to actually make them expose a sane public API and make unit testing easier.
- Increase the coverage of tests with unit-tests
- Make it extensible so that others can add their own Go code customizations for hybrid use cases. 
- Cleanup a bunch of issues that make the current implementation difficult to add new features and fix bugs while ensuring that things don't break.

### Goals

- SDK CLI tool providing Helm based-operator images using the pkg implementation done in [joelanford/helm-operator](https://github.com/joelanford/helm-operator)

### Non-Goals

- Split Helm operator implementation in its own repository.

## Proposal

### User Stories

### Apply the changes done in SDK to the new implementation

- I am a maintainer/contributor, I want to be able to release a new version without regressions, so that I will not need spend effort after the release addressing bugs or patch release
- I am a Helm operator developer, I want to check all bug fixes and improvement done in the current implementation in the new one, so that I can use it without face problems
- I am a Helm operator developer, I want to still able to check the diff of the resources in the logs at the same way that it is done with the current implementation, so that I can easily know what was applied when my operator releases and upgrades my helm-charts. 

### Applying the new implementation to SDK

- I am a maintainer/contributor, I want to track the changes performed, so that I have the required information to fix bug or regressions 
- I am a maintainer/contributor, I want to be able to have centralized all code implementation, so that I can easily know what needs to be moved to Helm repository when the projects be splitted. (see the proposal []())
- I am a maintainer/contributor, I want to have a more understandable implementation, so that I can spend less effort to contribute with the project 
- I am a maintainer/contributor, I want to ability to address features based on annotations and hooks, so that I can easily address requests made by the community 
- I am a Helm operator developer, I want to use the new version of helm based-operator image with this new implementation, so that I can get its improvements and bug fixes
- I am a Helm operator developer, I want to see my proposed feature requests addressed, so that I can address better my project requirements  
- I am a Helm operator developer,  I want to check only the relevant information in the logs, so that I can easily identify problems

### Risks and Mitigations

- The implementation in [joelanford/helm-operator](https://github.com/joelanford/helm-operator) gets outdated and be harder ensure that it is covering all bug fixes and improvements made in the [operator-sdk](https://github.com/operator-framework/operator-sdk).
- The same risk of any change to introduce new bugs or regressions
- Find out that the new implementation requires more adjustments than expected when we try to replace the old one. 

## Design Details

### Graduation Criteria

Following the description over what should be done to address the above User Stories.

#### Apply the changes done in SDK to the new implementation

We compared the new with the current implementation in order to ensure that no breaking changes and regressions would be faced when the current one will be replaced. Then, the following actions needs to be done: 

**New implementation is missing the diff in the logs**

In [operator-sdk/internal/helm/internal/diff](https://github.com/operator-framework/operator-sdk/tree/master/internal/helm/internal/diff) is implemented an util which is responsible for log the difference in the resources after the reconciliations. 

The [PR joelanford/helm-operator/pull/43](https://github.com/joelanford/helm-operator/pull/43) solves the issue [compare the logs from SDK and this project #14](https://github.com/joelanford/helm-operator/issues/14) which will ensure that the diff in the logs will be output in the new implementation as well. 

**Bug fix done in SDK**

Ensure that the fix [Helm operator sometimes doesn't update the CR with the InstallSuccessful status condition #3728](https://github.com/operator-framework/operator-sdk/issues/3728) also will be applied in the new implementation.

**SDK Metrics**

Ensure that [new reconciler.go](https://github.com/joelanford/helm-operator/blob/master/pkg/reconciler/reconciler.go) in the [joelanford/helm-operator](https://github.com/joelanford/helm-operator) is using the metrics implementation currently in place. (Note it is check by the [e2e-helm](https://github.com/operator-framework/operator-sdk/tree/master/test/e2e-helm) [here](https://github.com/operator-framework/operator-sdk/blob/master/test/e2e-helm/e2e_helm_cluster_test.go#L265) and then, the CI will let us know if it is not working as should be.) 

#### Applying the new implementation to SDK

After the above tasks we can create a pull request to replace the current implementation by the new. The directory [operator-sdk/internal/helm/](https://github.com/operator-framework/operator-sdk/tree/master/internal/helm) will be replaced by [joelanford/helm-operator/pkg/](https://github.com/joelanford/helm-operator/tree/master/pkg)
 
 However, note that the `operator-sdk/internal/helm/flags/flag.go` has precedence nd should be kept. Theoretically, all should still working and passing in the test, however, we might face the need to perform some adjustment. 

### Test Plan

The implementation in [joelanford/helm-operator](https://github.com/joelanford/helm-operator) is very covered with unit tests which ensure its functionalities. Also, [operator-sdk](https://github.com/operator-framework/operator-sdk) has an e2e test which has ensuring the main functionalities for the projects. The tests in place allow ensures the new implementation. 

#### Examples

Note that in the [joelanford/helm-operator](https://github.com/joelanford/helm-operator)
has an `example/` directory with a sample. Also, it is possible just to build a new image using its Makefile targets and check the new implementation working on with the [Helm Memcached Sample](https://github.com/operator-framework/operator-sdk-samples/tree/master/helm/memcached-operator)
 
### Upgrade / Downgrade Strategy

By centralizing the replaced of the old by the new implementation in a PR it is possible easily revert it and undo all changes if required.   

### Version Skew Strategy

Would be possible release a new beta version of the SDK after this change and ask for the community check before we do a stable release with it. 

## Alternatives

Address this requirement with the need to split the Helm to its repo simultaneously by using the [joelanford/helm-operator](https://github.com/joelanford/helm-operator) repository as a base instead of it. However, with this approach would be harder ensure that no breaking changes, issues or regressions would be introduced. 



