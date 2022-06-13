---
title: helm-mediatype
authors:
  - "@bowenislandsong"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-May-26
last-updated: 2020-Jun-25
status: provisional
see-also:
  - "./enhancements/csvless-bundles.md"
---

# helm-mediatype

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

 - Can OLM support Chart deprecation from a registry? [Deprecate Charts](https://helm.sh/docs/topics/charts/#deprecating-charts)?
 - Can OLM support at least parts of the Chart [Hooks](https://helm.sh/docs/topics/charts_hooks/) functionalities such as pre/post-install/upgrade? (OLM does not have operator rollback, delete, and test features at this moment.)
 - Should an operator package allow bundles of mixed media types and will upgrades between them sail smoothly?
 - How to avoid OLM/Registry from the previous versions to install/add Helm charts?
 - Can install mode for an operator be default to a value/inferred based on any information in the chart?
 - Is it safe to assume that all [built-in objects](https://helm.sh/docs/chart_template_guide/builtin_objects/), including capabilities, are evaluated during chart render time and should not affect OLM operations?
 - What would be a good CLI UX experience for users to put in chart settings/values on subscription/update?

## Summary

This enhancement adds support to accept `helm chart` media type for [Operator Bundle spec](https://github.com/openshift/enhancements/blob/master/enhancements/olm/operator-bundle.md), [operator-registry](https://github.com/operator-framework/operator-registry), and [OLM](https://github.com/operator-framework/operator-lifecycle-manager) to enable deployment/upgrading helm-based operators. This enhancement covers the design/UX for opm and OLM, and pipeline for building, adding, and installing helm-based operators that follow the [conventions](https://helm.sh/docs/chart_best_practices/conventions/) of `Helm+v3`.

## Motivation
OLM would like to encourage adoption by broader audiences who are more familiar with Helm charts and its way of packaging operators. Currently, Helm users hoping to have OLM to manage their operators would have to convert their existing chart into OLM bundle. As many similarities as there are between the two projects, supporting/installing through either mechanism expects different packaging methods and pipelining. By supporting [Helm chart](https://hub.helm.sh/) as a bundle media type, OLM becomes more approachable to the Helm community without requiring operator authors to explicitly support OLM. It also gives cluster admins the ability to bring Helm-based operators on clusters using OLM.

### Goals

- Support, install, build, and lifecycle `helm chart` typed bundles.
- Report errors and warnings based on the unsupported parts of the `helm chart` typed bundle.
- Support adding `helm chart` typed bundles to an [Operator Index](https://github.com/openshift/enhancements/blob/master/enhancements/olm/operator-registry.md#summary).
- Support package server surfacing data on `helm chart` typed bundles.
- Support rendering chart's Kube manifests based on templates and values including built-in objects.

### Non-Goals

- Support, update, and lifecycle existing operators installed via Helm.
- Building a bundle from helm charts without putting any restrictions on the contents of [templates](https://helm.sh/docs/chart_best_practices/templates/#helm) directory and metadata.
- Support/Replace all capabilities of Helm in deploying Helm charts.
- Enable support for arbitrary Helm charts.

## Proposal

OLM starts to support Helm chart as a type of bundle. The Helm chart as a bundle type will be converted to OLM bundle type upon use and serialized in its raw form. The following user stories analyze different design scenarios/alternatives pivoting between OLM functionalities and the extend of Helm support.

The following is Helm+v3 structure which is used for the rest of the enhancement:
```
helm-chart/
  Chart.yaml          # A YAML file containing information about the chart
  LICENSE             # OPTIONAL: A plain text file containing the license for the chart
  README.md           # OPTIONAL: A human-readable README file
  values.yaml         # The default configuration values for this chart
  values.schema.json  # OPTIONAL: A JSON Schema for imposing a structure on the values.yaml file
  charts/             # A directory containing any charts upon which this chart depends.
  crds/               # Custom Resource Definitions
  templates/          # A directory of templates that, when combined with values,
                      # will generate valid Kubernetes manifest files.
  templates/NOTES.txt # OPTIONAL: A plain text file containing short usage notes
```

The OLM specific information described in the following `bundle.yaml` are non-inferable information from raw Kube manifests. These information should either obtained from [Chart.yaml](https://helm.sh/docs/topics/charts/#the-chartyaml-file) or from an OLM dedicated YAML/JSON file.
```
name                # Name of the Operator (required)
version             # A SemVer version of the Operator (required)
minKubeVersion      # The minimum required Kubernetes cluster version for this operator (required)
installModes*       # Available namespace scope of the operator (required)
description         # A description for the operator
displayName*        # A display name for the operator
keywords            # A list of keywords about this operator
labels*             # A list of labels keyed by name
links               # A list of related links (Documentation, SourceCode, etc)
maintainers         # Operator maintainers contact information
maturity*           # Maturity of the operator
provider*           # Operator Provider
selector*           # A list of label selectors keyed by name
icon                # A URL to an SVG or PNG image to be used as an icon 
replaces*           # A list of previous versions of the operator this version replaces
annotations         # A list of annotations keyed by name 
apiDescriptions*    # A list of API descriptors including required APIServices and provided/required CRDs
  name              # Name of the API formatted `plural.group` for CRDs and `version.group` for APIServices (required)
  version           # Version of the API (required)
  kind              # Kind of the API (required)
  displayName        
  description        
  resources          
  descriptors       # API Descriptors
    descriptorType  # Type of the descriptor which could be one of `status`, `spec`, or `action` (required)
    path             
    displayName      
    description      
    xDescriptors     
    value            
``` 
\* Items that are OLM specific, non-native to `Chart.yaml`, and ignored by Helm. These information needs OLM's attention for parsing.

Special mappings from [Chart.yaml](https://helm.sh/docs/topics/charts/#the-chartyaml-file) -> `bundle.yaml`:
```
kubeVersion -> minKubeVersion  # KubeVersion describes a range of compatible Kube versions which olm takes the minimum value
home + source -> links         # Both links from home and source in Chart.yaml should go to links
```

> Note: The only required information for OLM is install mode.

### Configurable Helm Chart

According to a majority of the popular (stable and Bitnami releases) [helm charts](https://artifacthub.io/packages/search?page=2&repo=stable&repo=bitnami), chart configuration/parameter settings are a combination of a single/small batch of arbitrary value changes (ie. on_minikube=on, jsonOutput=true, etc) and a list of specific value changes (ie. [falco configurations](https://artifacthub.io/packages/chart/stable/falco#configuration)). The most common usages are split among using all default values, change a single/small batch of values, and define a new `value.yaml` to overwrite any and all default values. 

Since the motivation is to encourage a wider adoption of OLM instead of taxing developers to adjust their existing Helm charts, the majority of the adjustment falls under the responsibility of either or both pipeline owners and cluster admins to adapt helm charts onto a cluster managed by OLM. The conundrum is between injecting chart adjustment to the hands of pipeline owners or cluster admins in considering user stories. 

### User Stories for Design #1
---
We support Helm chart as a bundle type while adhering OLM's update strategy to build different channels for a given Helm-based operator version by filling and fixating templated values before pushing them into a registry. Without accepting any value change from cluster admins than what is specified in the Chart folder specifically from `values.yaml`, all templated Kube manifests including CRDs get values as is from the chart, except for namespaces which are configurable before applied to the cluster. All information described in `bundle.yaml` are defined in [Charts.yaml](https://helm.sh/docs/topics/charts/#the-chartyaml-file) along with OLM specific information which would be silently ignored if deployed by Helm. In this design, a mix of Helm-based operators and OLM operators in a single package is allowed.

#### Operator Developers
 - It is not necessary for operator developers to alter their existing Helm chart in order to be consumed by OLM. All alternations could be entirely up to pipeline owners when adding a version of the helm operator to an index. 
 - Operator developers hoping to ease transition for Helm charts into OLM bundles could create/update their Helm-based operator so that `Chart.yaml` could include required information for OLM from `bundle.yaml`.

#### Pipeline Owners
 - To adapt an arbitrary Helm-based operator under OLM's management, pipeline owners need to ensure all OLM specific information are present in the `bundle.yaml`.
 - Pipeline owners will be able to directly use `opm registry add` to target a raw Helm chart directory to add to an `index.db` where the chart raw data will be stored in the database. The conversion of a chart to an OLM bundle will happen during run time to successfully install an operator from the helm chart to cluster. Any values for the chart templates that should be set/alerted from default should be set/changed in `values.yaml` cater to a specific operator version. All [required values](https://helm.sh/docs/howto/charts_tips_and_tricks/#using-the-required-function) in the chart should be filled out.

#### Cluster Admins

 - Cluster Admins will be able to choose operators native to Helm that exists on the registry and subscribe to them. Upon subscription, OLM unpacks raw Helm chart data with default values from the registry and convert it to bundle format for consumption.

#### UX Experience

In this UX, we consider two operations including adding olm specific information to `chart.yaml` and setting values for adding a helm operator of a version to an index. For building/generate a bundle, `opm` no longer supports `package` or `channel` inference and all information about package, channel, default channel needs to be specified. For values/settings, they are either self-sufficiently defined in the chart or they could be trailing behind `helm-set` as a part of bundle build command. Its OLM version of the helm-based operator may diverge from its original version since the same helm operator version can have multiple configurations which are fixated in their own versions.

The following is an opm cmd for building or generating an OLM bundle based on a Helm-based operator. The version of the operator is changed based on channel releases and the chart configuration is set at this step either from value.yaml or the trailing values from helm-set.

```
opm alpha bundle build/generate --directory /helm-chart/0.1.0/ --tag quay.io/example/helm-operator:v0.1.1	--package test-helm-operator --channels stable,beta --default stable --helm-operator-version 0.1.1 --helm-set ...
```

The bundle can then be added to an index as regular bundle since the exact behavior of subscription and upgrades are identical to OLM bundles. The helm-bundle can exist on the same package and channel with OLM native bundles.

```
opm registry add -b quay.io/example/helm-operator:v0.1.1
```

 Pros: 
  - Subscription to helm-operators would following the current OLM operator version/channel pattern where all manifests are fixed for a version in terms of maturity, cluster type, etc.
  - No further adjustments/extra steps in UX when installing/subscribing to Helm-based operators via OLM.
  - Operator updates can be fully automatic without any intervention.
  - A mix of helm and OLM native bundles in the same package and channel is possible.

 Cons: 
  - Operators would require some adjustments and value changes to be included in an index so that it is deployable without any extra settings.
  - Same Helm-based operator version may require different OLM version release to satisfy channel difference.

### User Stories for Design #2 (Update under semver mode and allow cluster admin change value)
---
In this scenario, OLM isolates UX experience for Helm-based operator bundles and allows value settings during install/upgrade. OLM will accept Helm Chart as is and support as much feature as possible. All OLM specific information can be shipped in `olm/` within the Chart directory with `.helmignore` set to the directory:

```bash
./olm/*
```

In this story, we completely forgo the idea of channel and update graph to embrace all Helm concepts. The `bundle.yaml` file should not have the `replace` field and an update should just respect operator version number. All operator bundles in the same package should have Helm media type.

> Note: the poor naming of `olm/bundle.yaml` will change should olm go for this design. The use of folder instead of just `bundle.yaml` is to allow backward compatibility should OLM expand supporting files. Naming suggestions are welcomed.

#### Operator Developers

Developers can add a simple `bundle.yaml` file in the `olm/` directory to make Helm chart deployable by OLM. 

Pro: Charts that are relying on value settings during chart render time does not need any extra changes.

#### Pipeline Owners

Owners who would like to include any chart available/published can validate the compatibility with OLM and add such chart to the index with an `bundle.yaml` file specifying at least the required information.

Pro: A larger number of existing charts would be directly available for OLM to deploy. (Charts that do not rely on Hook feature or contains Kube manifests not supported by OLM.)

Con: Does not guarantee operators in the index would install successfully/work as intended as values can be changed later on.

#### Cluster Admins

Cluster Admins would install/upgrade helm-based operators manually while set/alter values are required or that they can confirm default value as an extra step. This experience would be specific to the helm-based operators upon upgrade/subscription by detecting bundle media type. 

Pro: This would mimic an exact Helm experience for cluster admins including variable settings.

Con: Unlike Helm, OLM does not support rollback operation where cluster admins could potentially try out different values during upgrades.

#### UX Experience

In this situation, the idea of channel disappears. The value setting would be part of subscription/upgrade.

```
opm alpha bundle build/generate --directory /helm-chart/0.1.0/ --tag quay.io/example/helm-operator:v0.1.0
```

```
opm registry add -b quay.io/example/helm-operator:v0.1.0
```

Pros: 
 - Reclaims Helm's update based on version abstraction and the ability to set/alter values during chart render time.
 - When deploying single bundle outside of an index via OLM is possible, a random helm-operator can be deployed via OLM under some restrictions.

Cons: 
- Isolates UX experience between different operator media types. 
- Disables Automatic update feature by OLM.
- A mix of Helm-based operators and OLM operators in a single package is not allowed.

### User Stories for Design #3 (A middle ground)

This scenario combines the concepts of OLM channels and update graph with Helm's version based release. OLM still isolates the UX experience for Helm-based operator bundles and allows value settings during install/upgrade with all [required value](https://helm.sh/docs/howto/charts_tips_and_tricks/#using-the-required-function) filled out so that auto upgrade is still possible. OLM will accept Helm Chart as is and support as much feature as possible. All OLM specific information can be shipped in `olm/` within the Chart directory with `.helmignore` set to the directory:

```bash
./olm/*
```

In this story, a mix of Helm-based operators and OLM operators in a single package/channel is allowed.

Pro: Reclaims Helm's update based on version abstraction and the ability to set/alter values during chart render time. 

Con: Mixing OLM and Helm experience may cause false expectations and confusing UX experience that will eventually outgrow the scope covered within the following situations.

#### Operator Developers

Developers can add a simple `bundle.yaml` file in the `olm/` directory to make Helm chart deployable by OLM. 

Pro: Charts that are relying on value settings during chart render time does not need any extra changes.

#### Pipeline Owners

Owners who would like to include any chart available/published can validate the compatibility with OLM and add such chart to the index with an `bundle.yaml` file specifying at least the required information.

Pro: A larger number of existing charts would be directly available for OLM to deploy. (Charts that do not rely on Hook feature or contains Kube manifests not supported by OLM.)

Con: Does not guarantee operators in the index would install successfully/work as intended as values can be changed later on.

#### Cluster Admins

Cluster Admins would install/upgrade helm-based operators manually while set/alter values are required or that they can confirm default value as an extra step. This experience would be specific to the helm-based operators upon upgrade/subscription by detecting bundle media type.

Pro: This would mimic an exact Helm experience for cluster admins including variable settings.

cons: 
- Unlike Helm, OLM does not support rollback operation where cluster admins could potentially try out different values during upgrades.
- The operator upgrades to a helm-based media type version would disable automatic updates and require value setting/altering. (This could be mitigated by requiring helm-based operator bundles live within the same channel.)

#### UX Experience

In this situation, the idea of channel disappears. The value setting would be part of subscription/upgrade.

```
opm alpha bundle build/generate --directory /helm-chart/0.1.0/ --tag quay.io/example/helm-operator:v0.1.0
```

```
opm registry add -b quay.io/example/helm-operator:v0.1.0
```

### Implementation Details/Notes/Constraints
---

To accept Helm charts as a supported media type for our bundle content, we parse/load raw helm charts using methods defined in [Helm](https://github.com/helm/helm) and fill the information required by OLM and arrange them in the [bundle format](https://github.com/openshift/enhancements/blob/master/enhancements/olm/operator-bundle.md).

#### Helm Linting

An incoming Helm Chart needs to at least pass [Helm linting](https://github.com/helm/helm/blob/d0422e6d82b9d7ddc9c24f02ed1f7d321f9e2646/pkg/action/lint.go). Since all Chart information comes from Helm methods, a `strict` lint run through for an incoming Helm chart before adding such Chart to a registry would be mandatory. 

#### Chart Dependency Strategy

A Helm chart may be dependent on a set of sub-charts as dependencies. Some of these dependencies are presented in the chart's sub-directories while others might be specified in the main `chat.yaml`. Not all dependencies in the subsidiary `chart` are accounted for in the `dependencies` section of the `chat.yaml`, therefore, we should be depending on Helm methods to [list all dependencies](https://helm.sh/docs/helm/helm_dependency_list/).

According to the [dependency resolution from Helm](https://helm.sh/docs/topics/charts/#operational-aspects-of-using-dependencies), the dependencies are gathered as a set of Kube Objects and installed in the order sorted by [Kind](https://github.com/helm/helm/blob/484d43913f97292648c867b56768775a55e4bba6/pkg/releaseutil/kind_sorter.go), Name alphabets, and `Create` over `Update` order. 

According to Helm [implementation](https://github.com/helm/helm/blob/e672a42efae30d45ddd642a26557dcdbf5a9f5f0/pkg/action/install.go#L230-L260), It does not seem possible for OLM to separate dependencies for individual sub-charts but rather install all dependencies at once making it an umbrella operator. The OLM strategy for Helm-based operators should, therefore, be similar to that of Helm and install all dependent Kube manifests at once.

#### Semver Mode and Helm Operator Version

Helm specifies `Chart` and `App` versions where `App` version is only informational. OLM should use [Chart version](https://helm.sh/docs/topics/charts/#charts-and-versioning) as the operator version which will be in Semver format which aligns with OLM's [Semver enhancement](https://github.com/operator-framework/enhancements/blob/master/enhancements/implicit-catalog-versioning.md). This means OLM could enforce [Semver mode](https://github.com/operator-framework/enhancements/blob/master/enhancements/implicit-catalog-versioning.md#mode) when adding Helm chart bundle to index.

### Risks and Mitigations

 - OLM and Helm specialize in different operations and not all capabilities of Helm can be captured by OLM. Despite this effort to include Helm chart as an input type for operator bundles, Helm charts will have a hard time becoming a first-class citizen supported by OLM. To remedy this issue, we propose to toil around Kube manifests centric concept and require less metadata from operator developers. Operators are based on a series of Kube manifests, the less unique information OLM requires from developers the easier to adapt to other or all forms of media types.

 - There remains a risk/concern that as OLM supports Helm charts, Operator developers would prefer producing Helm charts over native OLM bundles so that Operators can run by both Helm and OLM. As OLM grows towards community demands, it may eventually grow dependent on Helm's chart format, concept, and strategy. To this end, OLM is converting Helm charts to bundle format, as OLM develop towards Kube manifests centric concept, and there might be a consensus in the future defining operator packaging between Helm and OLM.

 - Due to Helm's concept of flattening dependent resources, all operator dependencies are collected as a set of resources to form an `umbrella operator` where dependencies can not be updated independently. This would also affect other operators on cluster wishing to update individual dependencies. This problem might start to affect OLM native operators when Helm/OLM operators cross dependencies on each other.

 All hooks features 

 OCI artifact image

 Non-operator Helm charts

## Design Details

### OLM Specific Information

Due to the similarities between OLM and HELM, most information requested by the OLM apart from Kube manifests exist in Helm's [chart.yaml](https://helm.sh/docs/topics/charts/#the-chartyaml-file). However, some restrictions and conversions are needed as described before.

### Validating Supportable Helm Charts

Not all Helm charts with their provided resources can be supported by OLM. The list of OLM supported Kube manifests kinds is available [here](https://github.com/operator-framework/operator-registry/blob/7437af6d40bd9992ddede1aaee6223677b3b896f/pkg/lib/bundle/supported_resources.go#L23). OLM should reject adding a chart to an index should any non-supported kinds exist in a chart including dependent manifests.

The other feature OLM can not support is the Hook feature. Any operator including chart hooks should be rejected.

To fully examine the compatibility of chart with OLM, all dependencies needs to be pulled on bundle building process before adding to registry in order to examine the kinds of all Kube manifests and hook requests from the chart and its dependencies.

### Image Kind Support

Helm is now experimentally supporting OCI images which might not be widely supported by the utilities available on all clusters. This would affect dependency pull time should chart or sub-chart sources are OCI image type. This section should be adjusted to the outcome of [bundle image enhancement](https://github.com/operator-framework/enhancements/pull/34/files).

- A simple solution could be not supporting charts from OCI image typed sources, where OLM disallowing OCI image source during bundle build or registry add time.

- Another solution could be pulling and unpacking all OCI image typed sources during bundle build time so that the helm-typed bundle contains all dependencies from OCI image source and does not need to pull those specific updates (comes free with Helm dependency update strategy) during install/upgrades on cluster. The problem with this approach is that dependencies would not be updated during install/update time.

### Test Plan

The implementation of this enhancement should live in the Operator-Registry repository and will require tests to convert a Helm chart into a bundle with deterministic results. The resulting bundle should also pass static bundle `format` and `content` validations.

An e2e test to add Helm chart to registry and pull down for conversion is required, and the content of the final conversion should be validated and compared to the expected value. The test should also ensure chart pull from a registry is the raw Helm chart content without modification.

### Version Skew Strategy

The previous OLM/Registry would not have the ability to support Helm Chart as a bundle media type and would throw errors when adding/pulling a Helm-based operator bundle from a registry. To avoid unnecessary confusion, registry/olm should disable adding/installing bundles with such media type for previous versions instead of throwing error after initiating such operation.  

