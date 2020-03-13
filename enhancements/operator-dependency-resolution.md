---
title: Operator Dependency Resolution
authors:
  - "@kevinrizza"
  - "@dinhxuanvu"
---

# Operator Dependency Resolution

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift/docs]

## Summary

Create a more explicit dependency model that allows operator authors to have a deterministic experience when defining dependencies and gives cluster administrators a better view into exactly what dependencies are installed when installing an operator.

## Motivation

Operator dependencies today are specified on the CRD level. This presents challenges when certain behaviours required by the dependent Operators are not exposed via their APIs. It's also harder to understand the implications of updates of the dependent Operators for developers. Due to the way some Operator authors test known-to-work-well sets of Operators together, dependencies specified by a version feel more natural, intuitive and safe to them.

### Goals

* Give operator authors the ability to define dependencies on specific operators instead of just CRDs and APIs
* Give operator authors the ability to define dependencies on a set of versions (ex. between 1.0.0 and 1.1.0)
* Give cluster admins ability to preview what operator dependencies will be installed (must be deterministic from this point)
* Give cluster admins better reporting when attempting to install incompatible dependency versions (given further move to “an operator is a cluster singleton”)
* Give cluster admins ability to override dependencies
* Retain backwards compatibility of existing CRD/API GVK dependencies

### Non-Goals

* Drive dependencies from any other resource information
* Make existing dependencies completely deterministic (CRD/API GVK dependencies)
* We will not solve all problems for umbrella operators (install sets)
* This will not add new APIs or remove existing APIs (e.g. InstallPlans)

## Proposal

Add a dependency specification file into the Operator Bundle that replaces the values currently defined in the owned CRDs in the CSV. This spec will allow users to reference specific versions and version ranges of operator packages that also exist in the catalogs on cluster. Build a robust dependency resolver that can resolve those version dependency ranges. Improve the cluster admin experience of viewing the results of the resolver based on packages available on a cluster.

### Current Constrains and Limitations:

* Dependencies are defined as a set of owned/required CRD/API GVKs
* Can’t define dependencies on specific operator versions
* OLM assumes latest of default channel for all dependencies
* The same operator can exist in multiple catalogs

### Resolver

#### OLM Specific Resolver Constraints

There are a set of specific constraints that the resolver needs to manage. At a high level, it needs to take a given operator, inspect the dependencies, and select a version to install for each dependency that satisfies the requirement. However, OLM has some existing behavior that will need to be handled and addressed.

* Backwards compatibility will be maintained from the perspective that we should continue to solve for CRD GVK dependencies. The resolver will need to translate those in the same way it translates version and package dependencies.

* The data pulled for the resolution is distributed across several catalogs in several kube namespaces. As a result, we need to weight the dependencies so that we can maintain the preference of same catalog, then same namespace for dependencies

* Replacement chains and update graphs (channels) are constraints. For a fresh install, we can prefer the default channel wherever possible. If an existing subscription owns a dependency, we will constrain the resolution to only consider versions in the channel for the existing subscription. We will also need to build the resolution incrementally in the case that the catalogs update and several new updates are available, stepping through each option and solving for each incremental update.

* Existing operator group and install modes. Once the operator group is created, we should only consider installing operators that have compatible install modes.

* Error messaging for when the resolution is unsatisfiable (due to missing dependencies or dependency conflicts) is something we need to address. See -- https://en.wikipedia.org/wiki/Resolution_(logic)#Propositional_non-clausal_resolution_example. Additionally, we will need to ensure that error messaging is bubbled up to the subscription and operator API.

#### SAT Solver

The implementation of a dependency resolver is a satisfiability problem and NP-Complete. As such, the runtime of a trivial resolver becomes 2^n. For an initial implementation to have relatively good average runtime, we need to transform the dependency problem into a set of boolean statements and feed them into a SAT solver algorithm. Rather than implement one from scratch, our proposal is to use a well tested existing SAT solver library to solve this problem. However, because dependency resolution is a relatively narrow field, there does not exist a generic dependency resolver that would be trivial for our project to import. Instead, we have evaluated several existing solver libraries and compared them with our own constraints and intend to wrap that solver in our own resolver library to satisfy all of them.

Imported SAT solver selection constraints:
* Written in pure Go
* Large project with a healthy ecosystem, good documentation and recent commits
* Common interface that would be easy to replace if needed
* Open Source license

SAT solver needs to take in account several other constraints in order to resolve dependencies correctly. Here is the list of potential resolution constraints:
* InstallMode (OperatorGroup)
* Minimum Kubernetes version
* Native APIs
* OpenShift version (OpenShift only)

After evaluating several libraries, we have chosen https://github.com/irifrance/gini as the SAT solver. This implementation of the solver is general, a popular library, and is written in pure go.

#### Distributed Problem

We also need to consider the performance impact of how the sat solver's dependencies are aggregated. The challenge with distributed catalogs is that the data isn't locally available and the path to resolution will by default create significant load on the existing registry pods for each query. We should attempt to limit this wherever possible and, based on profiling after doing some initial work building the resolver, build a resolution cache of all existing catalog at runtime. The initial design for that cache would require aggregating all data in the registry that the resolver cares about and preparing it for a single query, then having the catalog operator fetch it once whenever the registry is updated.

#### Library

The resolver will be a library that can be imported not just by OLM and used at runtime. This will be a separate component that can be used to determine how OLM will resolve a dependency outside the scope of a running cluster if need be. As a result, we will create a separate git repository for the resolver and have OLM import it.

### Dependency Specification

The dependencies of an operator are listed as a list in `dependencies.yaml` file inside `/metadata` folder of a bundle. This file is optional and only used to specify explicit operator version dependencies at first. Eventually, operator authors can migrate the API-based dependencies into `dependencies.yaml` as well in the future. The ultimate goal is to have `dependencies.yaml` as a centralized metadata for operator dependencies and moving the dependency information away from CSV.

The registry will parse the dependency information in `dependencies.yaml` and add to the database when the bundle is loaded into the registry. OLM will query the dependency information from the registry to facilitate the dependency resolution using sat solver.

### Syntax

The dependency list will contain a `type` field for each item to specify what kind of dependency this is. It can be a package type (`olm.package`) meaning this is a dependency for a specific operator. For `olm.package` type, the dependency information should include the `package` name and the `version` of the package in semver format. We use `blang/semver` library for semver parsing (https://github.com/blang/semver). For example, you can specify an exact version such as `0.5.2` or a range of version such as `>0.5.1` (https://github.com/blang/semver#ranges). In addition, the author can specify dependency that is similiar to existing CRD/API-based using proposed `olm.gvk` type and then specify GVK information as how it is done in CSV. This is a path to enable operator authors to consolidate all dependencies (API or explicit) to be in the same place.

An example of a `dependencies.yaml` that specifies Prometheus operator and etcd CRD dependencies:

```
dependencies:
  - type: olm.package
    name: prometheus
    version: >0.27.0
  - type: olm.gvk
    name: etcdclusters.etcd.database.coreos.com
    group: etcd.database.coreos.com
    kind: EtcdCluster
    version: v1beta2
```

In order to provide a more deterministic dependency resolution, CatalogSource weighting is introduced as a method to order CatalogSource selection when there are multiple CatalogSource providing the same operators and/or APIs. This CatalogSource weighting is specified in CatalogSource and can be modified by cluster admin.

Here is an example of CatalogSource with specified weight:
```
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: catsrc-example
spec:
  displayName: CatalogSource Example
  sourceType: grpc
  image: quay.io/example/catsrc-example:latest
  weight: 10
```
Note: The lower the weight number is the higher priority of the CatalogSource.

The CatalogSource weight setting is meant to be immutable after the CatalogSource is created. Regular users without cluster admin access are not allowed to change this setting to ensure they don't cause any changes in dependency resolution without cluster admin approval or awareness. Currently, the default CatalogSources are deployed via operator marketplace using `OperatorSource` that are shipped along with operator marketplace. Any changes to those OperatorSource are restricted by operator marketplace unless it is modified by cluster admin. This default CatalogSource installation mechanism is going to be changed as a part of OperatorSource API depreciation. The CatalogSource weighting setting should be enforced regardless how the default CatalogSources are installed in the future.

In addition of CatalogSource weighting, the new model will only attempt to resolve dependencies using the default channel. If the specified version of dependency can't be found in default channel, OLM will simply fail to resolve the dependency.

To further enhance the deterministic aspect of dependency resolution, a new field named `indexNamespace` is proposed to enable operator authors to choose a specific source where the operator is provided. This field is essentially a domain/label that can be applied to a specific package or a set of operators. This field combined with operator name will provide a "fully qualified domain name" that is unique for a specific operator. However, the implementation details for this feature is still under development. As a result, this feature is not expected to be included in near-future releases.

#### Provided APIs

The two default types of provided APIs (operator name + version and CRD GVK) will be inferred just by the registry parsing the manifests provided in the bundle. However, we acknowledge that in the future there may be additional APIs provided by operators that are not as easily discoverable.

There are cases when the operators provide/own certain APIs that are important to consider during dependency resolution in order to allow the operator to work properly. This information needs to be declared in the bundle so it can be queried later.

That being said, these additional types will need to be implemented as they become more clear. For now, `dependencies.yaml` will not include any explicit reference to provided APIs.

### Cluster Admin UX

Given the changes to the way OLM will resolve dependencies, it is also necessary to push more detailed data to cluster administrators about what dependencies were installed (or will be installed) and why. To that end, we will continue to use the InstallPlan status to push detailed resolution information to the user. This will include data about what dependencies were installed along with information about where child dependencies came from (essentially, the result of the resolver in detail).

There was also some initial discussion around what the user workflow would be to edit/override dependencies. Given the current state of the APIs that exist (specifically the Subscription and InstallPlan APIs), those would require extensive changes to create a trivial override for dependencies. Even then, there is a configurability problem where a single install overriding a dependency may not be enough and a cluster admin may want to create a cluster wide or namespaced override that is persistent and editable. Initially, there was some discussion about the possibility of a dependency Override API or the possibility of using the Operator API for this purpose, but those discussions are ultimately outside the scope of the immediate (3+ month) timeframe and scope of this enhancement proposal, and will be taken up further in the future.

### User Stories

#### Explicit Operator Dependency Metadata

As an operator author, I want to specify dependencies explicitly in the bundle. For example, I would like to specify that my operator depends on etcd operator version 0.9.0.
* Explicit operator dependency is natural, intuitive and safe to understand
* Avoid unexpected situation where two different operators owning the same CRD/API

Acceptance Criteria:
I can specify explicit dependencies in a file in the bundle and it can be included with other manifests and metadata in the bundle image.

Internal details:
Currently, the plan is to have `dependencies.yaml` as a centralized location for explicit dependency specification. However, we may plan to include that information in Dockerfile/manifest list so that information can be query without unpack the bundle image to see its content.

#### Explicit Operator Dependency Validation

As an operator author, I want to validate dependencies solution in the bundle. For example, I would like to make sure OLM will resolve and install all the dependencies I specify in the bundle correctly.
* Allow operator authors to test dependency resolution for their operators before publishing

Acceptance Criteria:
I can use some toolings to list the operators that will be installed for my operator that will meet all of my dependency requirements that I specify in the bundle.

Internal details:
The proposal sat solver (resolver) will be in a separate repository in operator-framework and it can be imported into operator-registry and `opm` can use the resolver to perform the dependency resolution that OLM will do. Then, operator author can utilize the `opm` binary for testing and validation
The other option is to have a tool that can interact with existing cluster to perform dependency resolution with all existing CatalogSource in the cluster as a "--dry-run" option.

#### Forced Installation and Skip Dependency Resolution
As an operator user, I would like to force OLM to just install the operator without resolving its dependencies. The reason is because I may already install all dependencies manually and I simply want OLM to install this particular operator alone.

Acceptance Criteria:
I can install the operator without resolving its dependencies. I understand I will need to take care of dependencies resolution manually if required.

Internal details:
Operator manifests can be installed manually using a series of commands (`kubectl`). We can provide guideline on how to do this properly.

#### Cluster Admin Dependency Observability

As a cluster admin, I want to see the list of dependencies that are going to be installed when I inspect an operator.
* Allow cluster admin to identify which operators that are going to be installed in order to satisfy the dependency requirement for a particular operator

Acceptance Criteria:
The cluster admin can see the dependency list in InstallPlan in the UI.

Internal details:
InstallPlan does contain a list of its components that it will install such as CSV, CRD and etc. In order to list operator dependency, it may require some changes in InstallPlan API to contain information about operator dependencies (an operator as a whole, not just operator components).

### Implementation Details

### Verify, Run and Test

#### Explicit Operator Dependency Metadata

The operator authors can specific explicit dependencies in `dependencies.yaml` in `/metadata` folder of bundle image.

```
$ tree test
test
├── manifests
│   ├── operator.crd.yaml
│   └── operator.v0.1.0.clusterserviceversion.yaml
└── metadata
    ├── dependencies.yaml
    └── annotations.yaml
```

Test cases will be added to registry test suit to ensure `dependencies.yaml` is formatted correctly and being added to bundle image during the build process.

#### Explicit Operator Dependency Validation

A separate repo will be the home of the resolver (sat solver) and a new CI/test suit will be added. Then, a new command will be added to `opm` binary that can resolve the dependencies of a given operator bundle and index by using the resolver library in the new repo.

```
opm alpha resolve --images quay.io/test/test-operator:v0.0.1 --index quay.io/test/test-index:v0.0.1
```

#### Cluster Admin Dependency Observability

New test cases will be added to validate the potential changes in InstallPlan API to ensure it contains correct expected resolution from the required dependencies in the bundle. Then, the information from InstallPlan will be displayed correctly in the console.
