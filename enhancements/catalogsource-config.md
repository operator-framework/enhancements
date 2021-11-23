---
title: catalogsource-config
authors:
  - "@perdasilva"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2021-11-18
last-updated: 2021-11-18
status: provisional
---

# catalogsource-config

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA


## Summary

Currently, there is no way to instruct OLM to schedule pods created from `CatalogSource` CRs on specific nodes.
This is desirable for admins as it gives them more control over where in their infrastructure the catalog source
pods will be created. In this enhancement proposal we
propose changes to the `CatalogSource` spec to enable this kind customization possible for `grpc` type CatalogSources 
that define a `spec.image` - the only catalog source configuration that produces a pod. 


## Motivation

This EP targets issue [#2452](https://github.com/operator-framework/operator-lifecycle-manager/issues/2452).
Users would like to be able to override the pod spec of the CatalogSource pod
created for `grpc` `CatalogSource`s. Specifically, they would like to be able to specify
pod tolerations and node selectors.

### Goals

 * At a minimum, provide a way for cluster admins to influence CatalogSource pod scheduling (add tolerations and node-selectors)
 * Generally, update the `CatalogSource` spec to allow cluster admins to affect the pod spec of `grpc` `CatalogSource` pods
 * Decide on which aspects of the pod spec can be overriden
 * Decide on whether to make breaking changes to the spec now, or later

### Non-Goals

 TBD

### Open Questions

I've put down the options as I see them. I need some input from the community to figure out the right way to go.
The first two would required breaking changes and therefore migration to a new schema. The [first option](#Union-types)
is better assuming we don't want to give users the option to implement controllers for additional, and custom,
`CatalogSource` types. Otherwise, the [seconds option](#CatalogSourceClass) would be better. If we have bigger changes on
the horizon, we may want to punt on the "right" approach, in favour of the [third option](#ImageConfig). This would 
mitigate the issue, but not necessarily in the best way.

I'm personally in favor of options 1 or 2, even though they will require more work, I think they provide an improvement
in user experience.

## Proposal

### Current State

The [current](https://github.com/operator-framework/api/blob/a80624eab36b2005fa5d84c9b539bb4817e44daa/crds/operators.coreos.com_catalogsources.yaml#L54) `CatalogSource.spec` accepts three difference values for the `spec.sourceType`: `grpc`, `configmap`, and `internal`. Depending on the value, different optional attributes are used to deploy the underlying infrastructure to serve the catalog content. For instance, if `spec.sourceType` is set to `grpc`, the user can set either the `spec.address` or `spec.image`. If both are set, `spec.image` takes precedence. If `spec.sourceType` is set to `configmap`,
`spec.configMap` gets used, but `spec.address` and `spec.image` get ignored. Since there could be attributes defined that just get ignored by the controller, it could be hard for a user to know what is happening given just a `CatalogSource` CR and no documentation.

### Approach Options

Two different options are outlined below. The first would require breaking changes to the API, while the second would be the "minimal changes" optional. The path taken likely depends on the whether we are planning for a new `CatalogSource` CRD version in the near future or not.

#### Union Types

We use a union type to define the type and configuration for the `CatalogSource`. This can be realised with the [`oneOf`](https://swagger.io/docs/specification/data-models/oneof-anyof-allof-not/) keyword supported in OpenAPI v3 CRD schemas. With `oneOf` the apiserver can validate that the correct fields were used in the right combination in the CR. The schema would then look something like:

```yaml
spec:
  type: object
  required:
    - sourceConfig
  properties:
    # Deprecate 
    # image: ... 
    # address: ... 
    # configMap: ...
    # sourceType: grpc | configMap | internal
    
    # Keep as is:
    description: ...
    displayName: ...
    icon: ...
    priority: ...
    publisher: ...
    secrets: ...

    # Add:
    # Note: 'internal' type is being deprecated so it gets no config structure
    grpcPodSource:
      podSpec:  # a subset (or the whole of) corev1.PodSpec
        image: ... # required
        # all other attributes are optional
        # and override the default catalog source pod spec
        ...
      ...
    grpcServiceSource:
      address: ...
      ...
    configMapSource:
      configMap: ...
      ... 
  oneOf:
    - required: ["grpcPodSource"]
    - required: ["grpcServiceSource"]
    - required: ["configMapSource"]
                  
``` 

This option introduces breaking changes. This means that while we are on the deprecation path, users will be able to configure their `CatalogSource`s in the current way, _OR_, use one of `grpcPodSource`, `grpcServiceSource`, or `configMapSource` (with precendence given to the new configuration). Once the fields haven deprecated a new CRD version would need to be created without the deleted fields.

**Pros**
  1. Each CatalogSource type has its own configuration structure
  2. We don't polute the top level with endless optional attributes that behave differently depending on the values of other attributes
  3. Better UX
  4. New catalog source types can be added easily


**Cons**
  1. Introduces breaking changes
  2. There are no kubebuilder annotations supporting `oneOf`. In order to make use of the apiserver for validation, we'd need to add it to the OpenApi v3 schema on the generated CRD. Additional automation will need to be added to include this validation in the schema.

#### grpcPodConfig

This option is the least invasive of the three. Here we add an additional optional attribute `grpcPodConfig` that contains the parts of the pod spec we'd like to override. For example:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: operatorhubio-catalog
  namespace: olm
spec:
  sourceType: grpc
  image: quay.io/operatorhubio/catalog:latest
  grpcPodConfig:
    nodeSelector:
        node-role.kubernetes.io/master: ""
    tolerations:
    - key: "node-role.kubernetes.io/master"
      operator: Exists
      effect: "NoSchedule"
  displayName: Community Operators
  publisher: OperatorHub.io
```

**Pros**
  1. Additive change, no need to migrate to a new schema

**Cons**
  1. We don't address the underlying flaw with the current schema

### Pod Spec Overrides

The next question to answer is which elements of the pod spec can be overriden. We cannot go down the path of allowing users
to override _any_ aspect of the pod spec. This would just give users a way to create arbitrary pods with OLM/Catalog Source controller
priviledges. 

However, at a minimum we need to surface everything in the [scheduling](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#scheduling) section of the pod spec (i.e. tolerations, affinity, nodeSelector, nodeName, etc.). Additional knobs could be added to affort the user to add additional annotations and labels to the pod (if this is seen as valuable).


To start with, I would suggest keeping the scope to only those scheduling related attributes in order to reduce the complexity and size of the change. Additive changes can always be easily made.

### Chosen Approach

TBD

### Risks and Mitigations

TBD

## Design Details

TBD

### Test Plan

TBD

### Graduation Criteria

TBD

### Upgrade / Downgrade Strategy

TBD

### Version Skew Strategy

TBD

## Implementation History

TBD

## Drawbacks

TBD

## Alternatives

TBD
