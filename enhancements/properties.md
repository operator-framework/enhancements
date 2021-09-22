# Explicit Bundle Properties

A bundle `property` is a key/value pair that represents some aspect of a bundle in a consumer-agnostic way.

This enhancement is for providing a first-class way to provide explicit `properties` for a bundle.

## Status

![GitHub issue/pull request detail](https://img.shields.io/github/issues/detail/state/operator-framework/olm-docs/94?label=docs%20) ![GitHub issue/pull request detail](https://img.shields.io/github/issues/detail/state/operator-framework/operator-registry/569?label=operator-registry%20)

## Motivation

### Consumer-defined Semantics

A `property` has no semantic meaning within the context of a bundle or a catalog. This is an important design consideration that permits a stronger degree of forward and backward-compatiblity of bundle and catalog content than would otherwise be possible. 

Much like HTML allows identifiers for elements (`id`, `class`, etc) that are then independently interpreted by other consumers (javascript, css, etc), or XML with its `attribute=value` pairs, `properties` get their meaning neither from the bundle schema nor the catalog version they happen to be included in.

A similar approach within operator-framework has been taken with olm's spec and status `descriptors`, which annotate fields of spec/status with opaque tokens to be interpreted by consumers. The primary consumer of descriptors is the OpenShift console to display custom UI for new APIs defined by CRDs, but the mechanism is designed to allow other consumers to interpret descriptors as desired: a CLI tool may use some of the information for generating prompts, or use more of it to provide a full console UI.

More examples of the pattern may be found [here](https://en.wikipedia.org/wiki/Attribute%E2%80%93value_pair).

### The resolver consumer

One major consumer of bundle properties is OLM, which reads them to determine the surface that is exposed to the dependency resolver.

The resolver and the registry apis already deal with certain aspects of a bundle via opaque `properties` via the `ListBundles` grpc API:

```proto
message Property{
  string type = 1;
  string value = 2;
}

message Bundle{
    ... 
	repeated Property properties = 12;
    ...
}
```

Many of the other bundle fields are replicated as properties. Here are the properties for an example bundle, rendered as psuedo-`json`:

```json5
{
    "type":  "olm.package",
    "value": `{"packageName":"etcd","version":"0.9.2"}`,
},
{
    "type":  "olm.gvk",
    "value": `{"group":"etcd.database.coreos.com","kind":"EtcdCluster","version":"v1beta2"}`,
},
{
    "type":  "olm.gvk",
    "value": `{"group":"etcd.database.coreos.com","kind":"EtcdRestore","version":"v1beta2"}`,
},
{
    "type":  "olm.gvk",
    "value": `{"group":"etcd.database.coreos.com","kind":"EtcdBackup","version":"v1beta2"}`,
},
{
    "type":  "olm.label",
    "value": `{"label":"testlabel"}`,
}
```

Properties intentionally closely align with the format for [`dependencies.yaml`](https://github.com/operator-framework/operator-registry/blob/master/docs/design/operator-bundle.md#bundle-dependencies).

Today, all properties are derived from other fields in a bundle. There is no way to write a new property `type` for a consumer (say, a newer version of the resolver, or an external tool that processes sets of bundles).

### Other consumers

Catalog and Bundles are consumed in many other contexts than the resolver. The package-server component presents catalog content for the purpose of selecting packages and channels, for example. 

### Goals

- Define the format for explicitly adding properties to a bundle.

## Proposal

A new file is accepted in the `metadata` folder for a bundle: `properties.(yaml|json)`

Here is an example of `properties.yaml`:

```yaml=
properties:
- type: olm.package
  value:
    packageName: prometheus
    version: "0.27.0"
- type: olm.gvk
  value:
    group: etcd.database.coreos.com
    kind: EtcdCluster
    version: v1beta2
- type: olm.label
  value: "LTS"
```

If properties are defined, they take precedence over the values derived from the bundle contents. In the above example, the `version` of the bundle will be considered `0.27.0` regardless of what the `ClusterServiceVersion`'s `spec.version` field indicates.

### Validate properties

Property validation is the domain of the consumer, not the producer. 

However, `opm` will ship with validation for all properties that the OLM resolver understands, and they will be validated as part of the `opm alpha bundle validate` command and the `operator-sdk` bundle validation. The validation will be implemented in `api` so that can be easily shared between both tools.

A future enhancement will discuss a formalized way to discover / publish validation rules for a given property `type`.

### Generate default properties

`opm` and `operator-sdk` will be capable of scaffolding the set of inferred properties from a given bundle. `opm` today infers a set of properties based on bundle content and stores them in the registry.

### Initial Resolver-Consumer Properties

This is a list of properties that the resolver consumer will read and understand.

- `olm.package`: The package name and bundle version. Used as a target of version range dependencies.
- `olm.gvk`: The GVK of a provided api. Used as the target of a required API dependency.
- `olm.channel`: The name of another bundle that this one replaces and a channel it replaces it in.
- `olm.skips`: A list of bundles that this one should skip
- `olm.skipRange`: A semver range specifier; any bundle that falls in this range can skip to the current one.
- `olm.label`: A string identifier for partitition bundles
- `olm.deprecated`: A bundle with this property will never be picked by the OLM resolver

See [declarative index config](https://github.com/operator-framework/enhancements/blob/master/enhancements/declarative-index-config.md) for more details of these initial properties and their schemas.

### Property Naming

Property types SHOULD be uniquely namespaced. 

For example, all properties that are supported by olm's resolver are prefixed with `olm.`, though this does not mean that the resolver component is the only valid consumer of those properties.

Failure to namespace may result in unintended interpretation by future or unknown consumers. 

In the future, a registry of known property types may be maintained in a central location, or a mechanism may be defined to discover property-type descriptions and definitions via a well-known interface.

### Version Skew Strategy

Property `types` may be versioned as needed / desired. For example, `olm.package.v2` may be introduced if the schema of `olm.package` needs to change.

A given consumer may or may not understand all `types` in a property set.
