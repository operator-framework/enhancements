---
title: Split Helm/Ansible into their own repos
authors:
  - "@camilamacedo"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-08-26
last-updated: 2020-08-30
status: implementable
---

# Split Helm/Ansible into their projects

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA

## Summary

Currently, the code implementation required to build the operator image used the Dockerfile of the projects built with the SDK CLI tool are in the directories [operator-sdk/internal/ansible/](https://github.com/operator-framework/operator-sdk/tree/master/internal/ansible) and [operator-sdk/internal/helm/](https://github.com/operator-framework/operator-sdk/tree/master/internal/helm). Also, the code used to scaffold the files are in the [operator-sdk/internal/plugins/ansible/v1](https://github.com/operator-framework/operator-sdk/tree/master/internal/plugins/ansible/v1) and [operator-sdk/internal/plugins/helm/v1](https://github.com/operator-framework/operator-sdk/tree/master/internal/plugins/helm/v1)

Note that, a new release version of their images (e.g. `quay.io/operator-framework/helm-operator:v1.0.0` and `quay.io/operator-framework/ansible-operator:v1.0.0`) is generated for all releases even when no changes have been made that would impact these images.

Also, all operators who are built with the tool ought to adopt a standard for its configurations and helpers with a design which allows its customizations and caveats. Otherwise, it can bring many complexities for users and to its maintainers/contributors to keep these types maintained to provide new features. For example, imagine the complexity required to maintain and implement features such as to integrate the projects with OLM without assuming that all types will respect these standards. 

 Ansible and Helm plugins were developed by copying these templates from upstream(Kubebuilder), and now all these files are duplicated on the repository.   

This proposal has the goal to describe how we can split Helm/Ansible projects and then, decoupled from the CLI tool. However, the current approach adopted in the project to develop these plugins are already hard enough to keep maintained since the files are duplicated. For Maintainers/contributors are capable of identifying what is shared and what is customized for each type is not trivial at all currently. Also, it is indirectly forcing Ansible/Helm contributor to have in-depth knowledge in the purpose of each config file and template copied from upstream to not deviate from its original purpose.   

By splitting Helm and Ansible in their repos, we will increase the chance to make these projects deviate as its complexities and started no longer been compatible with SDK features and or bring more complexities for the tool implementations and customizations as its users. 

In this way, as part of this proposal, we are trying to improve the maintainability, readability and reusability and then, mitigate these risks and complexities by proposing making upstream a source of truth and its utils package base. 

**Note** It is a follow up of the proposal [Split SDK into multiple projects](split-sdk-into-multiple-projects.md).   

## Open Questions

1. Any reason for not generate the samples on the Operator-Type repos? 
1. The templates copied from Kubebuilder were set in Kubebuilder as internal by us in the plugin phase 1 implementation. We use the plugins, and the templates are part of them, so I do not see any reason for we not be allowed to import its templates as well. However, do you have any strong argumentation to object it?      
1. The steps to reach the goal for the Operator Types projects have their own docs subdomains `ansible.operatorframework.io`, and `helm.operatorframework.io` shows have certain complexity. Have we any reason for it has not its specific proposal?

## Motivation

- Improve the maintainability, readability and reusability by adopting concepts such as [separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) and [Single Responsibility principle](https://en.wikipedia.org/wiki/Single-responsibility_principle). 
- Facilitate the development process and encourage more contributions
- Allow Lifecycle releases per type
- Minimize the CI processing time
- Upstream as single source of truth to reduce the complexities and mitigate the risks

### Goals

- The `operator-framework/operator-sdk` contains only the CLI code and the calls for the plugins.
- The `operator-framework/helm-operator` repository only contains `helm-operator` code used to generate its based-image and Helm plugin code.
The `operator-framework/ansible-operator` repository only contains `ansible-operator` code used to generate its based-image and Ansible plugin code.
- Ansible/Helm/SDK repositories with lint check to ensure that all expose methods will be documented
- Ansible/Helm repositories covered with unit tests
- Ansible/Helm repositories with their testdata sample
- Helm/Ansible binaries should be created/released from their repos and no longer from SDK CLI.
- The paths `kubebuilder/pkg/plugin/v2/scaffolds/internal/templates/` and `kubebuilder/pkg/plugin/v3/scaffolds/internal/templates/` renamed to `kubebuilder/pkg/plugin/v2/scaffolds/templates/` and to `kubebuilder/pkg/plugin/v3/scaffolds/templates/` respectively.
- Ansible/Helm using templates that are not customized from upstream 

### Non-Goals

- Steps required to achieve the goal of the Operator Types projects have their own docs subdomains `ansible.operatorframework.io` and `helm.operatorframework.io`  
- Improve the test coverage of the pre-existing code  

## Proposal

### User Stories 

#### Make upstream plugin Boilerplates public

- I am a maintainer/contributor, I do NOT want make many copies of upstream templates which does not require customizations, so that I can spend effort to maintain and read just what is specific for my types and in what adds values to it.
- I am a maintainer/contributor, I want to develop a new SDK command assuming the basic and common configuration is always the same for any type, so that I do not need to cover my code with a lot of conditions and different implementation logic for each type
- I am a maintainer/contributor, I want to use the config base templates from upstream, so that I will not spend any effort to update them on both Ansible and Helm repositories when these files changes.
- I am a maintainer/contributor, I want to use the config base templates from upstream, so that I will do not need fully understand its common usage to contribute to the project and then it mitigates the chance of I change the common behaviours because of some misunderstand or wrong assumption. (e.g scenario faced in the development process: start to use the edit and view roles created as a pre-requirement for the operator works when its purpose is just be a helper for admins)
- I am an operator developer, I want to see all Operator Types following the same standard configuration when possible, so that I switch between the available types to address more appropriated my needs with less complexity.

#### Split Helm/Ansible into their own repos

- I am a maintainer/contributor, I want to see the projects adopting concepts such as [separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) and [Single Responsibility principle](https://en.wikipedia.org/wiki/Single, so that I can maintain, read and re-use them easily
- I am a Helm/Ansible operator developer, I want to need to update my projects only when the new release address changes my type of project, so that I can easily keep my operator projects updated. 
- I am a Helm/Ansible operator developer, I want to need to update my projects only when the new release address changes my type of project, so that I can easily keep my operator projects updated. 

#### Provide the based-operator image implementation as lib

- I am a maintainer/contributor, I want to provide a chance for users address their specific needs, so that I can keep the supported projects to address the common requirements, however, I can offer a solution for the specific needs   
- I am an operator developer, I want to be able to use the Helm/Ansible implementations, so that I can develop my own Helm/Ansible based-operator images customized according to my needs.   

#### Lint check for docs on Ansible/Helm repositories

- I am a maintainer/contributor, I want to the projects use godocs generated automatically, so that I need only ensure that the code is properly commented and that the lint will not allow introducing changes without comments
- I am an operator developer, I want to have access to a good documentation of these projects, so that I know how can I use the libs provided

#### Operator Types Images   

- I am a maintainer/contributor, I want to see the CI building and pushing the images when a PR is merged and when a tag is pushed, so that I could not need spend effort on it
- I am a maintainer/contributor, I want to be able to use targets to generate the images, so that I can use them to perform the tests in the development process

#### Operator Types Binary 

- I am a maintainer/contributor, I want to use a Makefile target to build the binary, so that I would spend low effort on the release process to provide its asset
- I am an Ansible/Helm operator developer, I want to be able to use the command `make run`, so that I can run the project locally 
- I am a maintainer/contributor, I want to be able to use the Operator type binary to run the CLI commands for the plugin, so that I can work with the project as standalone and I can use it to implement e2e test.

#### Changelog for Operator Types

- I am a maintainer/contributor, I want to use a Makefile target to generate the changelog and Migration Guide, so that I would spend low effort on the release process

#### Tests for Operator Types

- I am a maintainer/contributor, I want to use check the Operator Types covered with unit tests, so that I can ensure the code used to build based operator images 
- I am a maintainer/contributor, I want to use check the Operator Types covered with e2e tests, so that I can ensure the projects that can be build with via any tool that will call its plugin

### Risks and Mitigations

Currently, when we start to support in SDK a new plugin version from upstream we can face a breaking change which will affects the specific SDK features such as for Scorecard and OLM. The same scenario might to be faced when we bump Ansible/Helm versions on SDK. 

### Implementation Details/Notes/Constraints 

**Details:**

- The Operator Types (Ansible/Helm) binaries should be released from their repos and no longer via SDK
- The operator based image implementation used on the Operator-Types (Ansible/Helm) should be public 
- Ansible/Helm repositories should contain the same changelog feature that is provided in SDK.

**Notes:**

As a longer-term option solution, after the second phase of plugins are implemented, and its design be consolidated, we might be able to provide from upstream a base config plugin which could scaffold the `config/` directory as a common Makefile for any Operator Type be able to re-use it and perform its customizations as scaffold its specific languages type instead of Go.  SDK also could provide as lib its plugins for customizations to be done on top of it.

This solution could be helpful to provide alternatives to developers quickly built their Operator Types such as to support [Argo](https://github.com/operator-framework/operator-sdk/issues/2497) and to allow the common needs to be re-used and be easily support new Operator Types such as [Hybrid Operators](https://github.com/operator-framework/operator-sdk/issues/670) or with other languages see [Meta: Operator SDK's in other languages #2745](https://github.com/operator-framework/operator-sdk/issues/2745), so that we could easily join these new Operator Types as incubator projects and define as pre-requirement following the same standards and workflows to work with the SDK features. However, it is definitely out of the scope of this EP. 

## Design Details

### Diagram of the dependencies between the projects

<img width="862" alt="Screen Shot 2020-09-07 at 09 56 04" src="https://user-images.githubusercontent.com/7708031/92369070-6ffeac80-f0f0-11ea-9b34-794e06c46aec.png">

### Definition of domain responsibility

| Project | domain responsibility 	| 
|---	|---	|
|  `operator-framework/operator-sdk`	|  Code required to implement the SDK CLI commands|  
|  `operator-framework/operator-lib`	| It has the purpose is to provide helpers and useful features for the projects which are scaffolds by SDK tool. It is a dependency that can be add to the user's projects which are built with the tool.   |  
|  `operator-framework/helm-operator`	|  Code implementation used to generate the based operator image and plugin used to scaffold Helm Operator projects | 
|  `operator-framework/ansinle-operator`	|  Code implementation used to generate the based operator image and plugin used to scaffold Ansible Operator projects | 

### Layout of the Operator Type Projects 

The following shows the final `helm-operator` layout with the description from where its implementation came from. See that `ansible-operator` will be very similar and has the same structure in the SDK repository which means that the following information is valid for both. 

```
.
└── helm-operator
    ├── bin/ (binaries reequired to run/test the project which should not be commited in the repo. We need esnure that they can be generated via makefile targets and will be ignored via .gitignore)
    ├── Dockerfile (define based-operator image)
    ├── LICENSE
    ├── Makefile (contains the targets with the development and maintainer purposes of the project such as: to test, to release and go fmt)
    ├── README.md 
    ├── go.mod
    ├── go.sum
    ├── main.go (used to run the project locally. Move operator-sdk/internal/cmd/helm-operator/run from SDK to here)
    ├── hack (store the shell scripts)
         └── test (contains the scripts required to run the e2e tests)        
    ├── pkg 
         └── helm (contains all based-operator image implementation and helpers. Move operator-sdk/internal/helm/ from SDK to here)        
    |   └── plugin 
            └── v1 (we will move operator-sdk/internal/plugins/helm/v1/ from SDK to here)
    ├── testdata
    │   └── test-chart-0.1.0.tgz (mock data to test the project if required)
    │   └── example (sample project that should be automatically generated based on the pr changes)
    ├── e2e-tests (should contain the e2e test of the projects. Note that it means copy operator-sdk/test/e2e-helm from SDK to here and cleanup the SDK test implementation since it will no longer required to test/ensure the Helm features)
    └── version
        └── version.go (Move operator-sdk/cmd/helm-operator/version from SDK to here)
```

### Code from SDK that should be moved 

**Helm**

| From `operator-sdk` | To `operator-helm` 	| Description  	|
|---	|---	|---	|
|  internal/cmd/helm-operator/run 	|   main.go	|   used to run the project locally	|
| internal/cmd/helm-operator/version  	|  version.go 	| define the version of the project and it used to build the bin	|
|  internal/helm/ 	|  pkg/ 	|   has the implementations used to generated the Helm based-operator image	|
|  internal/plugins/helm/v1/ 	|  pkg/plugin/v1 	|   has the implementation customized to scaffold the helm projects 	|
|  test/e2e-helm 	|  e2e-tests 	|   has the e2e test for the Helm project 	|
|  hack/tests/e2e-helm.sh	|  hack/tests/e2e-helm.sh 	|   has the e2e test for the Helm project 	|
| * hack/generate/samples/projects/*_helm_sample.go 	|  hack/generate/samples/projects/*_helm_sample.go |  It is not currently implemented. See the section `Operator Types Samples` on this document |

**Ansible**

| From `operator-sdk` | To `ansible-helm` 	| Description  	|
|---	|---	|---	|
|  internal/cmd/ansible-operator/run 	|   main.go	|   used to run the project locally	|
| internal/cmd/ansible-operator/version  	|  version.go 	| define the version of the project and it used to build the bin	|
|  internal/ansible/ 	|  pkg/ 	|   has the implementations used to generated the ansible based-operator image	|
|  internal/plugins/ansible/v1/ 	|  pkg/plugin/v1 	|   has the implementation customized to scaffold the ansible projects 	|
| test/e2e-ansible 	|  ansible-operator/e2e-tests 	|   has the e2e test for the ansible project 	|
|  hack/tests/e2e-ansible.sh	|  hack/tests/e2e-ansible.sh 	|   has the e2e test for the ansible project 	|
|  hack/tests/e2e-ansible-molecule.sh	|  hack/tests/e2e-ansible-molecule.sh 	|   has the e2e test the ansible project with molecule 	|
| test/ansible-memcached/	|  e2e-tests/ansible-memcached/ |   has the static mock data which still used by molecule tests	|
|  test/ansible/	|  e2e-tests/ansible/ |   has the static mock data which still used by molecule tests	|
| * hack/generate/samples/projects/*_ansible_sample.go 	|  hack/generate/samples/projects/*_ansible_sample.go |  It is not currently implemented. See the section `Operator Types Samples` on this document |

### Usage of Helm/Ansible plugins in SDK 

Helm and Ansible plugins will be imported and added to `operator-sdk`'s CLI much like they [currently are](https://github.com/operator-framework/operator-sdk/blob/master/internal/cmd/operator-sdk/cli/cli.go#L62-L63), except from their respective repositories.

However, note that in the [operator-sdk/internal/plugins](https://github.com/operator-framework/operator-sdk/tree/master/internal/plugins/) its implementation would be very similar to the [Go plugin](https://github.com/operator-framework/operator-sdk/tree/master/internal/plugins/golang/v2). SDK will consume the `helm-operator` and `ansible-operator` to build the projects much like its consume Kubebuilder to build the Go projects.   

### CLI usage on Operator Types project 

The binaries would also need to provide the CLI features to allow build the projects and test them without SDK CLI tool. However, this implementation came from Kubebuilder and its implementation would like it is done in [operator-sdk/internal/cmd/operator-sdk/cli/cli.go](https://github.com/operator-framework/operator-sdk/blob/master/internal/cmd/operator-sdk/cli/cli.go). 

### Update website SDK docs

 In this way, if we can accomplish all goes described here before being able to create the Proposal to describe and specify the steps required to achieve the goal of the Operator Types projects have their own docs subdomains `ansible.operatorframework.io` and `helm.operatorframework.io` or if we decide not move forward with it as well we can define that the website docs should be updated when a new version of these repos are bumped on the CLI. 

Note that, the above scenarios requires small changes in the docs usually. Also, changes in the tutorials will be more common when the upstream (Kubebuilder) implementation change and this case, also will be required update the website docs when a new Kubebuilder version be bumped. So, it shows achievable. 
  
### Operator Types Samples 

See the proposal [Automatic Samples Generation using Go language](https://github.com/operator-framework/enhancements/pull/47/files). The implementation to generate the samples (SampleContext) as it helpers could be on upstream. 
 
Then, each Operator Type should be able to build its own samples in the `testdata/` directory.

### Scope of Tests per project

- Operator Types (`helm-operator` and `ansible-operator`) should contains tests that will cover its libs and the project scaffold by them.  

- SDK CLI tool (`operator-sdk`) should contain the test required to cover the CLI commands and e2e tests which will cover the specific customizations made on top of helm/ansible plugins for example olm integrations. 

## Test Plan

Besides Ansible/Helm be decoupled from SDK CLI tool their tests should still ensure their functionalities. 

## Alternatives

### For Helm

For Helm, we have the [joelanford/helm-operator](https://github.com/joelanford/helm-operator) and the need to use the new pkg implementation. An alternative suggested was to use this repository as base for the split. However, with this approach would be harder ensure that no breaking changes, issues or regressions would be introduced since it also contains a new implementation for the base image. See the proposal [Proposal to introduce New based-operator image implementation #45](https://github.com/operator-framework/enhancements/pull/45).

Another alternative option could be to create a lib such as `SDK Operator Build Lib` with the code implementation that is common and used to build any kind of operator. (e.g. plugins, utils, templates). However, it would increase the effort to keep the projects maintained.  

**NOTES**

Following the script used to gen the diagram via https://www.planttext.com/.
 
```
@startuml

package "operator-sdk"  #34ebba {
    component [User Interface \n CLI tool] as SDK #FFFFFF
}

 
package "helm-operator"  #6fb2ed {
    component [+Helm plugin \n +Helm lib for based images] as Helm #FFFFFF
}

package "ansible-operator" #6fb2ed  {
    component [+Ansible plugin \n +Ansible lib for based images] as Ansible #FFFFFF
}

package "Kubebuilder" #34ebba {
    component [+Go plugin \n +Lib for CLI plugins] as Kube #FFFFFF
}

package "operator-lib" #9eb5e8 {
    component [+Lib \n +features to be used by the projects] as Lib #FFFFFF
}

Helm -up-> SDK
Ansible -up-> SDK
Kube -left-> Helm
Kube -up-> SDK
Kube -right-> Ansible
Lib -up-> Helm
Lib -up-> Ansible
@enduml
```
