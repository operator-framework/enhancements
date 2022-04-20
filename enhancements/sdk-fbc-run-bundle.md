---
title: sdk-fbc-run-bundle
authors:
  - "@rashmigottipati"
reviewers:
  - "@joelanford"
  - "@jmrodri"
  - "@VenkatRamaraju"
  - "@camilamacedo86"
approvers:
  - "@joelanford"
  - "@jmrodri"
  - "@VenkatRamaraju"
creation-date: 2022-04-20
last-updated: 2022-04-20
status: implementable
see-also:
replaces:
superseded-by:

---

# Add FBC support to run bundle

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [X] Design details are appropriately documented from clear requirements
- [X] Test plan is defined
- [X] Graduation criteria for dev preview, tech preview, GA

## Open Questions

- Do we need to build a File-Based Catalog to run this command?

  - In the case where an index image isn't provided, we need to build a File-Based Catalog on the fly when `run bundle` command is invoked.

- Do we need to make our own image to use as a default one for `run bundle` and `run bundle-upgrade`?

  - Currently, we are using `quay.io/operator-framework/opm:latest` as the default index image. This image should suffice as the default image for both index formats (FBC and SQLite images).

- While initializing the FBC, should the channel received from the bundle annotations be used to initialize the default channel name in the `opm init` command?

  - We should be able to handle bundles that don't have channel metadata on the annotations. We can either generate a random name or a non-random channel name with a meaningful prefix or a suffix.

- If we don’t get channel information from the bundle, can we add it to a default channel or to the channel HEAD?

  - For FBC, channel membership and package-level metadata embedded in bundles is ignored. So we should design FBC tooling to ignore that as well. The channel name can either be hardcoded (e.g. channel name = `develop`) or it can be derived. Hardcoding the channel name may be better as deriving it to a name that contains the package name seems to be redundant as the channel is already a member of the package.

- Can we assume the Channel’s “entries” field be initialized to an array with only one element, i.e., {name: “api-operator.v.0.1”} with no “replaces” field?

  - "replaces" field is required for `run bundle-upgrade` but for run bundle, it's ok to assume for the channel entries to have just one entry.

- Should we be pinning the upstream opm image `quay.io/operator-framework/opm:` or always use latest.
  - We should consider pinning this image to downstream builds.

## Summary

The goal of this enhancement is to discuss how the `run bundle` command in Operator SDK can begin to support and adopt the new format for indexes along with contuining to support the existing SQLite based index images.

## Motivation

One of the main goals of the Operator SDK is to enable operator developers to install and run their operators using Operator Lifecycle Manager (OLM). Currently, there are two commands in the Operator SDK for installing an operator bundle (`run bundle`) and upgrading an operator bundle to a newer version (`run bundle-upgrade`) using OLM. Operator SDK installs and upgrades the operator bundles using opm tooling that inherently uses a SQLite database under the hood, which is responsible for generating the registry database as well as the index images. We use `opm registry add` under the hood to create an ephemeral index image during the `run bundle` workflow; however, that command is being deprecated.

As OLM is transitioning to declarative config, which is a more human readable and consumable format of a catalog that is in the form of `JSON/YAML`, Operator SDK should transition the existing `run bundle` command to support declarative config index images and default to using File-Based Catalogs so that Operator authors can continue using Operator SDK to install their operator bundles using the new FBC format.

This proposal discusses changes to the `run bundle` functionality to be able to support File-Based Catalogs and how the registry pod should be created to support the generated FBC format i.e., to modify/replace the existing opm commands used to create the ephemeral index image during registry pod creation by leveraging the Operator Registry APIs.

### Goals

- Operator SDK’s `run bundle` command should install an operator bundle by creating a valid File-Based Catalog when a user does not provide an existing index image i.e. using a default index image.

- Operator SDK should allow users to install their operator bundle when they provide an existing index image by generating a valid File-Based Catalog and inserting the operator bundle into the index image.

- Operator SDK's `run bundle` command should still be able support the existing SQLite index images along with the FBC images until support for SQLite based images are deprecated and completely removed by OLM.

- The CLI for `run bundle` command should remain exactly the same i.e. no user-facing changes.

### Non-Goals

- Operator SDK’s `run bundle` and `run bundle-upgrade` commands share a lot of common functionality and this EP does not discuss the impact on the run bundle-upgrade command. A separate EP is required for the `run bundle-upgrade` command.

- Migrating from old catalog images to newer catalog versions with FBC support is not a goal of this EP.

- Another non-goal, is that `run bundle` is **NOT** intended to help users build and manage File-Based indexes. It is an opinionated command to help users test simple bundle-based execution scenarios with ephemeral indexes.

## Proposal

### User Stories

#### Story 1

As an Operator SDK/OLM maintainer, I want to ensure that the SDK is updated to use the latest format and APIs provided by OLM/Operator Registry so that there are no technical debts and we will be able to remove the codepaths that has support for the old image format when they are deprecated and completely removed by OLM.

#### Story 2

As an Operator SDK user, I would like to be able to provide an index image that is of the FBC format via the `--index-image` flag in the `run bundle` CLI, so that I can use this option with the index catalog built from [OperatorHub catalog](quay.io/operatorhubio/catalog) for example.

#### Story 3

As an Operator SDK user, I would like to still be able to provide an index that is of the SQLIndex format via the `--index-image` flag in the `run bundle` CLI, so that I can use this option with old indexes or indexes from sources that did not adopt the new format yet.

### Implementation Details/Notes/Constraints [optional]

The Operator SDK currently supports running an operator bundle using OLM via the run bundle command, which is suited for the SQLite-based index format. This enhancement will address how the Operator SDK will support the new File-Based Catalog format for operator catalogs.  More Information on FBCs and how operator bundles are created using Operator SDK can be found at the links below:

- [File-Based Catalogs](https://olm.operatorframework.io/docs/reference/file-based-catalogs/)
- [File-Based Catalog Issue](<https://github.com/k8s-operatorhub/community-operators/discussions/505>)
- [Operator-Bundles](<https://github.com/operator-framework/operator-registry/blob/master/docs/design/operator-bundle.md>)

The declarative config is a struct defined by the Operator Registry that represents a File-Based Catalog in a format that is easily transformed to a YAML/JSON format. The Operator SDK will invoke Operator Registry APIs to generate various components of a declarative config and eventually insert them into the declarative config object. This object will then be converted to a human-readable form like `YAML/JSON`.

```go
type DeclarativeConfig struct {
  Packages []Package
  Channels []Channel
  Bundles []Bundle
  Others []Meta
}
```

As seen in the struct above, a declarative config has three major components - Bundles, Packages, and Channels. For more information about these subtypes, refer to [Operator Registry DeclarativeConfig definitions](<https://github.com/operator-framework/operator-registry/blob/master/alpha/declcfg/declcfg.go>).

Essentially, a basic File-Based Catalog is composed of these three components in `JSON/YAML` format. The Operator SDK generates each of these components from the Operator Registry APIs in the following manner:

[Bundle](<https://github.com/operator-framework/operator-sdk/blob/18ca92070fb3dd5e9f7d4581b27cc3e167a082d9/internal/olm/operator/bundle/install.go#L239>)

[Package](<https://github.com/operator-framework/operator-sdk/blob/18ca92070fb3dd5e9f7d4581b27cc3e167a082d9/internal/olm/operator/bundle/install.go#L251>)

[Channel](<https://github.com/operator-framework/operator-sdk/blob/18ca92070fb3dd5e9f7d4581b27cc3e167a082d9/internal/olm/operator/bundle/install.go#L260>)

These three structs are used to construct the declarative config object. This declarative config object is a File-Based Catalog representation of an index with a single bundle present in it. This declarative config object can easily be transformed into `YAML/JSON` format.

#### Command Line Interface

The command line interface will remain exactly the same as the original `run bundle` command.

#### Assumptions

Like the existing run bundle command, we **assume** the following:

 1. OLM is already installed
 1. kube API is available with the permissions to create resources
 1. kubelet can see and pull the provided images (i.e. bundle and index images)
 1. If an index image is provided, we assume that it has:
    - `sh`, so we can change the entrypoint
    - if it is a sqlite index image, we assume that:
        - the existing registry DB located at `/database/index.db`
        - dependencies required for `opm registry add` so we can inject the bundle into the DB

#### What happens when the `run bundle` command is invoked?

1. The labels from the bundle annotations are parsed. The annotations are received from the bundle image that is passed in as a required argument for the run bundle command. Operator SDK will parse information like channel name and CSV name from the labels in the bundle annotations.

2. Additionally, the Operator SDK also parses the label on the index image. The label `operators.operatorframework.io.index.configs.v1=/configs` is used to differentiate FBC images from SQLite images. If the FBC label exists on the index image, we will always create the FBC on the fly and we have two scenarios:

    - If no index image was provided, we will serve the FBC that was generated on the fly.
    - If an FBC index image was provided, we will still generate the FBC (of the bundle-image) on the fly and then add it to the FBC representation of the index image. This resulting FBC (that has the new bundle in it) will get served.

    Note: If the label is not found on the image, that means the image is of the SQLite index format, in which case, we follow through the existing implementation of `run bundle`.

3. The Operator SDK, using the data gathered from the above steps as parameters, will invoke Operator Registry APIs that will generate three major components in the form of Operator Registry-defined objects - Package, Channel & Bundle.

4. The Operator SDK will convert these objects into more commonly-consumable formats like JSON/YAML blobs and combine these blobs together to generate a basic File-Based Catalog representation of the bundle image. In other words, the simplest File-Based Catalog consists of one package blob, one channel blob, and one bundle blob. These blobs represent the bundle data. This step represents the "generating FBC on the fly" scenario that we referred to in this document.

5. If an index image was passed in as a parameter through the `--index-image` flag of the run bundle command, a series of additional steps take place:

    - The index image will be rendered into a declarative config object, similar to the bundle image.
    - Operator SDK will check if the bundle image is already present in the index image.
        - If the bundle image is already present in the index image, Operator SDK will not add the bundle to the index but rather error out.
        - If the bundle is not present in the index image, Operator SDK will add the contents of the bundle image to the existing index image and serve the resulting index image (with the bundle added to it).

    - Note: If the index image that was passed is in SQLite format, Operator SDK will use a different container creation command that uses `opm registry add` to add the bundle to the index and `opm registry serve` to serve the index which is a database file on created on the pod.

6. The Operator SDK will invoke more Operator Registry APIs to ensure that the FBC that was generated from the above steps is indeed a valid FBC. This steps essentially validates the generated FBC by converting the declarative config object to a model object which internally performs validations equivalent to the `opm validate <FBC>` command.

7. Three unique data points will be stored in the `IndexImageCatalogCreator` object that will eventually be used in the ephemeral pod:

    - FBC directory name where the FBC file resides in → `FBCDir`
    - FBC file name that has all the contents to be served through the pod → `FBCFile`
    - Contents of FBC file (string YAML/JSON) → `FBCContent`

8. During the creation of the ephemeral pod (which is used to serve the File-Based Catalog to gRPC), the Operator SDK will invoke a set of CLI commands to create a directory and file on the pod during runtime.

9. The contents of the generated File-Based Catalog will be written to the file that is created on the ephemeral pod using the following command template -

    ```go
    `mkdir -p {{ .FBCDir }} && \
    echo '{{ .FBCContent }}' >> {{ .FBCFile }} && \
    opm serve {{ .FBCDir }} -p {{ .GRPCPort }}`
    ```

    - Note: It is worth noting that the new command to serve a File-Based Catalog to GRPC has been changed from `opm registry serve` to `opm serve`. The above command template reflects this change. `opm registry` commands will be deprecated in a future release. Also, there's a limit to the size of `FBCContent` string. This becomes an issue when a user provides an index image via the CLI, as `run bundle` would fail in those cases due to the request entity being too large. This is a potential risk and needs to be investigated/solved in the implementation.

10. In this way, the File-Based Catalog that was generated by the Operator SDK will be served to the gRPC API via the specified port (defaulted to `50051`) without any user-facing changes.

11. The remaining resources (`CatalogSource`, `Subscription`, `OperatorGroup`) are created in the same way as before. The run bundle command does not change from here on.

- Note: If a SQLite index is passed, the old commands will be used in the registry pod and none of the FBC creation will happen. Below is the container creation command used to handle SQLite index images (the optional flags are skipped here for simplicity purposes):

    ```go
    `mkdir -p {{ dirname .DBPath }} && \
    {{- range $i, $item := .BundleItems }}
    opm registry add -d {{ $.DBPath }} -b {{ $item.ImageTag }} --mode={{ $item.AddMode }} && \
    {{- end }}
    opm registry serve -d {{ .DBPath }} -p {{ .GRPCPort }}`
    ```

#### Example UX Scenarios

Running the following scenarios results in different outcomes for FBC creation. These are some example uses of the `run bundle` command. This is a not an exhaustive list of the command with the other flags.

```sh
- operator-sdk run bundle <bundle-image>
- operator-sdk run bundle <bundle-image> --index-image quay.io/example/testFBCCatalog
- operator-sdk run bundle <bundle-image> --index-image quay.io/example/testSQLiteCatalog
```

- `operator-sdk run bundle <bundle-image>` would create a File-Based Catalog (FBC) on the fly by rendering the bundle image and using the default upstream opm image `quay.io/operator-framework/opm:latest`. Bundle image contents will be served over a gRPC port through the ephemeral registry pod.
- `operator-sdk run bundle <bundle-image> --index-image quay.io/example/testFBCCatalog` would ensure that the bundle image doesn't exist in the `testFBCCatalog` index. If it exists, then we error out. If not, we add the bundle image to the index image by rendering both of these images using the Operator Registry APIs thereby appending the bundle contents to the index and serving the index over the gRPC port.
- `operator-sdk run bundle <bundle-image> --index-image quay.io/example/testSQLiteCatalog` will still be supported in the old way by making use of `opm registry add` and `opm registry serve` commands under the hood and without creating an FBC or rendering the SQLite based image.

#### Prototype Links

- [run bundle prototype branch](<https://github.com/rashmigottipati/operator-sdk/tree/run-bundle-prototype>).
- Initial [POC](<https://github.com/rashmigottipati/run-bundle-POC>) to create a declarative config using Operator Registry APIs.

### Risks and Mitigations

The following are some of the risks to using the default upstream opm image `quay.io/operator-framework/opm:`:

  1. if image doesn't have `sh`, we can't modify the entrypoint.
  1. if opm is building index images without `sh`, then there will be no index images we support.

The following is a risk to passing the index image via the CLI:

  1. As there is a limit on the `FBCContent` string, `run bundle` would fail with "request entity too large" in some scenarios where the index image size is huge. This needs to be investigated further and solved. However, this is not a concern for the default case. There are two potential ways to solve this:
  
      - Run the FBC catalog image and inject just the extra FBC that we need to generate for the new bundle.

      - Handle creation of FBC registry pod without embedding the rendered FBC content directly in the pod spec, by mounting from `ConfigMaps`.

## Design Details

### Test Plan

- Manual testing with different scenarios.
  
  - Installing an operator bundle when a user provides the operator bundle image they intend to install. This is the default case.
  - Installing an operator bundle when a user provides an index that is a File-Based Catalog (FBC) image.
  - Installing an operator bundle when a user provides an index that is a SQLite based image.
  - Installing an operator bundle when the FBC index provided by the user already has the operator bundle in it. This should error out.
  - Installing an operator bundle when the SQLite index provided by the user already has the operator bundle in it. This should cause the registry pod to fail with the error `Bundle already exists, Bundle already added that provides package and csv`.

- Unit tests for the new FBC changes added to the implementation (with coverage requirements).

- Possibly e2e tests as well (e2e tests don’t exist for the current run bundle command).

### Graduation Criteria

The `run bundle` command with fbc support will come in as non-alpha and be ready to use once merged.

### Upgrade / Downgrade Strategy

### Version Skew Strategy

N/A

## Implementation History

- [PR](<https://github.com/operator-framework/operator-sdk/pull/5671>) that implements FBC support in run bundle.

## Drawbacks

N/A

## Alternatives

There is a potential alternative to handling SQLite based images. Instead of taking the old route of using `opm registry add` and `opm registry serve` commands during the creation of ephemeral registry pod, we could render the SQLite based image using Operator Registry APIs or migrate the SQLite image to the new FBC format and serve the contents through the registry pod, which will trigger the installation of an operator bundle. Essentially, treat both images to be the same by rendering and serving the contents. Although this would require dealing with additional compilation to render the SQLite images to FBC. This is a potential alternative to when the `opm commands` such as `opm registry add` and `opm registry serve` are deprecated and support for those will be removed.

- Pros

  - Enables having one route for both SQLite and FBC images which is to use the Operator Registry APIs to render the images.

  - Easy to maintain without having to provide separate code paths for SQLite and FBC scenarios.

- Cons

  - Degredation to UX as converting SQLite images to the FBC format takes more time.

  - Dealing with the compilation issues and additional build complexity in the SDK to cross-compile for all the target OSes and arches.

  - Not all SQLite indexes will contain an `opm` binary with the `migrate` command so run bundle will break in those cases.

Another alternative (a potential follow-up) could be to handle creation of FBC catalog pods without embedding the rendered FBC content directly in the pod spec. This needs some investigation on whether we could mount it from one or more `ConfigMaps`.

## Infrastructure Needed [optional]

N/A
