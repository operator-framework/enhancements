---
title: csvless-bundles 
authors:
  - "@exdx"
  - "@njhale"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-03-09
last-updated: 2020-06-19
status: implementable
---

# CSVless Bundles 

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift/docs]

## Summary

Describes the specification of a new bundle mediatype that alleviates the need for end users to write and maintain `ClusterServiceVersions` (CSVs).

## Motivation

Operator authors find CSVs difficult to write and maintain. Expressing an OLM-managed operator as a set of related kubernetes objects is more intuitive to users. Supporting plain kubernetes manifests makes OLM more approachable to the wider community, which is relevant in light of the Operator Framework attempting to become a CNCF project.

### Goals

- Simplify the UX of creating and maintaining bundles
- Define a `registry+v1` equivalent bundle spec that doesn't require a CSV
- Describe `opm` commands for building and installing `registry+v2` bundles
- Support adding `registry+v2` bundles to an [Operator Index](https://github.com/openshift/enhancements/blob/master/enhancements/olm/operator-registry.md#summary)
- Support packageserver surfacing data on plain manifest bundles

### Non-Goals

- Consuming Helm charts
- Enable installing arbitrary kubernetes resources
- Design [OperatorHub](https://operatorhub.io) support for `registry+v2` bundles

## Proposal

Add support for a new mediatype, `registry+v2`, to the [Operator Bundle spec](https://github.com/openshift/enhancements/blob/master/enhancements/olm/operator-bundle.md), [operator-registry](https://github.com/operator-framework/operator-registry), and [OLM](https://github.com/operator-framework/operator-lifecycle-manager) that enables packaging an operator without defining a [ClusterServiceVersion](https://olm.operatorframework.io/docs/concepts/crds/clusterserviceversion/).

### User Stories

#### File Format

As an Operator Author, I want a way to store my `registry+v2` bundle as set of files and directories, so that I can:

- iterate on its content locally
- use standard source control tooling to:
  - persist and version it
  - drive pipelines

##### UX

A `registry+v2` bundle will be represented on-disk as a directory tree consisting of a root and two siblings:

- the root is any arbitrary directory in the filesystem that contains the siblings at depth 1
- the `manifests` directory, contains the resource manifests used to generate the final operator manifests at runtime
  - a flat directory
  - may only contain kubectl-able YAML streams
- the `metadata` directory, which contains all metadata
  - doesn't need to be flat
  - contains metadata required for OLM to manage the operator in a `bundle.yaml` file
  - contains an `annotations.yaml` file which contains labels to be projected onto the operator's [Image Format](#image-format).
  - may contain additional arbitrary metadata

_Ex: Kiali Operator_

```sh
$ tree kiali
kiali
├── manifests
│   ├── crds.yaml
│   ├── deployment.yaml
│   └── rbac.yaml
└── metadata
    ├── annotations.yaml
    └── bundle.yaml

2 directories, 5 files
```

In order to support all of the features of a `registry+v1` bundle without a CSV, OLM must be given and/or infer the information that would have been provided by the CSV via other resources. Here are the basic requirements and inferences for `registry+v2` bundles.

The `manifests` directory must have at least one `Deployment` resource (to take the place of a CSV's embedded `DeploymentSpec`).

_Ex. kiali/deployment.yaml_

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kiali-operator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kiali-operator
  template:
    metadata:
      name: kiali-operator
      labels:
        app: kiali-operator
        version: v1.9.1
    spec:
      serviceAccountName: kiali-operator
      containers:
      - name: ansible
        command:
        - /usr/local/bin/ao-logs
        - /tmp/ansible-operator/runner
        - stdout
        image: quay.io/kiali/kiali-operator:v1.9.1
        imagePullPolicy: "IfNotPresent"
        volumeMounts:
        - mountPath: /tmp/ansible-operator/runner
          name: runner
          readOnly: true
      - name: operator
        image: quay.io/kiali/kiali-operator:v1.9.1
        imagePullPolicy: "IfNotPresent"
        volumeMounts:
        - mountPath: /tmp/ansible-operator/runner
          name: runner
        env:
        - name: WATCH_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['olm.targetNamespaces']
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: OPERATOR_NAME
          value: "kiali-operator"
      volumes:
      - name: runner
        emptyDir: {}

```

Any `CustomResourceDefinitions` found in the streams are included in the bundle as provided APIs (a.k.a. "owned CRDs").

_Ex: kiali/crds.yaml_

```yaml
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: kialis.kiali.io
  labels:
    app: kiali-operator
spec:
  group: kiali.io
  names:
    kind: Kiali
    listKind: KialiList
    plural: kialis
    singular: kiali
  scope: Namespaced
  subresources:
    status: {}
  version: v1alpha1
  versions:
  - name: v1alpha1
    served: true
    storage: true
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: monitoringdashboards.monitoring.kiali.io
  labels:
    app: kiali
spec:
  group: monitoring.kiali.io
  names:
    kind: MonitoringDashboard
    listKind: MonitoringDashboardList
    plural: monitoringdashboards
    singular: monitoringdashboard
  scope: Namespaced
  version: v1alpha1

```

The `bundle.yaml` file consolidates all of the additional info needed by OLM to manage the life cycle of the operator. The set of additional info is exactly (w/ required info in **bold**):

- **name**
- **version**
- minimum kubernetes version
- description
- display name
- keywords
- labels
- links
- maintainers
- maturity
- provider
- selector
- icon
- API descriptions

The naming of `bundle.yaml` is strict and is used to identify the file. 

_Ex. bundle.yaml_

```yaml
operators.operatorframework.io.bundle.mediatype: registry+v2
name: kiali-operator.v1.9.1
version: 1.9.1
minKubeVersion: 1.17.3
description: |
    ## About this Operator

    ### Kiali Custom Resource Configuration Settings

    For quick descriptions of all the settings you can configure in the Kiali
    Custom Resource (CR), see the file
    [kiali_cr.yaml](https://github.com/kiali/kiali/blob/v1.9.1/operator/deploy/kiali/kiali_cr.yaml)

    Note that the Kiali operator can be told to restrict Kiali's access to
    specific namespaces, or can provide to Kiali cluster-wide access to all
    namespaces.
    
    ...
displayName: Kiali Operator
keywords: ['service-mesh', 'observability', 'monitoring', 'maistra', 'istio']
links:
- name: kiali-operator
  url: https://www.prometheus.io/
maintainers:
- name: Kiali Developers Google Group
  email: kiali-dev@googlegroups.com
maturity: beta
provider:
  name: Kiali
selector:
  matchLabels:
    name: kiali-operator
icon:
  - base64data: PHN2ZyB3a...
    mediatype: image/svg+xml
version: 1.9.1
apis:
- apiVersion: kiali.io/v1alpha1
  kind: Kiali
  description: A configuration file for a Kiali installation.
  displayName: Kiali
  version: v1alpha1
  resources:
  - kind: Deployment
    version: apps/v1
  - kind: Pod
    version: v1
  - kind: Service
    version: v1
  - kind: ConfigMap
    version: v1
  - kind: OAuthClient
    version: oauth.openshift.io/v1
  - kind: Route
    version: route.openshift.io/v1
  - kind: Ingress
    version: extensions/v1beta1
  descriptors:
  - displayName: Authentication Strategy
    description: "Determines how a user is to log into Kiali. Choose 'login' to use a username and passphrase as defined in a Secret. Choose 'anonymous' to allow full access to Kiali without requiring credentials (use this at your own risk). Choose 'openshift' if on OpenShift to use the OpenShift OAuth login which controls access based on the individual's OpenShift RBAC roles. Default: openshift (when deployed in OpenShift); login (when deployed in Kubernetes)"
    path: spec.auth.strategy
    capabilities:
    - 'urn:alm:descriptor:com.tectonic.ui:label'
  - displayName: Kiali Namespace
    description: "The namespace where Kiali and its associated resources will be created. Default: istio-system"
    path: spec.deployment.namespace
    capabilities:
    - 'urn:alm:descriptor:com.tectonic.ui:label'
  - displayName: Secret Name
    description: "If Kiali is configured with auth.strategy 'login', an admin must create a Secret with credentials ('username' and 'passphrase') which will be used to authenticate users logging into Kiali. This setting defines the name of that secret. Default: kiali"
    path: spec.deployment.secret_name
    capabilities:
    - 'urn:alm:descriptor:com.tectonic.ui:selector:core:v1:Secret'
  - displayName: Verbose Mode
    description: "Determines the priority levels of log messages Kiali will output. Typical values are '3' for INFO and higher priority messages, '4' for DEBUG and higher priority messages (this makes the logs more noisy). Default: 3"
    path: spec.deployment.verbose_mode
    capabilities:
    - 'urn:alm:descriptor:com.tectonic.ui:label'
  - displayName: View Only Mode
    description: "When true, Kiali will be in 'view only' mode, allowing the user to view and retrieve management and monitoring data for the service mesh, but not allow the user to modify the service mesh. Default: false"
    path: spec.deployment.view_only_mode
    capabilities:
    - 'urn:alm:descriptor:com.tectonic.ui:booleanSwitch'
  - displayName: Web Root
    description: "Defines the root context path for the Kiali console, API endpoints and readiness/liveness probes. Default: / (when deployed on OpenShift; /kiali (when deployed on Kubernetes)"
    path: spec.server.web_root
    capabilities:
    - 'urn:alm:descriptor:com.tectonic.ui:label'
- apiVersion: monitoring.kiali.io/v1alpha1
  kind: MonitoringDashboard
  description: A configuration file for defining an individual metric dashboard.
  displayName: Monitoring Dashboard
  descriptors:
  - displayName: Title
    description: "The title of the dashboard."
    path: title
    capabilities:
    - 'urn:alm:descriptor:com.tectonic.ui:label'
```

_**Note:** `capabilities` is the equivalent of the `x-descriptors` field found in CSVs._
_**Note:** The `path` field of a `descriptor` informs its "descriptor type" at install time. Only `.spec` and `.status` root paths are allowed and would map respectively to spec and status descriptors of an equivalent CSV. What was known as action descriptors -- with respect to CSVs -- are not supported, and in most cases can be exchanged for an equivalent spec or status descriptor anyway._

The `annotations.yaml` file contains the set of labels to be projected onto the container image representation of the bundle. More importantly, it contains the top-level discriminator field used to identify the mediatype of the bundle on-disk: `operators.operatorframework.io.bundle.mediatype: registry+v2`.

#### Build a Bundle Image Without a CSV

As an Operator Author, I want to build an Operator Bundle image from the same set of Kubernetes manifests I would use to deploy an operator manually, without OLM, so that I can avoid writing a CSV.

##### UX

Add a new command to `opm`, `opm bundle init`,  to initialize the current directory as a bundle, which has:

- a positional parameter -- `src` -- representing kubectl-able YAML streams to include in the bundle as manifests; can be a path to a directory of YAML streams or `-` to accept a stream from stdin.
- an option for specifying the target mediatype -- `-m, --mediatype` -- and attempts to infer it from the manifests in `src` if not specified
- an option enabling `Dockerfile` generation -- `-d, --generate-dockerfile` -- which defaults to disabled

When no source manifests are present in `src`, default resources are generated. Additionally, if a bundle already exists in the current directory, the command will return a non-zero exit code in the event the operation would overwrite any content.

_Ex. Initialize defaults for `registry+v1` in the current empty directory_

```sh
$ tree .
.

0 directories, 0 files
$ opm bundle init --mediatype 'registry+v1' .
# <logs>
$ tree .
.
├── manifests
│   └── csv.yaml
└── metadata
    ├── annotations.yaml
    └── dependencies.yaml

2 directories, 4 files
```

_Ex. Initialize defaults for `registry+v2` in the current empty directory and generate a dockerfile_

```sh
$ tree .
.

0 directories, 0 files
$ opm bundle init --mediatype 'registry+v2' -d .
# <logs>
$ tree .
.
├── Dockerfile
├── manifests
│   └── deployment.yaml
└── metadata
    ├── bundle.yaml
    ├── annotations.yaml
    └── dependencies.yaml

2 directories, 5 files
```

_Ex. Fail to detect mediatype on empty directory_

```sh
$ tree .
.

0 directories, 0 files
$ opm bundle init .
# <log: "unable to infer mediatype when no manifests are present">
```

_Ex. Infer mediatype from src manifests_

```sh
$ tree kiali-csvless
kiali-csvless
├── crds.yaml
├── deployment.yaml
└── rbac.yaml
$ opm bundle init kiali-csvless
# <logs>
$ tree .
.
├── kiali-csvless
│   ├── crds.yaml
│   ├── deployment.yaml
│   └── rbac.yaml
├── manifests
│   ├── crds.yaml
│   ├── deployment.yaml
│   └── rbac.yaml
└── metadata
    ├── annotations.yaml
    ├── bundle.yaml
    └── dependencies.yaml

3 directories, 9 files
```

Add a new command to `opm`, `opm bundle build`, which takes as input a directory tree -- described in [File Format](#file-format) -- and outputs an Operator Bundle image.

_Ex._

```sh
$ opm bundle build . file://bundle.tar.gz # detect the mediatype of the current directory, build it, and if successful, write the image tar.gz to a file
$ opm bundle build . image://quay.io/my/bundle:v1.0.0 # detect the mediatype of the current directory, build it, and if successful, push the image to a remote reference
```
#### Additively Build an Index

As a Pipeline Maintainer, I want to additively build an index of `registry+v2` bundles so that I can release new bundles to an existing index.

##### UX

Extend the `opm index add` command to accept `registry+v2` bundles.

#### Direct Application

As an Operator Author, I want a way to directly apply a `registry+v2` bundle to a cluster -- without a `Subscription` -- so that I can:

- validate the bundle's content
- rapidly prototype my operator

_**Note:** This command is needed because an on-cluster representation of an operator must be generated from the bundle's metadata for OLM to manage it._

##### UX

Add a new command to `opm`, `opm bundle list`, which takes as input an Operator Bundle and outputs the set of kubectl-able manifests that OLM would apply to the cluster for that bundle.

```sh
$ opm bundle list . # detect the mediatype of the current directory and write the YAML stream of bundle manifests to stdout
$ opm bundle list image://quay.io/my/bundle:v1.0.0 > manifests.yaml # pull and unpack the bundle image, then write the YAML stream to a file
$ opm bundle list image://quay.io/my/bundle:v1.0.0 | yq 'map(select(.apiVersion == "apiextensions.k8s.io/v1" and .kind == "CustomResourceDefinition"))' - | kubectl apply - # pull and unpack the bundle image, then apply only the CRDs to a cluster

```

#### Validate Bundles

As a Pipeline Maintainer, I want a way to validate `registry+v2` bundles, so that I can ensure OLM is able to unpack and apply it before I add it to any release index.

##### UX

Add a new command to `opm`, `opm bundle validate`, which takes as input an Operator Bundle and fails with informative errors should the bundle be invalid.

```sh
$ opm bundle validate . # validate the bundle in the current directory, printing the results to stdout
$ opm bundle validate image://quay.io/my/bundle:v1.0.0 # pull, unpack, and validate the remote bundle image, printing the results to stdout
```

This command return value if validation fails.

### Risks and Mitigations

One risk is around backporting indexes with mixed mediatypes to older clusters. This needs to be supported to ensure upgrades and downgrades of OpenShift behave as intended. This is outlined further in the Upgrade/Downgrade section below.

Another risk is around operatorhub.io - it will need to be adjusted to support the `registry+v2` bundles. This work is not scoped as part of this enhancement and is done by a seperate team - how much effort is involved there is unclear.

Security impacts are minimal because the number of kubernetes objects that OLM supports is still the same - just the delivery mechanism (the bundle format) is changing.

## Design Details

### Implementation Details


#### Registry Database Schema Changes

In order to support adding `registry+v2` bundles to a registry database, the `operatorbundle` table schema must be updated to enable more than one mediatype. The table will be extended with a new `mediatype` column with a value that can be one of `{registry+v1, registry+v2}`. A respective migration will also be added to [operator-registy](https://github.com/operator-framework/operator-registry/tree/master/pkg/sqlite/migrations) to cover this schema change.

#### Registry API Changes

The `Bundle` gRPC message protobuf will be extended with a new field that will expose the a bundle's mediatype, which will inform client-side unpacking strategy:

```protobuf
message Bundle{
    // <existing fields>
    string mediatype = 12;
}
```

All request messages will also be extended with an `accepts` field, which clients can use to inform the server of the mediatypes they are capable of using:

```protobuf
message ListPackageRequest{
    repeated string accepts = 1
}

message ListBundlesRequest{
    repeated string accepts = 1
}

message GetPackageRequest{
	string name = 1;
    repeated string accepts = 2
}

message GetBundleRequest{
	string pkgName = 1;
	string channelName = 2;
	string csvName = 3;
    repeated string accepts = 4
}

message GetBundleInChannelRequest{
	string pkgName = 1;
	string channelName = 2;
    repeated string accepts = 3
}

message GetAllReplacementsRequest{
	string csvName = 1;
    repeated string accepts = 2
}

message GetReplacementRequest{
	string csvName = 1;
	string pkgName = 2;
	string channelName = 3;
    repeated string accepts = 4
}

message GetAllProvidersRequest{
	string group = 1;
	string version = 2;
	string kind = 3;
	string plural = 4;
    repeated string accepts = 5
}

message GetLatestProvidersRequest{
	string group = 1;
	string version = 2;
	string kind = 3;
	string plural = 4;
    repeated string accepts = 5
}

message GetDefaultProviderRequest{
	string group = 1;
	string version = 2;
	string kind = 3;
	string plural = 4;
    repeated string accepts = 5
}
```

_**Note:** An [enum]( https://developers.google.com/protocol-buffers/docs/proto3#enum) could be used in place of a string for the `mediatype` and `accepts` fields for tighter API-level constraints_

The server will remove all response elements that reference bundles with mediatypes not present in the `accepts` field of the request, so clients are never given bundles with mediatypes they don't understand.

If no `accepts` field is present on a request message, then the server will assume that only `registry+v1` bundles are accepted by the client. This allows for compatibility with older clients unaware of the `accepts` field.

#### Index Add Changes

When a `registry+v2` bundle is added to an index, `opm` will synthesize and add a CSV to the respective entry in the `operatorbundle` table. A lot of the existing `opm` and `operator-registry` code assumes bundles always contain CSVs, which make synthesis the simplest path to supporting `registry+v2` bundles.

#### Installtime Bundle Unpacking

At install, a CSV must be generated for a `registry+v2` bundle so that OLM can manage the operator it represents on the cluster. It must also take care not to apply any bundle resources that are usually generated by a CSV (e.g. APIService and Service). This means that when OLM resolves and unpacks a bundle, it needs to recognize its mediatype and take appropriate action.

#### Bundle Validation

The [operator-framework API module](https://github.com/operator-framework/api) will be extended with validators to support `registry+v2` [file format](#file-format) validation in advance of the implementation of the `opm bundle validate` command.


#### Legacy Support for AppRegistry

A bundle of type `registry+v2` can be stored in an AppRegistry type catalog after synthesizing a CSV and removing resources that are usually generated by CSVs. This allows clusters with AppRegistry type catalogs to receive updates to operators whose authors have switched to publishing `registry+v2` bundles.

### Test Plan

There will be e2e tests alongside unit tests. The e2e test will verify that the functionality is delivered as expected - bundles consisting just of kubernetes manifests are installed and managed on cluster by OLM. Potentially existing tests can be reused but with the newer bundle format.

Outside of core OLM there will also be tests in operator-registry to ensure the new bundle format is consistently generated as we expect. Its also critical to test how indexes composed of registry+v2 bundles will behave.

### Drawbacks

- `registry+v2` bundles depart from the established CSV interface OLM has always required and users are familiar with
- In `registry+v1` bundles, all installation details are denormalized in the CSV -- install strategy, provided APIs, etc. -- allowing Operator Authors to publish less manifests and describe their operator as a transaction
