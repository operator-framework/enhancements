---
title: README describing new command workflows for OLM enabled operators
authors:
  - "@varshaprasad96"
reviewers:
  - "@jlanford"
  - "@dmesser"
  - "@estroz"
approvers:
  - "@jlanford"
  - "@dmesser"
  - "@estroz"
creation-date: 2020-04-09
last-updated: 2020-04-09
status: provisional
---

# README describing new command workflows for OLM enabled operators

## Release Signoff Checklist

- \[ \] Enhancement is `provisional`
- \[ \] Design details are appropriately documented from clear requirements
- \[ \] Test plan is defined
- \[ \] Graduation criteria for dev preview, tech preview, GA
- \[ \] User-facing documentation is created.

## Summary

This proposal provides documentation describing the possible workflows of new commands for generating operator manifests with user inputs and operator bundle for OLM to deploy the operator.

We plan to improve the developer experience for operator authors using Operator SDK with OLM by:
1. Providing an interactive subcommand to get inputs for generating CSV.
2. Providing a single command for generating operator manifests and operator bundle image.

## Motivation

Currently, the `operator-sdk generate csv` command is used for generating CSVs, which scaffolds the project and writes the CSV manifest on disk. This process still requires the developers to manually edit the CSV when any of the required fields are missing. Also, with the deprecation in the use of package manifests for OLM, generating CSVs and creating operator bundle does not require two separate commands. Integrating both the functionalities and providing a single command with an interactive input for collecting the required UI metadata to generate CSV and operator bundle will result in a better developer experience.

## Goals

The proposal is aimed at explaining the resulting process of developing an OLM enabled operator from scratch with the above mentioned improvements.

## Proposal

### README

The following document discusses workflows for creating a new operator and running it with OLM:

Operator creation workflow:

1. Create a new operator project using the SDK Command Line Interface(CLI)
2. Define new resource APIs by adding Custom Resource Definitions(CRD)
3. Define Controllers to watch and reconcile resources
4. Write the reconciling logic for your Controller using the SDK and controller-runtime APIs
5. Use the SDK CLI to build and generate the operator deployment manifests

OLM Workflow:

1. Use `operator-sdk generate bundle` to generate kustomize base templates with CSV metadata and Dockerfile to build the operator.
  * If the kustomize template having the operator and UI metadata is present, it is utilized along with the existing CRDs to build the operator image bundle.
  * If the kustomize template is not present, provide inputs regarding the UI metadata to the interactive prompts appearing further.
2. Use `operator-sdk run --olm` to generate kustomize base template with operator metadata, build and run the operator.

Command details:

1. `operator-sdk generate bundle`:

The `operator-sdk generate bundle` command will generate a kustomize base template containing metadata for the generation of CSV, and will also create or update the Dockerfile which can be used to build operator images. The dockerfile will be created in `./bundle` folder unless specified by the `--output-dir` flag.

```
Usage:
    operator-sdk generate bundle <IMG_REGISTRY>/<TAG> [optional flags]

    Flags:
        -p, --package string    (optional) The name of the package that bundle belongs to. Set if package name differs from project name.
        -d, --kustomize-dir     (optional) Path to the directory containing kustomize base templates. Default: ./config.
        -o, --output-dir        (optional) Optional output directory if operator manifests are to be written on disk. Default: ./bundle

If the command is run on a vanilla Operator SDK project or when the existing kustomize template does not contain UI metadata for generating CSV, the command will be followed by a series of prompts asking the user for inputs. Example:
$ Provide DisplayName for your operator:		
$ Provide version for your operator:		
$ Provide minKubeVersion for CSV:
$ Provide the list of maintainers for the operator:
$ Provide the list of relevant links for the operator:
```
On every run, the bundle is regenerated and based on the available metadata in kustomize template the subcommand prompts appear asking for data regarding the missing fields.

2. `operator-sdk run --olm`:

The `operator-sdk run --olm` command will render Kubernetes resouces from the kustomize base templates conataining operator metadata and will deploy the operator on cluster. It can be used to rapidly build, deploy and test your operator using OLM. The path to the existing kustomize templates or manifests or dockerfile can also be specified using `--bundle-dir` flag.

```
Usage:
    operator-sdk run --olm [optional flags]

  Flags:
      -v, --operator-version    (optional) The version of operator which you would like to run. Default: version specified in the kustomize template for generating CSV.
      -d, --bundle-dir          (optional) Path to the directory containing kustomize base templates, manifests and dockerfile for the bundle (if present). Default: ./bundle.
```

3. `operator-sdk cleanup --olm`:

The `operator-sdk cleanup --olm` deletes the operator and its related resources. Specify the version of the operator to be removed using the `--operator-version` flag, otherwise which the most recently deployed version will be removed.

```
Usage:
    operator-sdk run --olm [optional flags]

  Flags:
      -v, --operator-version    (optional) The version of operator to be deleted. Default: The most recently deployed version.
```

### Create and deploy an app-operator

```sh
# Create an app-operator project that defines the App CR.
$ mkdir -p $HOME/projects/example-inc/
# Create a new app-operator project
$ cd $HOME/projects/example-inc/
$ operator-sdk new app-operator --repo github.com/example-inc/app-operator
$ cd app-operator

# Add a new API for the custom resource AppService
$ operator-sdk add api --api-version=app.example.com/v1alpha1 --kind=AppService

# Add a new controller that watches for AppService
$ operator-sdk add controller --api-version=app.example.com/v1alpha1 --kind=AppService

# Set the username variable
$ export USERNAME=<username>

# Build and push the app-operator image to a public registry such as quay.io
$ operator-sdk build quay.io/$USERNAME/app-operator

# Login to public registry such as quay.io
$ docker login quay.io

# Push image
$ docker push quay.io/$USERNAME/app-operator

# Generate kustomize base templates with UI metadata and dockerfile.
# If you would like to write the operator manifests on disk, specify the path using the flag "--output-dir" 
$ operator-sdk generate bundle <IMAGE_REGISTRY>/<TAG>

# If the kustomize templates do not contain UI metadata for populating CSV, provide inputs for the interactive prompts that would appear further		
$ Provide DisplayName for your operator:		
$ Provide version for your operator:		
$ Provide minKubeVersion for CSV:		
...

# Deploy and test the operator on the cluster using OLM
# If path to the kustomize base templates is not present, they are generated from scratch inside ./bundle folder
$ operator-sdk run --olm 
INFO[0000] loading packages
INFO[0000] generating manifests
...
NAME                            NAMESPACE    KIND                        STATUS
appservice.app.example.com    default      CustomResourceDefinition    Installed
app-operator.<version>        default      ClusterServiceVersion       Installed
...

# Cleanup
$ operator-sdk cleanup --olm
```