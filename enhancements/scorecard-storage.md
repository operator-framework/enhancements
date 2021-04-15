---
title: scorecard-persistent-storage-option
authors:
  - "@jemccorm"
reviewers:
  - TBD
  - "@estroz"
  - "@jesusr"
  - "@bparees"
approvers:
  - TBD
creation-date: 2021-03-26
last-updated: 2021-04-15
status: implementable
see-also:
  - "/enhancements/this-other-neat-thing.md"  
replaces:
  - "/enhancements/that-less-than-great-idea.md"
superseded-by:
  - "/enhancements/our-past-effort.md"
---

# scorecard-storage-option

This enhancement provides scorecard users a means to persist reports or other output from their scorecard tests to a persistent location that can be collected
by scorecard.

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

None

## Summary

QE and other test writers sometimes require storage within their test environment (e.g. test container) to write reports and other test output material.  Today within scorecard this is not possible since a persistent volume is not available within the test container.  As the test completes, the container ends, leaving only the container log on the k8s cluster until the Pod is deleted.

This enhancement would offer an optional mechanism whereby test writers have an emptyDir volume mounted within their test containers enabling them to write test output. Scorecard would set this emptyDir up and extract test output from
it before the test is deleted.


## Motivation

Allowing scorecard tests a place to persist test output is key to running advanced tests and supporting complex test output.

QE teams for example might already have tests which produce test output files, in order for them to integrate their tests into scorecard, they need a persistent volume to write test results into.

This feature enables a wider adoption of tests that can be executed within scorecard.

### Goals

Goals of this proposal include:
* provide a writable volume for scorecard test developers
* Collect test output when scorecard completes

### Non-Goals

## Proposal

### User Stories

#### Story 1
A scorecard user can specify for a given test that they want storage to be made available to that test.  This is specified within the scorecard configuration file as a storage spec.  For example, a user would specify they want storage for a test by specifying
in the scorecard configuration file the `storage` spec as in the following example:

```yaml
  tests:
  - image: quay.io/custom-scorecard-example/example-test:latest
    entrypoint:
    - example-scorecard-test
    - example-test
    labels:
      suite: example
      test: example-test
    storage: 
      spec:
        mountPath:
          path: /test-output
```

Note, that if the mountPath is not specified, then a default mount path
of /test-output would be used.

#### Story 2
Scorecard gathers any persisted volume data upon test completion and stores it in a local directory where scorecard is executed.  Users
can specify a non-default local path for test output using a command flag as follows:
```bash
operator-sdk scorecard ./bundle --selector=suite=custom --test-output=/mytestoutput
```

#### Story 3
A scorecard user can specify storage globally for any selected tests.
The `storage` spec can be defined globally as in the following example:

```yaml
  storage: 
    spec:
      mountPath:
        path: /test-output
  tests:
  - image: quay.io/custom-scorecard-example/example-test:latest
    entrypoint:
    - example-scorecard-test
    - example-test
    labels:
      suite: example
      test: example-test
```

This would mean that any selected test would assume the global storage
settings.  If a test has its own unique test storage defined, then those
settings would override the global storage settings.

### Implementation Details/Notes/Constraints

For each test that specifies storage, scorecard will interate
this sequence for each test:
 * scorecard adds a sidecar container into the test pod, the sidecar container would also mount the same emptyDir volume that is mounted into the test container
 * upon test container completion and before removing the test pod, scorecard
will extract (exec/tar) the test output contents from the sidecar container to the local host that scorecard is executing upon
 * scorecard cleanup would cleanup the test Pod as normal, thereby also removing the added storage sidecar container

A successful scorecard execution of tests that use this storage feature
would include a local directory of the collected test output, with test output being divided into subdirectories based on suite and test names

### Risks and Mitigations

Scorecard users that don't require persistent storage are not impacted by this proposed feature.

## Design Details

### emptyDir Volumes
Each scorecard test that specifies the storage feature will have a unique emptyDir Volume into which they can write test output. This ensures that each
test can be write output in isolation from other tests.  These volumes
will be removed as part of scorecard's test Pod cleanup.  The presence of
a storage spec within the scorecard config file triggers the addtion of the storage sidecar.

By using emptyDir volumes for this feature, the cluster doesn't have to have a storage class defined and volume cleanup is included as part of test Pod cleanup.

### Errors

### Test Output
Scorecard test output is gathered from the emptyDir volumes using a sidecar container that continues to run after the test container completes. The extraction of test output is performed by exec/tar into the sidecar container.
Scorecard test output is extracted from the 'sidecar' container using client-go APIs (e.g. exec).  

Test output is organized by test name in sub-directories.  For example, gathered output by default is stored in the local directory where
scorecard was executed, however the user can also specify a command line flag (e.g. --test-output) to have the
test output stored at a custom location.  The gathered output appears as follows:
```bash
./test-output
./test-output/custom-suite/example-test1/somefile
./test-output/custom-suite/example-test2/anotherfile
./test-output/custom-suite/example-test3/foo.log
```
This directory will be over-written if scorecard is executed multiple times, it is therefore up to the end user
to perform data management of this test output directory as they require.

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:
- Maturity levels - `Dev Preview`, `Tech Preview`, `GA`
- Deprecation

Clearly define what graduation means.

#### Examples

These are generalized examples to consider, in addition to the aforementioned
[maturity levels][maturity-levels].

##### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers

##### Tech Preview -> GA 

- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**



### Upgrade / Downgrade Strategy

This feature doesn't impact existing scorecard tests or test configurations.

I currently don't see value in backporting this feature to previous versions of the operator-sdk.

### Version Skew Strategy


## Implementation History



## Drawbacks

Using emptyDir volumes means the on-cluster volumes are not accessible when the test pod is removed.  However, there is no current requirement to have 
volumes live beyond the life of the test Pod.

## Alternatives

### Alternative 1
An alternative to this feature would be to design a means of embedding test content into the test Pod log itself, co-mingled with the current scorecard test result JSON.

This seems really complicated and hard to maintain so I rejected this in favor of this proposal.

### Alternative 2

Another alternative to using emptyDir volumes would be to provision
persistent volumes per test.  The downside of this approach is
that you have to have persistent storage available on the cluster
and you might have to manually manage error cases that would leave storage
volumes on-cluster.

