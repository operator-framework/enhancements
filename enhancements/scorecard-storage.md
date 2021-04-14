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
last-updated: 2021-04-12
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

This enhancement would offer an optional mechanism whereby test writers have a persistent volume mounted within their test containers, provisioned, enabled, and cleaned up by scorecard.


## Motivation

Allowing scorecard tests a place to persist test output is key to running advanced tests and supporting complex test output.

QE teams for example might already have tests which produce test output files, in order for them to integrate their tests into scorecard, they need a persistent volume to write test results into.

This feature enables a wider adoption of tests that can be executed within scorecard.

### Goals

Goals of this proposal include:
* provide a persistent volume for scorecard test developers
* Collect persistent volume output when scorecard completes

### Non-Goals

The names of the persistent volumes created by scorecard for this feature
are opaque to the test writer, meaning that they are not defined in a way
where test writers would need or want to know the names of their volumes.

## Proposal

### User Stories

#### Story 1
A scorecard user can specify for a given test that they want persistent storage to be made available to that test.  This is specified within the scorecard configuration file as a PersistentVolume spec.  For example, a user would specify they want storage for a test by specifying
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
        storageClassName: hostpath
        capacity:
          storage: 1Gi
        mountPath:
          path: /test-output
```

#### Story 2
A scorecard user can optionally specify storage class details within the scorecard configuration file to override storage defaults. Volume size, access mode, storage class, and mount paths can be specified as follows:
```yaml
  tests:
  - image: quay.io/custom-scorecard-example/example-test:latest
    entrypoint:
    - example-scorecard-test
    - example-test
    labels:
      suite: custom
      test: example-test
    storage: 
      spec:
        storageClassName: hostpath
        capacity:
          storage: 1Gi
        mountPath:
          path: /test-output
```

Valid values for volume size must be formatted according to the rules defined for storage 
in https://pkg.go.dev/k8s.io/apimachinery/pkg/api/resource#Format.  Typical values for 
storage are like:
 * 1Gi (1 gigabyte) (default)
 * 100M (100 megabyte)

Storage classes can be found on a cluster by the following command:
 * kubectl get sc

A default storage class is the storage class on a cluster that has the following annotation:
```yaml
  metadata:
    annotations:
      storageclass.kubernetes.io/is-default-class: "true"
```


#### Story 3
Scorecard gathers any persisted volume data upon test completion and stores it in a local directory where scorecard is executed.  Users
can specify a non-default local path for test output using a command flag as follows:
```bash
operator-sdk scorecard ./bundle --selector=suite=custom --test-output=/mytestoutput
```

#### Story 4
A scorecard user can specify storage globally for any selected tests.
The `storage` spec can be defined globally as in the following example:

```yaml
  storage: 
    spec:
      storageClassName: hostpath
      capacity:
        storage: 1Gi
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

This feature, if specified by a scorecard user, requires a storage class be defined and operational on the k8s cluster nodes that scorecard tests would be executed upon.  If no storage class is specified or a default storage class is not available an error is returned for the scorecard test.

For each test that specifies storage, scorecard will interate
this sequence for each test:
 * scorecard allocates a unique PVC for a test if that test requests storage
 * scorecard mounts that PVC into the test Pod so the test can write to
a given mountPath any test output they want
 * upon test completion and before cleaning up the tests, scorecard
will create a container that mounts the PVC solely to harvest (exec/tar)
the test output contents to the local host that scorecard is executing upon
 * scorecard cleanup would include PVC cleanup as well as the current Pod cleanup

A successful scorecard execution of tests that use this storage feature
would include a local directory of the harvested test output, with test output being divided into subdirectories based on suite and test names

### Risks and Mitigations

Scorecard users that don't require persistent storage are not impacted by this proposed feature.

If Persistent Storage is used then you have the risk of leaving PVCs on your cluster if scorecard were to exit abnormally.  This is similar to having scorecard not cleanup Pods when it exits abnormally.

## Design Details

### Unique Volume Claims
Each scorecard test that specifies the storage feature will have a unique PersistentVolumeClaim created for that test.  This ensures that each
test can be scheduled on any cluster node and that multiple tests will not co-mingle test output onto the same volume.  Scorecard deletes
these volumes after test output has been gathered from the volume.  If the user specifies the command line flag of "--skip-cleanup", the volume will not
be deleted.

### Default Storage Values
The following defaults will be supplied for the storage feature:
 * 1Gi volume size
 * RWO (ReadWriteOnce) access mode
 * default storage class will be used
 * /test-output mount path

These defaults will likely cover the majority of use cases.  Users can override these values however
as documented above.

### Errors
This storage feature, when specifed in the configuration, requires a valid storage class be present on the cluster.  If a storage
class is not found on the cluster, an error message will be raised in the scorecard test.

When a user specifies invalid storage settings (e.g. storage class, access mode, volume size), an error is raised in the scorecard
test result.

### Test Output
Scorecard test output is gathered from the provisioned volumes using a container that includes the tar utility.  Scorecard test output
is extracted from the 'gather' container using client-go APIs (e.g. exec).  

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

This feature and suggested implementation depend on dynamic storage
being available on-cluster.


## Alternatives

### Alternative 1
An alternative to this feature would be to design a means of embedding test content into the test Pod log itself, co-mingled with the current scorecard test result JSON.

This seems really complicated and hard to maintain so I rejected this in favor of this proposal.

### Alternative 2

Another alternative to creating PVCs for test output storage, you could:
 * create a sidecar container in the test pod
 * share an emptydir volume between the containers
 * have the test pod write to that emptydir any test output
 * change scorecard to not remove the test pod until the
test output is harvested from the sidecar.

This would work, but it means depending on emptyDir storage which makes
per-test configuration of storage probably not possible.  The other
issue might include no way to access the test output after a test concludes
which might limit some use of this feature longer term. Test output PVCs 
could possibly be used for other applications down the road for example.

The upside to this alternative is that you don't depend on a dynamic
storage class being available on the test nodes.


