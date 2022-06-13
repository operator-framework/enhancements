---
title: cluster constraint and property
authors:
  - "@dinhxuanvu"
reviewers:
  - "@kevinrizza"
  - "@joelandford"
approvers:
  - "@kevinrizza"
creation-date: 2021-10-08
last-updated: 2020-10-11
status: provisional
---

# cluster-constraint-and-property

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

The dependency resolution in OLM needs to work with cluster constraints in order to effectively install operators that will work properly in the cluster at runtime. Plus, OLM needs to support cluster properties which reflects the runtime information that the cluster may provide and allows operators to depend on those properties if needed to ensure proper installation and operation.

## Motivation

### Existing dependency resolution

OLM currently only resolves constraints that are provided by the operators and the resolution outcome at runtime is solely based on the constraints being satisfied by other operators who provide the required properties. It is not possible at the moment for OLM to consider cluster information or properties during resolution or apply cluster constraints to the available operators to prevent or allow certain operators to be installed successfully.

### Cluster information

Operators can depend on certain information such as usable API(s) that are available in the cluster. It is important to surface those information so operators can determine at runtime if they will be able to install and work properly in the cluster. The cluster information/properties can be depended upon as a constraint for a particular operator. This dependency relationship will ensure only applicable operators can be installed in the cluster and prevent possible failed installation and potentially nonworking condition.

### Cluster restriction

The cluster itself can impose certain restriction to which operators can be installed in the cluster via the properties associated with the operators. For example, the cluster imposes a minumum kubernetes version all for installable operators. A constraint can be constructed for a specific cluster restriction and then will be resolved during dependency resolution to ensure only operators with corresponding property with a satisfiable value.

### Goals

- Provide the mechanism to provide cluster properties and constraints
- Expose cluster properties and constraints to users

### Non-Goals
- Determine which cluster properties/constraints are required on the cluster
- Provide mechanism to retrieve and/or set up cluster properties and constraints automatically
- Provide mechanism to have separate namespaced and cluster-scoped properties/constraints

## Proposal

A new constraint API provides an effective away for cluster administrators to create custom resources (CR) to contain cluster information such as cluster constraints and properties.

OLM resolver will retrieve the cluster constraints and properties from the CR(s) and the information can be used during resolution.

The cluster constraints and properties are presented as a virtual cluster-scoped operator that is installed in the namespace. For all incoming operators that are intended to run the namespace, they must satisfy all constraints that are associated with this cluster-scoped operator. At the same time, the installable operators can depend on the properties that are provided by this virtual operator.

### User Stories

#### Cluster constraints

As a cluster admin, I would like to specify constraints that are specified to the cluster and they that will be applied to all operators. As a result, only operators that satisfy these constraints to be installable in the cluster.

#### Cluster properties

As a cluster admin, I would like to provide cluster properties that are relevant to cluster availability and capabilities so that operator authors can depend on those properties if needed to ensure their operators are installable and functioning properly in the cluster.

#### Bundle properties and constraints

As an operator author, I would like to include constraints that rely on cluster properties so my operators in my bundles so that they can only be installed on suitable clusters.

I also would like to provide essential properties that are used in cluster constraints in my bundle so that they can be screened at runtime if the clusters have certain requirements that may impact my operators' operation.

### Design Details

#### Cluster information API

The cluster constraint API is constructed using CustomResourceDefinition (CRD):

```yaml=
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: constraints.operators.coreos.com
spec:
  group: operators.coreos.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                constraints:
                  type: string
                properties:
                  type: string
  scope: Cluster
  names:
    plural: constraints
    singular: constraint
    kind: constraint
    shortNames:
    - constraint
```

The CRD is cluster-scoped as the CR(s) are representing cluster information. The cluster administrators have RBAC permission to create CustomResource (CR) that will contain cluster constraints and properties. For example:

```yaml=
apiVersion: operators.coreos.com/v2
kind: Constraint
metadata:
  name: cluster-constraints
spec:
  constraints: '[{"type":"olm.constraint","value":{"evaluator":{"id":"cel"},"rule":"properties.exists(p, p.type == \"certified\")","message":"require to have certified property","action":{"id":"require"}}}]'
  properties: '[{"type":"olm.k8s","value":{"version":"1.21"}},{"type":"olm.label","value":{"label":"secured"}}]'
```

The CR's spec contains two fields: `constraints` and `properties`. The `constraints` field is a list of `olm.constraint` type (See [1](https://github.com/operator-framework/enhancements/pull/91)). Each constraint is specified using the following syntax:

```json=
{
   "type":"olm.constraint",
   "value":{
      "evaluator":{
         "id":"cel"
      },
      "rule": "properties.exists(p, p.type == \"certified\")",
      "message": "require to have certified property",
      "action":{
         "id":"require"
      }
   }
}
```

The `properties` field is a list of properties that are specific to the cluster at the runtime. Each property is specified using the following syntax:

```json=
{
   "type":"olm.k8sversion",
   "value":{
       "version": "1.20.1"
      }
   }
}
```

A acceptable property is required to have `type` and `value` fields. The value for `type` can be any string. The value for `value` field can be a single data type or a struct that contains nested data type.

The API will come with data verification to ensure the data provided in `constraints` and `properties` fields are syntactically acceptable.

#### Bundle property and constraint

At the moment, OLM allows the bundle to include properties via `properties.yaml` in `/metadata` directory of the bundle image. This continues to be the mechanism for operator authors to supply essential properties for their operators in the bundle.


For constraints, the `dependencies.yaml` in `/metadata` directory continues to the place for operator authors to include constraints which rely on cluster properties by using the [generic constraint API](https://github.com/operator-framework/enhancements/pull/91).

#### Cluster property/constraint observability

The new constraint API is intended for cluster administrator use. Only admins should have RBAC to create, delete and update the constraint CR(s). However, regular users should have read RBAC to access the information on the CR(s) in order to see what cluster properties that they can rely upen while building their operators.

### Risks and Mitigations

## Alternatives
