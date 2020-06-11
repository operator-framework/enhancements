---
title: alert-on-operator-updates
authors:
  - "@harishsurf"
reviewers:
  - 
approvers:
  - 
creation-date: 2020-06-10
last-updated: 2020-06-10
status: POC
see-also:
  - "N/A"  
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Validate a catalog to list "bad" bundles (client-side validation)

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary
This enhancement adds additional rules to pre-validate an [operator bundle](https://github.com/openshift/enhancements/blob/master/enhancements/olm/operator-bundle.md) before it's installation on to the cluster. We only care about the resources created and managed by OLM that are packaged within the bundle.

The enhancement also aims at creating a tool that given a [catalog source](https://olm.operatorframework.io/docs/concepts/crds/catalogsource/), it lists all "bad" bundles. Note: "Bad" bundles are those that do not meet the static validation rules.


## Motivation

- Provide static validation for [SAP Gardner objects](https://github.com/operator-framework/operator-lifecycle-manager/pull/1564/files?short_path=de0d44a#diff-de0d44a4ac4cb616e43a9a106cdf5701) that would be packaged as part of the [operator bundle](https://github.com/openshift/enhancements/blob/master/enhancements/olm/operator-bundle.md). 
- The tool could be used by RedHat operators and possibly IBM operators


### Goals

- Define a set of rules for validating a [operator bundle](https://github.com/openshift/enhancements/blob/master/enhancements/olm/operator-bundle.md) before it gets installed on to the cluster. Following resources that are packaged inside a bundle are considered:
    - [Pod Disruption Budget (PDB)](https://kubernetes.io/docs/tasks/run-application/configure-pdb/#specifying-a-poddisruptionbudget) 
    - [PriorityClass](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/#priorityclass)
    - [VerticalPodAutoScaler](https://cloud.google.com/kubernetes-engine/docs/concepts/verticalpodautoscaler)
- Design a tool that lists all "bad" bundles for a given [catalog source](https://olm.operatorframework.io/docs/concepts/crds/catalogsource/)


### Non-Goals

- Server-side validation of Kube manifests as part of a controller

## Open Questions
    
1. Should the tool provide an option to choose a container tool (`--image-builder` flag)?
2. Potential changes to Scorecard?

## Static Validation Rules

### PDB yaml validation
Example PDB.yaml

```yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  minAvailable: 2  
  selector:
    matchLabels:
      app: zookeeper
```
- Either `minAvailable` or `maxUnavailable` should be included, but not both
- `minAvailable` field cannot be set to 0 or less
- `maxUnavailable` cannot be set to greater than the number of pods (TODO: check with Dan)


### PriorityClass yaml validation
Example PriorityClass.yaml
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```
- `globalDefault` should always be set to false


### VerticalPodAutoScaler yaml validation
Example VerticalPodAutoScaler.yaml
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-rec-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind:       Deployment
    name:       my-rec-deployment
  updatePolicy:
    updateMode: "Off"
```
- 

## Proposal

The design proposal has two parts:
- The first discusses how to define rules for validating SAP Gardener objects using golang. It also discusses a potential alternative approach using [CUE](https://cuelang.org/docs/integrations/yaml/#validate-yaml-files)
- The second part discusses a design approach for having a tool to list "bad" bundles in a catalog

## Part 1: Validate bundle objects

### 1. Extending Validation library to add validations in `Golang`
- Extend [BundleValidator](https://github.com/operator-framework/api/blob/master/pkg/validation/internal/bundle.go#L14) to include new validation functions for each of the SAP Gardener objects. (ToDo: Do we add any checks under [BundleFormat](https://github.com/operator-framework/operator-registry/blob/master/pkg/lib/bundle/validate.go#L65) ?)
- The validation function for each kube manifest object would check for conditions specified under [Static Validation Rules](https://hackmd.io/7TlDq9t_RuGjGm8YsEbyHA?both#Static-Validation-Rules).
- (ToDo: 
    -    Do we want to maintain some sort of `struct` for storing various kube-manifest objects as these objects could grow (in the future) or
    -    Wait to see how these new objects are represented in the [Bundle](https://github.com/operator-framework/api/blob/master/pkg/manifests/bundle.go#L11) as part SAP gardener PR and then take a call)
 
### Alternate approach of using Using [CUE](https://cuelang.org/docs/integrations/yaml/#validate-yaml-files) to validate kube manifests

- CUE is a configuration language that allows you to define a `<pdb-checks>.cue` template for validating Kube Manifests like PDBs.
- CUE can be an extendable validation library to accept validate template for a resource like PDBs before applying them on to a cluster.

#### Generate `.cue` files from SAP Gardener object's YAML
- `cue import <filename.yaml>` generates a `.cue` file which can be modified to define rules on certain fields. It provides [lexical elements](https://github.com/cuelang/cue/blob/master/doc/ref/spec.md#lexical-elements) to help define these rules
```cmake
$ cue import pdb.yaml

apiVersion: "policy/v1beta1"
kind:       "PodDisruptionBudget"
metadata: name: "zk-pdb"
spec: {
	minAvailable: >0 & <=7
	selector: matchLabels: app: "zookeeper"
}
```
- For example, a PDB cannot have it's `minAvailable` set to 0. We could enforce this restriction as shown in the above yaml
> **_NOTE:_**  The generation of .cue file and persisting them in a directory is done once when a new k8 object is included as part of the bundle (This is part of the initial setup)

#### Validate YAML against the generated `.cue` 
- The `vet` command of the cue command line tool can validate YAML files using a CUE schema
```yaml
$ cue vet pdb.yaml pdb.cue

spec.minAvailable: invalid value 8 (out of bound <=7):
    ./pdb.cue:5:21
    ./pdb.yaml:6:18
```
- These .cue templates could be stored in a directory to be consumed by `opm` sub command (detailed in part 2)
-  The basic idea is to unpack the catalog image, and for each bundle image, unpack it's content into a temp dir.
-  We then identify the gvk type(k8 object, say PDB) from the manifest and run `cue vet <pdb.cue> <pdb.yaml>` which returns a `bool` with an error message.


#### Pros
- Reduce boilerplate code 
- Not hinging on a third-party operator like OPA Gatekeeper.
- Validation is extendable to customized .cue files
- As a future scope, CUE can be wrapped around using [Tilt](https://tilt.dev/) to be running against a k8 cluster as we make changes to the cue configuration files. [More details here](https://garethr.dev/2019/04/automating-the-cue-workflow-with-tilt/)
- Cue also allows you to define custom cmd flags under CUE cli [(only mentioned in source code)](https://github.com/cuelang/cue/blob/2b0e7cd9f63a190e762d7c802b98528ff80dcb7c/cmd/cue/cmd/cmd.go#L31) but there is no documentation on this

#### Cons
- The set of rules are defined as part of the pipeline (pre-flight checks) and the cluster-admin has no customization option
- Not a lot to offer in terms of flexibility as compared to [OpenPolicyAgent (OPA)](https://kubernetes.io/blog/2019/08/06/opa-gatekeeper-policy-and-governance-for-kubernetes/). With OPA, you can enforce custom policies for create, update and delete of Kubernetes objects without recompiling or reconfiguring the Kubernetes API server or even Kubernetes Admission Controllers. 
- Require users supplying CUE validation template to be familiar with a new type of language. Same is the case for OPA
- Documentation is vague and doesn't have concrete implementation examples
- The error messages are very basic and not customizable


## Part 2: Tool to list "bad" bundles in a catalog
The basic approach will be to extend [`opm`](https://github.com/operator-framework/operator-registry/blob/master/docs/design/opm-tooling.md) tooling to: 
- Modify existing `opm index export` to make `--package` optional so that it pulls down and unpacks all of the bundle images from the index image specified by the existing `--index` flag
- `opm index validate <unpacked bundle location>` - validate all the bundles in the downloaded directory
Few things to consider:
- Exporting of an index image can be time-consuming(at times) and so we need to provide a control option to resume extraction from its previous state.
- There should be an option to unpack only the latest version of the bundle 

#### Modify `opm index export`  
- Currently, the `opm index export` takes in an index image and unpacks only the bundle specified by the `--package` flag. This flag will be made optional and when omitted, it will unpack all of the bundle images and store each of them in its own directory in a location specified by optional `--download-folder`
```shell=
$ opm index export --index=quay.io/operator-framework/upstream-community-operators:latest --download-folder=./bundles
```

- The `--index`points to the catalog source image which has the running index image
- `--download-folder` flag points to a directory location where the extracted file(s) gets stored
- Modify the [`ExportFromIndex`](https://github.com/operator-framework/operator-registry/blob/master/pkg/lib/indexer/indexer.go#L410) API to separate out the code for appregistry format that currently exists and include the new change
- Include a flag `-latest` to unpack only latest versions from each channel using the `channel` table from the registry file
- Unpacking should have a resumable option so as to allow unpacking from a previously stopped point. 
  -  This could be done by uniquely identifying an index image(possibly an image digest) and having a temporary file that caches the unpacked bundles incrementally. This file could be cleared once the unpacking is complete (ToDo: Sketch out specific details)

#### Add a new `opm index validate` command 
```shell=
$ opm index validate <bundles dir-location>
```
-  The command takes in the directory location of the unpacked bundles as a positional argument
-  Reuse the code from [`opm alpha bundle validate`](https://github.com/operator-framework/operator-registry/blob/master/cmd/opm/alpha/bundle/validate.go#L84) to validate the bundles. 

 
## Future Work
- Have server-side/on-cluster validation. [APIServer dry-run](https://kubernetes.io/blog/2019/01/14/apiserver-dry-run-and-kubectl-diff/) could be a viable option.