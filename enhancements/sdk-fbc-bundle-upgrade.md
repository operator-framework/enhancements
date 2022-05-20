---
title: sdk-fbc-run-bundle-upgrade
authors:
  - "@rashmigottipati"
reviewers:
  - "@joelanford"
  - "@VenkatRamaraju"
  - "@everettraven"
  - "@jmrodri"
approvers:
  - "@VenkatRamaraju"
  - "@everettraven"
creation-date: 2022-04-26
last-updated: 2022-05-19
status: implementable
see-also:
replaces:
superseded-by:

---

# Add FBC support to run bundle-upgrade

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [X] Design details are appropriately documented from clear requirements
- [X] Test plan is defined
- [X] Graduation criteria for dev preview, tech preview, GA

## Open Questions

- Should the FBC image be a required flag for bundle-upgrade?  

  - The FBC or SQLite image can always be inferred from the annotations (for both `run bundle` and traditional OLM case).
  - For the cases where the previous version of the operator was installed traditionally, we can infer the latest CSV from channel entries of the rendered index image.

- We have bundle add mode that describes how a bundle is added to an index image; do we need those for FBC support? If so, are there any new modes for FBC other than `semver` and `replaces`?

  - There are no add modes for FBC, hence, we should error out when a mode flag is specified and if the index image is a File-Based Catalog/Default Index image.

## Summary

The goal of this enhancement is to discuss how the `run bundle-upgrade` command in Operator SDK can begin to support and adopt the new format for indexes along with continuing to support the existing SQLite index images.

## Motivation

Currently, there is `run bundle-upgrade` command in the Operator SDK for upgrading an operator bundle to a newer version using OLM. OLM has transitioned to the new [File-Based Catalog (FBC)](https://olm.operatorframework.io/docs/reference/file-based-catalogs/#docs) for catalog images which is a more human readable and consumable format of a catalog that is in the form of `JSON/YAML`. As of now, Operator SDK's `run bundle-upgrade` command upgrades an operator bundle using [opm tooling](https://github.com/operator-framework/operator-registry/blob/master/docs/design/opm-tooling.md#opm) that uses a SQLite database under the hood, which creates the registry database as well as index images. Support for these opm commands will be completely removed in a future release, hence Operator SDK should enable support for the new FBC format along with continuing to support the existing SQLite format so that Operator Authors can continue using SDK to perform and test operator upgrades on cluster using OLM for both formats.

### Goals

- Upgrade to a newer version of an operator bundle on-cluster when the previous version of the operator was installed through `run bundle` command using the default index image.

- Upgrade to a newer version of an operator bundle on-cluster when the previous version of the operator was installed through `run bundle` command using an FBC or a SQLite based index image.

- Upgrade to a newer version of an operator bundle on-cluster when the previous version of the operator was installed traditionally using OLM by creating a `Catalog Source`, `Operator Group`, and a `Subscription`.

- The CLI for `run bundle-upgrade` command should remain the same i.e. no user-facing changes.

### Non-Goals

- This proposal and the implementation for `run bundle-upgrade` to support FBC will only perform incremental upgrades, i.e. a single upgrade at a time and skipping through upgrades isn't allowed.

  - Using `skip` and `skipRanges` is out of scope for this EP and would require future amendment of the EP or a new enhancement if we decide to support those.

- Migrating old catalog images that are of SQLite format to newer catalog versions with FBC support is not a goal of this EP.

## Proposal

### User Stories

#### Story 1

As an Operator SDK user, I want to be able to upgrade my operator bundle with File-Based Catalog support to a newer version from a previous version, if the original bundle was installed using the `run bundle` command, with and without the use of the `--index-image` flag that can either be of FBC or SQLite format.

#### Story 2

As an Operator SDK user, I want to upgrade my operator bundle to a newer version when the previous version of the operator was installed traditionally using OLM with a catalog image that can either be of FBC or a SQLite format.

#### Story 3

As an Operator SDK/OLM maintainer, I want to ensure that Operator SDK is updated to use the latest format and APIs provided by OLM/Operator Registry to mimimize technical debts so that Operator Developers can test their operator upgrades using both index formats (FBC and SQLite).

### Implementation Details/Notes/Constraints [optional]

#### Command Line Interface

The command line interface will remain exactly the same as the original `run bundle-upgrade` command.

#### Assumptions

We make the following assumptions for operator upgrade using the Operator SDK:

1. OLM is already installed.
1. A Kube API is available with permissions to create resources.
1. Kubelet can see and pull the newer version of the bundle image from a registry.
1. A previous version of the operator bundle has already been installed in the cluster either via `run bundle` command or traditionally using OLM.

#### What happens when the `run bundle-upgrade` command is invoked?

Let's consider the first use-case when the original bundle was installed using `run bundle` command:

1. The labels from the bundle annotations are parsed and used to construct an SDK-defined object, `FBCContext`. This contains various data points that will be used to invoke Operator Registry APIs and enable in creation of the resulting File-Based Catalog.

2. There are two possible scenarios for the run bundle-upgrade: if run bundle was invoked  with or without an index image. The `run bundle` command (which is a prerequisite for the `run bundle-upgrade` command) will create a `CatalogSource` resource in the cluster. The `CatalogSource` resource has annotations that indicate whether the `--index-image` flag was used during the invocation of run bundle along the list of bundles that have been injected so far.

    - Without an index image: If an index image was not specified as a flag during the run bundle invocation, the Operator SDK will generate its own File-Based Catalog for the `run bundle-upgrade` process. It does so by obtaining the list of previously injected bundles from the CatalogSource’s annotations and renders these bundle images to form bundle blobs. The Operator SDK will also add a Package and Channel blob to generate a File-Based Catalog representation of the injected bundles and propagate this object to the ephemeral pod for the upgrade process.

    - With an index image: There are two sub-possibilities on the type of index image that was passed into the `run bundle` command:

        - SQLite based Image: The Operator SDK will continue the `run bundle-upgrade` process as it currently does: via opm index add and opm registry serve on the ephemeral pod.

        - File-Based Catalog Image: If a File-Based Catalog was passed during the invocation of the run bundle command, the Operator SDK will render this index image into a declarative config format. It will also render out the list of previously injected bundles, which can be obtained from the CatalogSource’s annotations. Then, the Operator SDK will merge these two declarative configs into one, i.e., it will insert all the injected bundles, channels and packages into the index image declarative config. The channel entries will be inserted in the order of the injected bundles, creating a sequential linear update graph of the injected bundles. In this way, a new File-Based Catalog is created and will be propagated to the ephemeral pod for the upgrade process.

3. The Operator SDK will perform the upgrade in the following manner: In the specified channel, the Operator SDK will insert an entry into the Entries field of the Channel object (defined by Operator Registry code). The element that will be inserted into the Channel Entries looks like the following:

    ```go
      declarativeconfig.ChannelEntry{
      Name:     “memcached-operator.v0.0.2”,
      Replaces:  “memcached-operator.v0.0.1”,
    }
    ```

    - This entry will imply that the `memcached-operator.v0.0.1` will be upgraded to `memcached-operator.v0.0.2` when the command succeeds. The "Name" field is taken from the CSV name of the bundle image that is passed in through the `bundle-image` argument of the `run bundle-upgrade` command. The "Replaces" field is taken from the last element of the "injectedBundles" array that is present on the CatalogSource annotations. Using these two data points, the ChannelEntry object is constructed that will be used in the upgrade process.

4. The Operator SDK will append all the bundle blobs corresponding to the CSVs in the InjectedBundles array (from CatalogSource annotations) along with the bundle blob of the latest bundle image (taken from the argument of the `run bundle-upgrade` command)

5. Operator SDK will invoke more Operator Registry APIs to ensure that the upgraded FBC generated from the above steps is indeed a valid File-Based Catalog.

6. During the creation of the ephemeral pod (which is used to serve the File-Based Catalog to gRPC), the Operator SDK will invoke a set of CLI commands to create a directory and file on the pod during runtime.

7. The contents of the generated File-Based Catalog will be written to the file that is created on the ephemeral pod using the following command template -

    ```go
    mkdir -p {{ .FBCdir }} && \
    echo {{ .FBCcontent }} >> {{ .FBCfile }} && \
    opm serve {{ .FBCdir }} -p {{ .GRPCPort }}
    ```

    - Note: It is worth noting that the new command to serve a File-Based Catalog to gRPC has been changed from `opm registry serve` to `opm serve`. The above command template reflects this change. `opm registry` commands will be deprecated and completely removed in a future release.

8. In this way, the File-Based Catalog that was generated by the Operator SDK will be served to the gRPC API via the specified port (defaulted to 50051). The remaining portion of the upgrade process will remain the same.

#### What happens when the `run bundle-upgrade` command is invoked in the traditional case?

Now, lets consider the second use-case when the original bundle was installed traditionally using OLM:

1. When an operator is installed traditionally via OLM a `CatalogSource` and a `Subscription` will have been created. When installed traditionally, the `CatalogSource` will not contain annotations so the version being upgraded from will need to be inferred.

    Traditional CatalogSource example:

    ```yaml
    apiVersion: operators.coreos.com/v1alpha1
    kind: CatalogSource
    metadata:
     name: hello-kubernetes-operator-catalog
     namespace: operators
    spec:
     displayName: Hello Kubernetes Operator
     image: <registry>/<test>/hello-kubernetes-fbc:latest
     sourceType: grpc
    ```

    Traditional Subscription example:

    ```yaml
    apiVersion: operators.coreos.com/v1alpha1
    kind: Subscription
    metadata:
      name: hello-kubernetes
     namespace: operators
    spec:
      channel: "alpha"
      installPlanApproval: Automatic
      name: hello-kubernetes
      source: hello-kubernetes-operator-catalog
      sourceNamespace: operators
      startingCSV: hello-kubernetes.v0.0.1
    ```

    - We are able to get the index image that is used in a traditional OLM installation by retrieving the value in the `spec.image` field in the example `CatalogSource`.

    - It can also be seen in the example `Subscription` that we are able to get the channel that is used in a traditional OLM installation by retrieving the value in the `spec.channel` field.

    - Using these values we can render the index image used in the `CatalogSource` and search for channel entries that match the channel that is specified in the `Subscription`. To infer the latest CSV that was installed we can check the channel entries for an entry that does not appear in a replaces field. This latest CSV obtained will be the edge to upgrade from.

2. For example, given the following channel entries in an index image:

    Example channel entries:

    ```json
    {
      "schema": "olm.channel",
      "name": "alpha",
      "package": "hello-kubernetes",
      "entries": [
          {
              "name": "hello-kubernetes.v0.0.1"
          },
          {
              "name": "hello-kubernetes.v0.0.2",
              "replaces": "hello-kubernetes.v0.0.1"
          }
        ]
      }
    ```

    - It can be inferred that the most recently installed CSV would be `hello-kubernetes.v0.0.2` as it does not appear in a channel entry replaces field.

3. After obtaining the CSV name from the above step, Operator SDK will insert an entry into the Entries field of the Channel object which looks like the following:

    ```go
        declarativeconfig.ChannelEntry{
        Name:     “hello-kubernetes.v0.0.3”,
        Replaces:  “hello-kubernetes.v0.0.2”,
    }
    ```

    - From this example, `hello-kubernetes.v0.0.3` will be the CSV that we will be upgrading to from `hello-kubernetes.v0.0.2`.

4. This entry will be added to the rendered index image declarative config and the upgraded bundle will also be appended to the index image declarative config.

5. Furthermore, Operator SDK will convert the index image declarative config to a string format, ensure that the rendered declarative config is a valid FBC and serve the validated FBC during the ephemeral registry pod creation.

- Note: Additionally, during `run bundle-upgrade` in this use-case, Operator SDK adds annotations such as bundle image, index image and injected bundles to the `CatalogSource` annotations which will be used for the subsequent upgrades without having to infer from the channel entries of the index image declarative config.

- Now let's consider upgrading from `hello-kubernetes.v0.0.3` to the next version `hello-kubernetes.v0.0.4` from the above example:
  
  - For this case, the annotations would have been written out to `CatalogSource` during the previous upgrade to `v0.0.3`.

  - We obtain the index image from CatalogSource's `spec.image` field and render it.

  - We infer that the upgrade edge is `hello-kubernetes.v0.0.2` from the bundle image annotation on the `CatalogSource` and also obtain the injected bundles from the annotation which currently only has `hello-kubernetes.v0.0.3`. Every CSV in the injected bundles array will get replaced by it's next element, creating a linear upgrade graph.

  - Thereby, we construct the entries that upgrades from `v0.0.1` -> `v0.0.2` -> `v0.0.3` -> `v0.0.4`, so the latest upgrade edge in the index image will get upgraded to the first bundle in the injected bundles array and eventually get upgraded to `v0.0.4` which is the bundle we want to upgrade to.

  - In this way, Operator SDK constructs entries in a sequential order and performs the upgrade to the latest entry in the array. Operator SDK also adds relevant packages, channels, bundles to the index to form a complete File-Based Catalog Representation.

#### Prototype Links

- [run bundle-upgrade POC](<https://github.com/operator-framework/operator-sdk/commit/b8480288899f52b04f0f0a56bf4376d143abaed3>).

### Risks and Mitigations

## Design Details

### Test Plan

- Manual and automated tests with different scenarios.
  
  - Upgrading an operator bundle v1 --> v2 --> v3 without an index image in the `run bundle` invocation. This is the default case.
  - Upgrading an operator bundle v1 --> v2 --> v3 with an index image in the `run bundle` invocation. The index image can either be a SQLite or a File-Based Catalog image.
  - Upgrading an operator bundle v1 --> v2 --> v3 when v1 was installed traditionally using OLM without invoking `run bundle`.
  - Upgrading an operator bundle  v1 --> v2 --> v3 with an index image that also contains various other bundles.

### Graduation Criteria

The `run bundle-upgrade` command with FBC support will come in as non-alpha and be ready to use once merged.

### Upgrade / Downgrade Strategy

### Version Skew Strategy

N/A

## Implementation History

## Drawbacks

N/A

## Alternatives

## Infrastructure Needed [optional]

N/A
