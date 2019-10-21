# Disconnected OLM

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift/docs]

## Summary

This enhancement outlines the changes necessary to `olm`, `operator-registry`, and `opm` to support taking operator catalogs and packaging them for installation in an offline cluster.

This enhancement also covers how this process will change as our distribution mechanism changes and appregistry support is deprecated.

## Motivation

Disconnected installation of OLM catalog data requires a different approach from other openshift components, because:

 - The catalog of operator data currently exists in appregistry repositories (Quay.io)
 - That catalog references a set of operators that are versioned out-of-sync with OpenShift
 - Each version of each operator may require an additional set of `operand` images which need to be mirrored.

The [current process](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.2/html/operators/olm-restricted-networks) for offlining a catalog involves many manual steps, and may fail to capture all images required for disconnected environments. 


### Goals
* Define a mechanism for operators to specify related operand images that should be collected for disconnected installation.
* Define the changes required in `olm` and `operator-registry` to allow those images to be discovered
* Define `opm` commands which can build and mirror all images required for mirroring to a disconnected envorinment

### Non-Goals
* Provide an api for fine-grained control over which operators / operands are mirrored. 

## Proposal

The basic approach will be to:

- Allow operator manifests to specify [related images](https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/contributors/design-proposals/related-images.md) in their manifests
- Generate catalog container images that can replace the appregistry repositories
- For each operator within the catalog image, mirror it to the disconnected environment


### Specify related images

We add a new field to the `ClusterServiceVersion` spec. 

```yaml
kind: ClusterServiceVersion 
spec:
  relatedImages:
  - name: default
    image: quay.io/coreos/etcd@sha256:12345 
  - name: etcd-2.1.5
    image: quay.io/coreos/etcd@sha256:12345 
  - name: etcd-3.1.1
    image: quay.io/coreos/etcd@sha256:12345  
```

This is covered in depth in the [related images](https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/contributors/design-proposals/related-images.md) proposal for OLM.

### Expose related images

`operator-registry` has tools for building an index of operator manifests into a sqlite artifact, with interfaces provided for querying that database.

A new table will be added to the sql schema:

```sql
	CREATE TABLE IF NOT EXISTS related_image (
	    image TEXT,
		operatorbundle_name TEXT,
	    FOREIGN KEY(operatorbundle_name) REFERENCES operatorbundle(name)
	);
```

Two new queries should be added to allow effecient collection of related images:

```go
type Query interface { 
  ListImages(ctx context.Context) ([]string, error)
  GetImagesForBundle(ctx context.Context, operatorName string) ([]string, error)
  // the rest of the existing interface
}
```

The loading process will read the new fields in the `ClusterServiceVersion` and populate the `bundle_image` table with the related images. Consumers can query for those via the provided query interface.

There does not currently seem to be a reason to expose this over the registry's grpc API, though nothing prevents that in the future.

### Generate and mirror a catalog

`opm` will be extended with a command:

```sh
$ opm registry build --appregistry-namespace=quay.io/community-operators --to quay.io/ecordell/community-operators-catalog:4.2.1 
```

`opm registry build` will:

- Use appregistry protocol to retrieve all of the operators in a namespace of Quay.io specified by `appregistry-namespace`
- Load all of the downloaded operators into a versioned `operator-registry` sqlite artifact
- Build a runnable operator-registry image by appending the database to the `operator-registry` base image.
- Mirror that image to the target defined by `--to`

It will have the following flags:

- `--from=ref` - the base image to add the built operator database into. Defaults to the operator-registry image shipped with the version of OpenShift that `opm` came from.
- `--to=ref` - the location that the image will be mirrored.
- `--save=path` - instead of mirroring, this will output a tarball containing the image
- `--auth-token=string` - the auth token ([instructions](https://github.com/operator-framework/operator-courier#authentication)) for authenticating with appregistry.
- `--appregistry-endpoint=url` - the CNR endpoint to authenticate against. Defaults to `"https://quay.io/cnr"`, the endpoint used by OpenShift 4.1-4.3.
- `--appregistry-namespace=string` - the namespace in appregistry to mirror. Each repository within the namespace represents one operator. 
- `--db-only` - if true, and `--save` is specified, the operator database will be saved in the directory noted by `--save`.

This command generates and mirrors in one step, because we do not assume that there is any registry available aside from the target disconnected registry. `--save` can be used to save the image locally without a registry available.

### Extract the contents of a catalog for mirroring

`opm` will be extended with a second command:

```sh
$ opm registry images --from=quay.io/ecordell/community-operators-catalog:4.2.1 --to=localhost:5000/community-operators --manifests=./mirror-manifests

quay.io/coreos/etcd-operator@sha256:12345=localhost:5000/community-operators/quay-io-coreos-etcd-operator quay.io/coreos/etcd@sha256:12345=localhost:5000/community-operators/quay-io-coreos-etcd quay.io/coreos/etcd@sha256:54321=localhost:5000/community-operators/quay-io-coreos-etcd
...

$ ls ./mirror-manifests
imagecontentsourcepolicy.yaml
catalogsource.yaml
```

`opm registry images` will:

- Pull the catalog image referenced by `--from`
- Read the database to get the list of operator and operand images
- Output that list in a form that can be consumed by `oc image mirror`
- Output a set of manifests that, if applied to a cluster that has access to the mirrored images, will correctly configure nodes and OLM to use those images.

It has the following flags:

- `--from=ref` - a catalog image (such as one built by `opm registry build`)
- `--from-archive=path` - a reference to a tarball image, such as one built with `opm registry build --save=path`
- `--to=ref` - optional; the location that the image will be mirrored. This should include only a registry and a namespace that images can be mirrored into.
- `--manifests=path` - default `./manifests`, the path at which manifests required to mirror these images will be created. This includes an `ImageContentSourcePolicy` that can configure nodes to translate between the image references stored in operator manifests and the mirrored registry, and a `CatalogSource` that configures OLM to read from the mirrored catalog image referenced by `--from`.

If `--to` is omitted, `opm registry images` will only output a list of images:

```sh
$ opm registry images --from=quay.io/ecordell/community-operators-catalog:4.2.1

quay.io/coreos/etcd-operator@sha256:12345
quay.io/coreos/etcd@sha256:12345
quay.io/coreos/etcd@sha256:54321
...
```

Advanced users may wish to use this list to perform their own transformations / mirroring steps.

### Full Example

```sh
$ opm registry build build --appregistry-namespace=quay.io/community-operators --to disconnected-registry:5000/openshift/community-operators-catalog:4.2.1 
pulling quay.io/community-operators/etcd...
buliding catalog...
mirroring catalog...
quay.io/community-operators mirrored to disconnected-registry:5000/openshift/community-operators-catalog:4.2.1

$ docker pull disconnected-registry:5000/openshift/community-operators-catalog:4.2.1

$ opm registry images --from=disconnected-registry:5000/openshift/community-operators-catalog:4.2.1 --to=disconnected-registry:5000/community-operators --manifests=./mirror-manifests | xargs oc image mirror

$ oc apply -f ./mirror-manifests
```

### Future Work

Appregistry support for catalogs will be deprecated by 4.4. When it is deprecated, catalogs will be shipped as catalog images similar to those built by `opm registry build`. This means that `opm registry images` will continue to be used for disconnected installs, and after a support window, support for `--appregistry-*` flags can be droped.

`opm` will provide utilities for adding/removing packages from a catalog database. This utilities can be used for modifying a database before mirroring, so that unwanted packages and images are not mirrored.

The mechanism by which we specify related images is subject to change. Adding this data to the `ClusterServiceVersion` spec is necessary, because we are targeting 4.1-4.3 clusters using appregistry and options are limited. In the future it is likely that related images will be associated with the operators in a way that does not require modifying the manifests that represent the operator on-cluster.

