---
title: audit-command-operation
authors:
  - "@camilamacedo86"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2021-02-22
last-updated: 2021-04-21
status: implementable
see-also:
---

# Audit command operation

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions 

1. Should this command be added into SDK or OPM or in a new tool?

Beside the [catalog builds in operator-sdk](operator-sdk-builds-catalogs.md) proposal, SDK maintainers (@jesus, @estroz) point out that they think that it would better fit in OPM and not in SDK and that while the SDK has some catalog abilities in that EP see that it has been using OPM binary as dependency to achieve it and the [non-goal](https://github.com/operator-framework/enhancements/blob/master/enhancements/operator-sdk-builds-catalogs.md#non-goals).
 
However, this proposal aims to discuss the audit command and how and what it will do and not exactly where it will live. This decision is out of this proposal's scope since it brings strategy decisions. In this way, let's keep it open until we are able to properly defined all that should be done by audit and how to have the resources required to address and implement this proposal. It probably will be done first as POC in its own repository.

2. Could not the audit command be only a shell script? 

Golang provides facilities for us to achieve the required goals describe in this EP and provides it with good maintainability as required since audit command has a potential to grow. For example, one of the requirements is to provide a spreadsheet that can be done and maintained by using [excelize](https://github.com/360EntSecGroup-Skylar/excelize), which is a lower required effort compared to manipulated tabulate files. It also can be improved and get very fancy for future versions.  

In this way, shell script shows not bring any advantage or help us to reduce the effort required to provide its first version. Also, complex shell scripts have lower maintainability, which concludes that it is not the right choice for this proposal and an analytic solution.

3. How the proposal [New OpenShift Validator][ocp-validator-proposal] affects the audit?

See that in this proposal, we are providing some alternatives to implement the specific check. However, to address specific downstream concerns and checks are out of the scope of this proposal. To address specific downstream concerns and checks are out of the scope of this proposal. 

## Summary

This proposal describes some possibilities of a new operation provided via an **alpha/POC solution which is not committed GA functionality** with the purpose of auditing a specific OLM index catalog and then output a report. The audit command is an analytic operation that levarage on Operator Framework  solutions. Its purpose is to obtain and report and aggregate data provided by checks and analyses done in the bundle and index catalog level to address different requirements for distinguished personas via its customized options of usage.

## Motivation

The audit command that could be used by , for example; the PE team, external users, and internal product teams as many other personas to compare their operators against various audit criteria. 

The audit tool should be used to examine a given catalog's set of operators and produce a report that is readable by human via the `xls` format but also be exported to `JSON` formats that can be processed and used by CI jobs for example.

### Goals

- Be able to audit OLM index catalog and output a report by gathering aspects of the operators bundles, packages and channels 
- Be able to extract a report with the audit results and in some formats such as json. 
- Be able to run specific custom checks for each bundle in order to audit the catalog
- Be able to perform validations and analyses in the bundle and index catalog and bundle level.

### Non-Goals

- Add new checks or validators in [operator-framework/api][operator-framework/api]. This proposal has the goal to define the command and operation only. The checks should be addressed and discussed in their respective proposals.
- Provide documentation that will describe how to add the custom scorecard tests in the bundle done with [Writing Custom Scorecard Tests][writing-custom-scorecard]  manually for Non-SDK projects or bundles which are not built with `make bundle` and how they are used with the audit command.
- Change the workflow or make any new behaviour as mandatory to build the operator bundles
- Any specific criteria per vendor proposed for example [New OpenShift Validator][ocp-validator-proposal].

**IMPORTANT**: Note that we are describing here an analytic tool/command that will start to be implemented in **alpha/POC**. This feature uses what is in place and implemented already and then, it does not make any change or requires any new behaviour for the operator authors.

## Proposal

Add a new command `audit` with the purpose to audit all desired operator bundle data of a specific index catalog. 

By default, the command output a report where it is possible to have a view of the main characteristics and aspect of all operator bundles shipped in a specified OLM index catalog as check if they are passing in criteria and checks required to ensure their quality. Another group of checks and analyses that ought to be addressed by audit command are validations which looks at all the bundles of a package holistically and attempts to detect potential error conditions. 

Following some desirable flags to illustrate its possibilities: 

```
$ audit bundles --help

Flags:
  --help                     help for audit
  --index-image              index image to get package from. 
  --container-tools          the container tool used to login in the registry and pull and extract the images. One of: [docker, podman] (default "docker")
  --wait-time duration       minutes to wait for the report to be complete for each bundle. Example: 20m (default timeout is 10m)
  --output                   output format for results. Valid values: xlsx, json (default "xlsx")
  --disable-scorecard bool   if set, disable the scorecard checks. Note that scorecard test requires a cluster running.
  --scorecard-config         path of the scorecard test directory which will be used as the default scorecard config which be added to the operator bundles. If none be informed the default audit configuration will be used instead
  --disable-validators bool    if set, disable the validators checks. 
  --validators-values          string map with key and values which will be informed to the validators e.g`"k8s=1.21"` 
  --checks-only                     If set, disable the data report. It is useful for who would like to audit the index only runnning the checks and validations. 
  --head-only                If set, audit only the operator bundles which are the head of the channel. (default all bundles of the index catalog image will be used)
  --error-only               If set, raise only the errors and disable the report of warning(s) found in the checks (default warning and errors will be reported)
  --deprecated               If set, the audit will also check the deprected operator bundles. By default they are ignored.
  --filter                   This flag will filter the data and will only provide the results where the operator bundle maches with the regex informed.
```

Note that its command(s) will:
- Use the index image to gathering the data
- Download the operator bundles
- Run checks and validations in the bundle level
- Run checks and validations in the catalog level
- Output the results

### User Stories 

#### As an admin and or maintainer of OLM index catalog, I would like to easily know more about the bundles in the catalogue so that, I can easily provide strategies which will be useful for the biggest part of the authors of the projects published on it.

To address this need audit command will use the data from the bundle manifests and operator bundle repository. This proposal detailed described each field and how to process each required information.

Note that the flag `--output=xls` also was added to address this scenario which indeed allow we do things very fancy with which is what these personas usually does with spreadsheets. for further information see the [Report result columns][#report-result-columns]. 

#### As an admin and or maintainer of OLM index catalog, I would like to check if all bundle operators in the catalogue are respecting all expected criteria so that, I can ensure a better quality of all bundles in the catalogue.

To address this need, we will use [api](https://github.com/operator-framework/api) and run the operator bundles against the validators.

#### As an admin and or maintainer of OLM index catalog, I'd like to identify problems and concerns in the operators bundles of some catalogue so that, I can easily decide if they should or not be removed from the index catalog.

To address this need, we will use [SDK](https://github.com/operator-framework/operator-sdk) command `scorecard bundle` and run it against each bundle extracted from the index image.

**NOTE** To address additional needs and keep the simplicity as reduce the effort required audit command will provide the same SDK bundle validation available options to let users list and select the validators. 

#### As an admin and or maintainer of OLM index catalog I would like to audit the OLM index catalog by running custom tests for each operator bundle that, I can use it to verify if all operator bundles will work as expected.

By using the option `--scorecard-config` audit users are able to provide their own custom tests that should be used to check each operator bundle of a catalog.

#### As an admin and or maintainer of OLM index catalog I would like to audit the OLM index catalog by considering the tests provided by the operator author in each operator bundle that, I can use the operator author tests to ensure the operator bundles of a catalog for each specific scenario and use case.

Operators authors are able to write custom tests using [SDK scorecard][sdk-scorecard] and push them in the operator bundle. Note that any custom tests added to an SDK project by following the documentation [Writing Custom Scorecard Tests][writing-custom-scorecard] will be also added to the bundle automatically when the operator author uses the Makefile target [make bundle][sdk-make-bundle]. See in this example in the community operators [here][community-operators-with-scorecard] the default tests generated by SDK are also published in.

**NOTE** To allow operator authors provided the same option for Non-SDK projects would only be required describe the steps do add them into the operator bundle manually. 

#### As an admin and or maintainer of OLM index catalog I would like to audit the OLM index catalog by ignoring the tests provided by the operator author in each operator bundle that, I can use the operator author tests to ensure the operator bundles of a catalog for each specific scenario and use case.

Users are able to use the flag `--scorecard-custom-tests` to disable the ability to use all custom tests provided by the author. 

#### As an admin and or maintainer of OLM index catalog, I'd like to check the operator bundles against the criteria for a specific cluster version (Kubernetes or OCP) so that, I can easily decide what operator bundles should be or not in the index catalog which I intend to use with some specific cluster version.

The audit flag `--validator-values` can allow we pass this value to the validators. (e.g `--validator-values="k8s-version=1.22"`)

For further information also check the proposal [operator-framework/api: add new checks to the operatorhub validator][ep-new-checks-for-operator-hub].

#### As an admin and or maintainer of OLM index catalog, I would like to use audit to only perform the checks and get its result on JSON format so that, I can use it in CI job.

To address this requirement user are able to use the flags `--output=JSON` and the `--checks-only`.

Also, see that in this case user might only get the errors and not the warnings the flag `--errors-only` can also be used to address this need.

#### As an admin and or maintainer of OLM index catalog, I'd like to be able to disable the validations and/or checks done for the operator bundles so that, I can easily decide what check is more appropriate for my scenario.

To address this need users will be able to use the specific flags.

#### As an admin and or maintainer of OLM index catalog, I would like to use audit to only audit the head bundle operators so that, I can have a faster result.

To address this requirement user are able to use the flag `--head-only`.

#### As an admin and or maintainer of OLM index catalog, I would like to define what is the tool which will be used to pull and save the images so that, I can define it accordingly with the tool which I have installed and has my credentials when required.

To address this requirement user are able to use the flag `--container-tools` and inform the option as please them.

#### As an admin and or maintainer of OLM index catalog, I'd like to define a timeout so that, I can customize it according to my expectations and avoid a process get stuck or increase the default time when I know that it will require more time to be processed.

To address this requirement user are able to use the flag `--wait-time` and inform a time in minutes which will be used instead of the default timeout.

#### As an admin and or maintainer of OLM index catalog, I'd like to use the audit tool only to check operator bundles for a specific package, so that I can have the report result for a specific project

To address this requirement a user will be able to use the flag `--filter` where the tool will only get the operator bundles, channels or packages where the name ideally matches with a regex informed. Initially the usage of an SQL Like option such as `%name%` for the alpha version can also be acceptable.  

#### As an admin and or maintainer of OLM index catalog, I'd like to look at all the bundles of a package holistically and attempts to detect potential error conditions so that, I can easily identify errors in the upgrade graph paths.

We can create roles here such as: get all bundles that are using replace for the package and check if the bundle set in the `replace` bundle exists. 

### Implementation Details/Notes/Constraints 

Following the actions which are being proposed in this document:

- Extract the database from the image informed
- Get all bundles name and paths from the index catalog image
- Download and extract all bundles files by using the operator bundle path
- Get the required data for the report from the index, from the operator bundle manifest files and from the operator repository
- Use the operator-framework/api to validate the bundles
- Use sdk tool to execute the scorecard bundle checks
- Perform the catalog-level checks and analyses
- Output a report providing the information obtained and processed. 

#### Using the index image to gathering the data

Following an example to extract the sql database from the image currently: 

```sh
docker create --name rh-catalog registry.redhat.io/redhat/redhat-operator-index:v4.6 "yes"
docker cp rh-catalog:/database/index.db .
```

After that, the `index.db` file can be used by the tool which can gathering the required information via sql. 

To know how to use SQL to read the index.db file check [here](https://github.com/jmccormick2001/indexdump/blob/main/pullrepos.go#L74) a code example.

The above steps are base on the current index catalog format as a database. It might be changed for JSON format which should be used instead. The audit command just requires to support the latest index catalog format which in this case will be JSON. More info [Package representation and management in an index](https://github.com/operator-framework/enhancements/blob/master/enhancements/declarative-index-config.md).

#### Downloading and extracting the operator bundles

The OLM index image does not store the operator bundle manifests. The operator bundle registry address can be found in the `bundlepath` entry from the`operatorbundle` table. 

Also, see that deprecated bundles requires to be ignored by default. If the user would like to indeed check these ones then, the flag `--deprecated` requires to be set. In this way, ensure that in the query above we are filter by that as well. For further info see the [Deprecating Bundles](enhancements/deprecated-bundles.md).

We need to use `docker` or `podman` tools to pull and extract the operator bundle files. Note that the audit command let the user choose what container tool should be used via flag `--container-tool`. 

Following an example with the manually steps to download an extract the bundle operator manifests only to let you know how to check and test it locally:

```
$ docker pull <bundlepath>
$ docker save <bundle image> > mybundle.tar
$ tar -xvf mybundle.tar 
$ tree
.
├── 696782e86a62476638704177b71dd37382864a1801866002cd6628d1a3eec4c0
│   ├── VERSION
│   ├── json
│   └── layer.tar
├── f6bbe84c78f5d0725d248207f92a23edfcbe66d3016f841018c03f933341dae3.json
├── manifest.json
└── mybundle.tar
$ 
``` 

Now, see that all manifests are in the `layer.tar`:

```
$ tar -xvf 696782e86a62476638704177b71dd37382864a1801866002cd6628d1a3eec4c0/layer.tar
$ tree
.
├── 696782e86a62476638704177b71dd37382864a1801866002cd6628d1a3eec4c0
│   ├── VERSION
│   ├── json
│   └── layer.tar
├── f6bbe84c78f5d0725d248207f92a23edfcbe66d3016f841018c03f933341dae3.json
├── manifest.json
├── manifests
│   ├── myopertor.v0.5.5.clusterserviceversion.yaml
│   ├── apps_v1alpha1_apimanager_crd.yaml
│   ├── capabilities_v1alpha1_api_crd.yaml
│   ├── capabilities_v1alpha1_binding_crd.yaml
│   ├── capabilities_v1alpha1_limit_crd.yaml
│   ├── capabilities_v1alpha1_mappingrule_crd.yaml
│   ├── capabilities_v1alpha1_metric_crd.yaml
│   ├── capabilities_v1alpha1_plan_crd.yaml
│   └── capabilities_v1alpha1_tenant_crd.yaml
├── metadata
│   └── annotations.yaml
├── mybundle.tar
└── root
    └── buildinfo
        ├── Dockerfile-myoperator-rhel7-operator-metadata-2.8.2-5
        └── content_manifests
            └── myopertor-bundle-container-2.8.2-5.json
```

#### Using SDK scorecard bundle

For the SDK tool be able to run scorecard tests its configuration requires to be wrote in the bundle directory (e.g `tests/scorecard/config.yaml` see [here](https://github.com/operator-framework/operator-sdk/blob/master/testdata/go/v3/memcached-operator/bundle/tests/scorecard/config.yaml)). Operator bundles which are build with SDK will have the scorecard tests configured by default. (e.g see [here](https://github.com/operator-framework/community-operators/tree/master/community-operators/namespace-configuration-operator/1.0.1)).
 
Note that the bundles might have some specific tests implemented by them using the option to write custom tests. For further information check the [Writing Custom Scorecard Tests](https://sdk.operatorframework.io/docs/advanced-topics/scorecard/custom-tests/) documentation. In this way, if a bundle does not have the default scorecard test configured audit command will add it. Otherwise, the audit command by default will use what was provided instead. 

Keep in mind that the location of the scorecard tests can be changed and then, we can verify that by looking in the bundle index annotations `operators.operatorframework.io.test.config.v1`. See [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/metadata/annotations.yaml#L11). 

In this way, the audit command will:
- Check if the operator bundle has the manifest `metadata/annotations.yaml` with the annotation `operators.operatorframework.io.test.config.v1: tests/scorecard/` (e.g see the [bundle](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/metadata/annotations.yaml#L11))

```yaml
annotations:
  ...
  operators.operatorframework.io.test.config.v1: tests/scorecard/
```
- Ensure that at least 1 manifest of the kind `Configuration` is present in the path informed on the bundle. (e.g see the `Configuration` for the bundle [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/metadata/annotations.yaml#L11https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/metadata/annotations.yaml#L11) which is in `tests/scorecard/`).
- If not, then the same command will write on the operator bundle the same manifest scaffold by the sdk tool [example](https://github.com/operator-framework/community-operators/tree/master/community-operators/namespace-configuration-operator/1.0.1)

Now see how to use [SDK](https://github.com/operator-framework/operator-sdk) tool to check the bundles with scorecard:

```
 $ operator-sdk scorecard bundle --wait-time=120 --output=json
{
  "kind": "TestList",
  "apiVersion": "scorecard.operatorframework.io/v1alpha3",
  "items": [
    {
      "kind": "Test",
      "apiVersion": "scorecard.operatorframework.io/v1alpha3",
      "spec": {
        "image": "quay.io/operator-framework/scorecard-test:v1.4.0",
        "entrypoint": [
          "scorecard-test",
          "olm-spec-descriptors"
        ],
        "labels": {
          "suite": "olm",
          "test": "olm-spec-descriptors-test"
        }
      },
      "status": {
        "results": [
          {
            "name": "olm-spec-descriptors",
            "log": "Loaded ClusterServiceVersion: memcached-operator.v0.0.1\nLoaded 1 Custom Resources from alm-examples\n",
            "state": "fail",
            "errors": [
              "size does not have a spec descriptor"
            ],
            "suggestions": [
              "Add a spec descriptor for size"
            ]
          }
        ]
      }
    },
    {
      "kind": "Test",
      "apiVersion": "scorecard.operatorframework.io/v1alpha3",
      "spec": {
        "image": "quay.io/operator-framework/scorecard-test:v1.4.0",
        "entrypoint": [
          "scorecard-test",
          "olm-crds-have-resources"
        ],
        "labels": {
          "suite": "olm",
          "test": "olm-crds-have-resources-test"
        }
      },
      "status": {
        "results": [
          {
            "name": "olm-crds-have-resources",
            "log": "Loaded ClusterServiceVersion: memcached-operator.v0.0.1\n",
            "state": "fail",
            "errors": [
              "Owned CRDs do not have resources specified"
            ]
          }
        ]
      }
    },
    {
      "kind": "Test",
      "apiVersion": "scorecard.operatorframework.io/v1alpha3",
      "spec": {
        "image": "quay.io/operator-framework/scorecard-test:v1.4.0",
        "entrypoint": [
          "scorecard-test",
          "olm-crds-have-validation"
        ],
        "labels": {
          "suite": "olm",
          "test": "olm-crds-have-validation-test"
        }
      },
      "status": {
        "results": [
          {
            "name": "olm-crds-have-validation",
            "log": "Loaded 1 Custom ...,},}]\n",
            "state": "pass"
          }
        ]
      }
    },
    {
      "kind": "Test",
      "apiVersion": "scorecard.operatorframework.io/v1alpha3",
      "spec": {
        "image": "quay.io/operator-framework/scorecard-test:v1.4.0",
        "entrypoint": [
          "scorecard-test",
          "olm-bundle-validation"
        ],
        "labels": {
          "suite": "olm",
          "test": "olm-bundle-validation-test"
        }
      },
      "status": {
        "results": [
          {
            "name": "olm-bundle-validation",
            "log": "time=\"2021-02-23T19:32:41Z\" level=debug msg=\"Found manifests directory\" name=bundle-test\ntime=\"2021-02-23T19:32:41Z\" level=debug msg=\"Found metadata directory\" name=bundle-test\ntime=\"2021-02-23T19:32:41Z\" level=debug msg=\"Getting mediaType info from manifests directory\" name=bundle-test\ntime=\"2021-02-23T19:32:41Z\" level=info msg=\"Found annotations file\" name=bundle-test\ntime=\"2021-02-23T19:32:41Z\" level=info msg=\"Could not find optional dependencies file\" name=bundle-test\n",
            "state": "pass"
          }
        ]
      }
    },
    {
      "kind": "Test",
      "apiVersion": "scorecard.operatorframework.io/v1alpha3",
      "spec": {
        "image": "quay.io/operator-framework/scorecard-test:v1.4.0",
        "entrypoint": [
          "scorecard-test",
          "olm-status-descriptors"
        ],
        "labels": {
          "suite": "olm",
          "test": "olm-status-descriptors-test"
        }
      },
      "status": {
        "results": [
          {
            "name": "olm-status-descriptors",
            "log": "Loaded ClusterServiceVersion: memcached-operator.v0.0.1\nLoaded 1 Custom Resources from alm-examples\n",
            "state": "fail",
            "errors": [
              "memcacheds.cache.example.com does not have a status descriptor"
            ]
          }
        ]
      }
    },
    {
      "kind": "Test",
      "apiVersion": "scorecard.operatorframework.io/v1alpha3",
      "spec": {
        "image": "quay.io/operator-framework/scorecard-test:v1.4.0",
        "entrypoint": [
          "scorecard-test",
          "basic-check-spec"
        ],
        "labels": {
          "suite": "basic",
          "test": "basic-check-spec-test"
        }
      },
      "status": {
        "results": [
          {
            "name": "basic-check-spec",
            "state": "pass"
          }
        ]
      }
    }
  ]
}

```

#### Scorecard config manifests and options

This proposal describes the audit command adding the same config manifests added by default for all projects built with SDK ONLY when they are NOT provided as described above. However, the behaviour of the command operation in the audit command can be changed via the flag options: 

```
  --scorecard-config         path of the scorecard test directory which will be used as the default scorecard config which be added to the operator bundles. If none be informed the default audit configuration will be used instead
  --scorecard-custom-tests    set false to disable the scorecard checks use the tests which are configured in the bundle. If this option be choosen audit tool will always set its default tests. (default true)
```

So, if the audit user provide `scorecard-config` then this value should be inject for each bundle unless it has  scorecard defined already. However, note that it can also be used combined with `--scorecard-custom-tests=false` which will indeed replace any scorecard test which might exists in the operator bundle already by the directory provided. 

#### To generate the report result

The idea propose here is to use [excelize](https://github.com/360EntSecGroup-Skylar/excelize) lib for go which has an excellent adoption in the community. By using this lib, shows low effort write the spreadsheet and easy to keep it maintained. For further information check its [docs](https://pkg.go.dev/github.com/360EntSecGroup-Skylar/excelize/v2#readme-excelize) which has a lot of code examples.

#### Description of possible columns results

Following a description of the possible results output in the reports of this tool to describe how the data can be obtained.   

| Column | Description | 
| ------ | ----- | 
| Operator Name |  This value can be found in the `package` table in the index catalog image. (e.g `select name from package`)
|  default channel | This value can be found in the `package` table in the index catalog image. (e.g. `select default_channel from package`) 
| Operator Bundle Channel | This value can be found in the `channel` table in the index catalog image. (e.g `select channel.name from channel`)
| Operator Bundle Name | This value can be found in the `operatorbundle` table in the index catalog image. (e.g`select operatorbundle.name from operatorbundle`)
| Operator Bundle Version | This value can be found in the `operatorbundle` table in the index catalog image. (e.g`select version from operatorbundle`)
| Links | This value can be found in the CSV `spec.links`. (e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/manifests/namespace-configuration-operator.clusterserviceversion.yaml#L284-L288))
| Maintainers | This value can be found in the CSV `spec.maintainers`. (e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/manifests/namespace-configuration-operator.clusterserviceversion.yaml#L289-L291))
| Provider | This value can be found in the CSV `spec.provider.name`. (e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/manifests/namespace-configuration-operator.clusterserviceversion.yaml#L294))
| Maturity | This value can be found in the CSV `spec.maturity`. (e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/manifests/namespace-configuration-operator.clusterserviceversion.yaml#L292))
| Capabilities | This value can be found in the CSV `annotations` with the key `capabilities`. (e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/manifests/namespace-configuration-operator.clusterserviceversion.yaml#L76))
| Categories | This value can be found in the CSV `annotations` with the key `categories`. (e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/manifests/namespace-configuration-operator.clusterserviceversion.yaml#L77))
| Certified | This value can be found in the CSV `annotations` with the key `certified`. (e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/manifests/namespace-configuration-operator.clusterserviceversion.yaml#L78)). **Note** with the results are in the `xls` format the command ought to replace all `true` and  `false` values by `YES` or `NO`. 
| CreatedAt | This value can be found in the CSV `annotations` with the key `createdAt`. (e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/manifests/namespace-configuration-operator.clusterserviceversion.yaml#L80))
| Description | This value can be found in the CSV `annotations` with the key `description`. (e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/manifests/namespace-configuration-operator.clusterserviceversion.yaml#L80))
 | Repository | This value can be found in the CSV `annotations` with the key `repository`. (e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/manifests/namespace-configuration-operator.clusterserviceversion.yaml#L84) )
 | **(optional)** Builder | This value can be found in the CSV `annotations` with the key `operators.operatorframework.io/builder` (e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/manifests/namespace-configuration-operator.clusterserviceversion.yaml#L82)). However, if this annotation does not exist in the CSV or in the operator index image then, the tool will use the repository info to try to obtain it. See the section [Getting the data from the annotations and repository](#getting-the-data-from-the-annotations-and-repository). Also, its current options are `[SDK, KUBEBUILDER]`)
 | **(optional)** SDK version | This value can be found in the CSV `annotations` with the key `operators.operatorframework.io/builder` (e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/manifests/namespace-configuration-operator.clusterserviceversion.yaml#L82)). However, if this annotation does not exist in the CSV or in the operator index image then, the tool will use the repository info to try to obtain it. See the section [Getting the data from the annotations and repository](#getting-the-data-from-the-annotations-and-repository).
 | **(optional)** Project Layout | This value can be found in the CSV `annotations` with the key `operators.operatorframework.io/project_layout` (e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/manifests/namespace-configuration-operator.clusterserviceversion.yaml#L83)). However, if this annotation does not exist in the CSV or in the operator index image then, the tool will use the repository info to try to obtain it. See the section [Getting the data from the annotations and repository](#getting-the-data-from-the-annotations-and-repository). Note that this field let us know the version of the plugins used to build the project and it is only possible be obtained if the project was built with SDK or Kubebuilder tool.
 | **(optional)** Language | This value can be found in the CSV `annotations` with the key `operators.operatorframework.io/project_layout` (e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/manifests/namespace-configuration-operator.clusterserviceversion.yaml#L83)). However, if this annotation does not exist in the CSV or in the operator index image then, the tool will use the repository info to try to obtain it. See the section [Getting the data from the annotations and repository](#getting-the-data-from-the-annotations-and-repository). Also, its current options are `[golang, ansible, helm]`)
| **(optional)** Golang Version | See the section [Getting the data from the annotations and repository](#getting-the-data-from-the-annotations-and-repository) to know how this information can be obtained. 
| Multiple Architectures | To obtain this info we will need to looking for the CVS  labels `operatorframework.io/os.<value>` and `operatorframework.io/arch.<value>`. Then, we will add in the column the list of the values found such as `os.windows` and `arch.s390x`. **NOTE** List values should be broken in lines for the `xls` output format.
| validations error(s) | List of error(s) found by check the bundles against the validators implemented in the API.
| validations warning(s) | List of warning(s) found by check the bundles against the validators implemented in the API.
| scorecard failures | List of tests which fails with the error(s) found by running `operator-sdk scorecard bundle`

#### Getting the data from the annotations and repository

##### By looking the annotations

Check in the CSV `annotations` by the key `operators.operatorframework.io/builder` (e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/manifests/namespace-configuration-operator.clusterserviceversion.yaml#L82)). If the value is not present in the CSV annotations look for it in the `metadata/annotations.yaml` file in the operator bundle manifests. (e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/metadata/annotations.yaml#L8)). Now see how the audit command will use the values for its result if the value found is, for example, `operator-sdk-v1.3.0`:

| Column | Description | 
| ------ | ----- | 
| Builder | `SDK`|
| SDK Version | `v1.3.0`|

Now let's check in the CSV `annotations` for the key `operators.operatorframework.io/project_layout` (e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/manifests/namespace-configuration-operator.clusterserviceversion.yaml#L83)). At the same way, if the value is not present in the CSV annotations look for it in the `metadata/annotations.yaml` file in the operator bundle manifests. ( e.g see [here](https://github.com/operator-framework/community-operators/blob/master/community-operators/namespace-configuration-operator/1.0.1/metadata/annotations.yaml#L10)). Now see how the audit command will use the values for its result:

| Column | Description | 
| ------ | ----- | 
| Builder | `SDK` (if not found with the annotation described above)|
| Project Layout | value of `operators.operatorframework.io/project_layout` |
| Language | It will be `golang`, `ansible` or `helm`. Use the value of `operators.operatorframework.io/project_layout` to get this information. For example, if the annotation value is `ansible.sdk.operatorframework.io/vX` then, the language is ansible.|

##### (OPTIONAL) By looking into the repository 

**It is desirable but is not a mandatory requirement for the alpha version.** 

If the CSV and the index operator image has not the metric annotations then, we will need to check the info by verifying the files into the operator project repository.
 
**NOTE** To know how it can be implemented check [here](https://github.com/jmccormick2001/indexdump/blob/main/gethttp.go) a code example of a similar implementation.

**Gathering the info from PROJECT file**

We can check if the Builder is `Kubebuilder` or `SDK` and its other aspects by looking into the `PROJECT` file see:

- If the attribute `layout` of the `PROJECT` file be `ansible.sdk.operatorframework.io/vX` (see e.g [here](https://github.com/operator-framework/operator-sdk/blob/v1.4.2/testdata/ansible/memcached-operator/PROJECT#L2)):

| Column | Description | 
| ------ | ----- | 
| Builder | `SDK` |
| Language | `ansible` |
| Project Layout | `ansible.sdk.operatorframework.io/vX` |
| SDK Version | Check in the Dockerfile. (e.g see [here](https://github.com/operator-framework/operator-sdk/blob/v1.4.2/testdata/ansible/memcached-operator/Dockerfile#L1) where it will be `v1.4.2`). |

- If the attribute `layout` of the `PROJECT file be `helm.sdk.operatorframework.io/vX` (see e.g [here](https://github.com/operator-framework/operator-sdk/blob/v1.4.2/testdata/helm/memcached-operator/PROJECT#L2)):

| Column | Description | 
| ------ | ----- | 
| Builder | `SDK` |
| Language | `helm` |
| Project Layout | `helm.sdk.operatorframework.io/vX` |
| SDK Version | Check in the Dockerfile. (e.g see [here](https://github.com/operator-framework/operator-sdk/blob/v1.4.2/testdata/helm/memcached-operator/Dockerfile#L1) where it will be `v1.4.2`) |

- If the attribute `layout of the`PROJECT` file be `go.kubebuilder.io/vX` (see e.g [here](https://github.com/operator-framework/operator-sdk/blob/v1.4.2/testdata/go/v3/memcached-operator/PROJECT#L2)):

| Column | Description | 
| ------ | ----- | 
| Builder | Check if it has plugins with `sdk.operatorframework.io/` (e.g see [here](https://github.com/operator-framework/operator-sdk/blob/v1.4.2/testdata/go/v3/memcached-operator/PROJECT#L12-L14)) to set as `SDK` otherwise it will be `Kubebuilder`|
| Language | `golang` |
| Project Layout | `go.kubebuilder.io/vX` |

However, note that for the previous versions of projects built with `Kubebuilder` we will not found the layout information in the `PROJECT` file as well. (e.g see [here](https://github.com/kubernetes-sigs/kubebuilder/blob/v2.3.2/testdata/project-v2/PROJECT)) then:

| Column | Description | 
| ------ | ----- | 
| Builder | `Kubebuilder`|
| Language | `golang` |
| Project Layout | If the attribute `version` in the `PROJECT` file be `2` it will be `go.kubebuilder.io/v2` otherwise, it will be `go.kubebuilder.io/v1` |

**Gathering the info from go.mod file**

Now, if the repository has not a `PROJECT` file then, it means that the project could also be built with an SDK version < 1.0 or any other tool which is not SDK or Kubebuilder. In this way, check if the project has a `go.mod` file with the sdk dependency `github.com/operator-framework/operator-sdk`: 

```go
go 1.13

require (
	github.com/operator-framework/operator-sdk v0.17.0
```

Following how we would set the values for the above example. 

| Column | Description | 
| ------ | ----- | 
| Builder | `SDK`|
| Language | `golang` |
| SDK Version | `v0.17.0` |
| Golang Version | `1.13` |

See [here](https://github.com/jmccormick2001/indexdump/blob/main/gethttp.go#L69) a code example of this implementation done in a POC created as a base for the audit command. Check that the POC is getting the version properly by looking at its result as an example [here](https://github.com/jmccormick2001/indexdump/blob/main/report.txt.11-11-2020#L13).

**NOTE** If the `go.mod` was found but not the SDK dependency we will set the `Language` as `golang`. Also, we will always check the `go.mod` file for Golang projects to know the Golang version used.  

**Gathering the info from build/Dockerfile**

You will only need to check the `build/Dockerfile` when the project is not writen in Go. In this way, when the `go.mod` file cannot to be found then, it might to be an operator project using Helm or Ansible which was built with an SDK version < 1.0. In this case, we will be looking for `quay.io/operator-framework/ansible-operator` or `quay.io/operator-framework/helm-operator` images. Let's check an Ansible Operator. See the [Dockerfile](https://github.com/operator-framework/operator-sdk-samples/blob/v0.17.0/ansible/memcached-operator/build/Dockerfile): 

```
FROM quay.io/operator-framework/ansible-operator:v0.17.0
```

| Column | Description | 
| ------ | ----- | 
| Builder | `SDK`|
| Language | `ansible` |
| SDK Version | `v0.17.0` |

Now, see a [Dockerfile](https://github.com/operator-framework/operator-sdk-samples/blob/v0.17.0/helm/memcached-operator/build/Dockerfile) for a Helm operator project built with SDK version < 1.0. Then, note that the same fields will be obtained:

| Column | Description | 
| ------ | ----- | 
| Builder | `SDK`|
| Language | `helm` |
| SDK Version | `v0.17.0` |

**NOTE** The Dockerfile is a text file that can be parsable as any string. The audit command only requires to check by the line with `FROM quay.io/operator-framework/` and get the value after the `:` which in this example is `v0.17.0`. See [here](https://github.com/jmccormick2001/indexdump/blob/main/gethttp.go#L107) this implementation is done for the POC used as a base for this proposal.

### Auditing just the latest channel heads 

This option allows the user audit only the latest bundle versions (default channel heads) for each package. It can be enabled by setting `--head-only` boolean flag.

By using this flag, the audit command will filter the operator bundles in the first step to download and extract just the head ones. 

To know the operator bundles which are the latest head we will get only the entries which has values in the `operatorbundle` table for both `csv` and `bundle` column. Then, we also need to ensure using semver that it is the latest/upper version of the channel. For further information check the [Update Graph Generation in the opm docs](https://github.com/operator-framework/operator-registry/blob/master/docs/design/opm-tooling.md#update-graph-generation). 

#### Disabling the data to perform only the checks

By using the option `--checks-only`, the audit command will only be gathering the operator bundle names and run the checks. All other columns described above will be skipped.

This option is useful for who would like to add the audit in an automated CI job for example to check the catalog.  
 
#### Disabling the warning(s)

By using the option `--error-only`, the audit command will only raise the errors  found.

This option is useful for who would like to add the audit in an automated CI job for example to check the catalog combined with the flag `--checks-only`.  

#### Allowing re-start the process from the latest point (Optional for alpha)

In the case of command failures, the user needs to re-start the process from where it stops. This feature's motivation is to users does not need to re-start a job from the beginner, which we have an educated guess that it will take on average 2 hours to be finished, when/if an issue is faced. 

Then, the command will always keep the info processed as its state wrote on disk. Then, the command can ask the user the job should re-start from where it stops. However, keep in mind that all interaction needs to be allowed via command line and useful to be executed via jobs.

To know that the command finished successfully is expected the exit(0). See that a CI job can re-start a job from where it stops without the need to be re-started. It is achievable by retrying X times, for example, in a shell.

**NOTE** It does not show a high effort to be implemented. However, it is not a mandatory feature to be addressed in the alpha stage. Ditto so, only keep in mind this goal for the future implementations of the audit command to not take code/design decisions that might make it harder to get done in the future.

#### Informing the command progress

It is a command that usually can take a while to be processed. Then, users require to be informed about its progress.

Since it is an initial/alpha stage would be acceptable the tool does not work asynchronous and just output logs that let the user follow up its process, which is a behaviour that can be improved in the future.

A friendly and easy option to be implemented would also add a counter with a progress bar for the command line. However, we would also need to add more one flag option to output the logs for the CI jobs and save them in the log file when this option is not used to allow troubleshooting.

#### Handle failures scenarios

The command tool needs to be implement to not stop the process in case of failures such as:

**Unable to find the operator bundle repository** 
 
If the audit command cannot find an operator repository, http code error 404, then the command can retry once but after that should skip the task and move forward.

For this specific case, the command should set in the repository column of its result something such as "unable to find (http error code)". Also, the tool should able to log the data required to troubleshooting these scenarios.

**Facing error(s) when try to run sdk commands** 

If the tool face an error such as a timeout issue to run scorecard for any operator bundle then, it should log the error and continue the process. 

#### Timeout option

Note that the tool will have a default timeout set up for the command and then, it can be changed by the flag `--wait-time for who is using it and has a better idea over what would be the expected reasonable time for the specific scenario.
          
The proposal has no intention to address a complexity logic to try to guess a default timeout based on the number of bundles and other possible combination aspects. Beside be possible to try to calculate that based on some logic such as `wait time * bundles in catalog * number of tests` it would bring complexities which show not required at least in the first moment since users can indeed set any value as please them. See that, it can change a lot since the user might be running the full report or not, might be using the checks or not, etc. So, to keep the simplicity the approach adopted is to reduce the effort required to put the solution in place.

### Risks and Mitigations

#### Take to long to be executed

A risk is the command take to long to be executed. In order to mitigate it, the flags `--wait-time` and options to disable some checks will be also be added. 

**Scorecard performance**

See [here](https://github.com/operator-framework/operator-sdk/blob/master/testdata/go/v3/memcached-operator/bundle/tests/scorecard/config.yaml#L6) that default configuration will allow scorecard run the tests using parallelism. Following the execution times for this feature using the [Memcached bundle sample](https://github.com/operator-framework/operator-sdk/tree/master/testdata/go/v3/memcached-operator/bundle) in a local environment:

```sh
$ time operator-sdk scorecard bundle
real 0m7.264s
user 0m0.131s
sys 0m0.039s
 
$ time operator-sdk scorecard bundle --output=json --selector=suite=basic
real 0m3.296s
user 0m0.088s
sys 0m0.040s
 
$ time operator-sdk scorecard bundle --output=json --selector=suite=olm
real 0m5.661s
user 0m0.104s
sys 0m0.030s
```

By using as a reference an OpenShift catalog for 4.6 it has 224 bundles which would take in average 30 minutes to execute all scorecard checks for all operator bundles. 

It still done in a reasonable time for we are able to audit all operators of a catalog with. However, note that the flag `--disable-scorecard` will allow users disable this check when required. Also, we have the flag `--head-only` which will make the audit command only check the latest bundles and will decrease this time a lot.

## Design Details

### Version Skew Strategy

The audit command at this stage does not require support and work with the previous OLM index catalog format/versions or the old Package Manifest format.

## Implementation History

Note that this proposal shows replace the need of others open pull requests such as:
- [Enhancement for validating bundles in a catsrc image #48](https://github.com/operator-framework/enhancements/pull/48). The audit command shows cover all requirements raised in this proposal.

[ep-new-checks-for-operator-hub]: /enhancements/operator-hub-check
[operator-framework/api]: https://github.com/operator-framework/api
[ocp-validator-proposal]: https://github.com/openshift/enhancements/pull/661
[sdk-scorecard]: https://sdk.operatorframework.io/docs/advanced-topics/scorecard/
[sdk-make-bundle]: https://github.com/operator-framework/operator-sdk/blob/v1.4.2/testdata/go/v3/memcached-operator/Makefile#L126-L131
[writing-custom-scorecard]: https://sdk.operatorframework.io/docs/advanced-topics/scorecard/custom-tests/
[community-operators-with-scorecard]: https://github.com/operator-framework/community-operators/tree/master/community-operators/namespace-configuration-operator/1.0.1