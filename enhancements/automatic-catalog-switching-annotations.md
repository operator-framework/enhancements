---
title: automatic-catalog-switching-annotations
authors:
  - "@jchunkins"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2021-07-02
last-updated: 2021-07-02
status: implementable
see-also:
  - n/a
replaces:
  - n/a
superseded-by:
  - n/a
---

# Automatic Catalog Switching on Cluster Upgrades

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

- none at this time

## Summary

Operator compatibility with the underlying platform can be expressed in various ways, 
one of which is: catalogs that are specifically created for a particular platform release.

For example the cluster may undergo a transition from one of these releases:

- Kubernetes 1.19 vs Kubernetes 1.20
- OpenShift 4.5 vs. OpenShift 4.6

When this transition occurs, a catalog image may need to be changed to represent a 
compatible set of operators that will work properly in the new environment.

For on-board OCP Operators, the change of the catalog image is automatic because it 
is controlled by the Cluster Version Operator. Such control is missing for custom, 
third party catalogs. This proposal describes a simple approach to 
switching the catalog image based on platform updates.

## Motivation

All third party Operators have a similar requirement to automatically upgrade the Operator if 
the underlying platform gets upgraded, to ensure users are left with a compatible and
supported Operator install.

### Goals

- Enable automatic changing of the Operator catalog image as part of a platform upgrade in 
order to avoid the problem where cluster upgrades leave Operator installations in 
unsupported states or without a continued update path.
- Use a simple URL templating scheme to achieve 80% of the use use case seen today

### Non-Goals

- Using advanced solutions
  - a catalog of catalogs via declarative config that automatically determine which is the correct catalog
  - new APIs that stamp out CatalogSource objects
  - proxies at the registry side serving the catalog images

## Proposal

This enhancement should enable catalog image references to contain template values that can be replaced with
actual values from the underlying cluster. These templates should allow for flexibility so that an
image reference can be be composed in such a way that allows third parties to create image name / tag 
in whatever way they choose. A template can consist of either a well known name, or a Group / Version / Kind (GVK)
with a JSON Path. Regardless of template type, the template itself will be replaced by a value that is 
retrieved from the cluster. 

The current proposal concentrates on well known template names that correspond to portions of a version 
(following semver major, minor, and patch). There are other possible well known template values, 
but most have been discussed and rejected. Therefore the kubernetes version is the primary focus 
for well known templates in this proposal. 

Note that well known template names must be surrounded by `{` and `}`, contain no spaces, and be all lower case.

Optionally the GVK with JSON path template type consists of the following variables:

- `group` - the group in [DNS Subdomain](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-subdomain-names) format
- `version` - the version in [DNS Label](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-label-names) format
- `kind` - the singular kind in CamelCase format
- `name` - any [valid name reference](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#names) allowed by the kind and can be left blank if resource does not have a name
- `namespace` - the namespace in [DNS Label](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-label-names) format and can be left blank if resource is not namespaced
- `jsonpath` - a valid JSON Path value compatible with client-go syntax 

These variables allow the user to identify arbitrary values to use for template substitution by pinpointing a 
specific manifest by GVK, with a given name in a given namespace (if namespace scoped), and pull out any value within that manifest. 
All of these parameters MUST be provided, and the operator must have permissions to access the resource for 
this lookup to succeed. 

The GVK with JSON path variables MUST be provided in the exact order shown in the following example, where the 
variable name is followed by a colon and separated from the next variable by a comma:

```
{group:foo.example.com,version:v1,kind:Sample,name:MySample,namespace:ns,jsonpath:spec.foo.bar}
```

Failure to follow this syntax will result in a invalid template which will be ignored. 

Note that a GVK with JSON path template must be surrounded by `{` and `}`, contain no spaces, and follow the case sensitivity rules 
defined for each parameter above.

### Required well known templates

These templates are well known and can be resolved by any kubernetes distribution.

- {kube_major_version}
  - example resolved value: 1
- {kube_minor_version}
  - example resolved value: 19
- {kube_patch_version}
  - example resolved value: 0

### Optional GVK templates with JSON Path

- `{group:foo.example.com,version:v1,kind:Sample,name:MySample,namespace:ns,jsonpath:spec.foo.bar}`

### Templates that have been discussed and rejected

#### OLM version

The OLM version is not consistent between upstream and downstream and therefore unreliable

- {olm_major_version}
  - example resolved value: 0
- {olm_minor_version}
  - example resolved value: 18
- {olm_patch_version}
  - example resolved value: 1

#### Platform architecture

Architecture may not be consistent if running in a heterogeneous cluster environment

- {platform_architecture}
  - example resolved value: x86_64

#### Distribution specific

Declaring distribution specific variables does not scale well

- {ocp_major_version}
  - example resolved value: 1
- {ocp_minor_version}
  - example resolved value: 1
- {ocp_patch_version}
  - example resolved value: 1
- {ocp_update_channels}

#### Distribution version

Declaring generic distribution variables leaves gaps in support since there
are many distributions to consider and could lead to confusion when a 
particular distribution is not supported.

- {distro_major_version}
  - example resolved value: 1
- {distro_minor_version}
  - example resolved value: 1
- {distro_patch_version}
  - example resolved value: 1

### CatalogSource updates via annotations

The mechanism for declaring an image using templates involves an annotation called
`olm.catalogImageTemplate` whose value consists of the image reference with one or more templates included.
A controller (which can exist in either the OLM catalog operator or within a standalone operator) is 
responsible for watching `CatalogSource` instances looking for the `olm.catalogImageTemplate` annotation.
Upon identifying a candidate `CatalogSource` the template values can be resolved and assuming all values
are available, the `spec.image` of the `CatalogSource` is updated with the resolved value ONLY if the
resolved value and the `spec.image` values differ. Once the `spec.image` is updated the OLM catalog 
operator will handle the image update as if a user had changed the image value manually. 

Consider the following example, where a catalog can be switched out based on the Kubernetes version in use:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: dynamic-catalog
  namespace: olm
  annotations:
    olm.catalogImageTemplate: "quay.io/kube-release-v{kube_major_version}/catalog:v{kube_major_version}.{kube_minor_version}"
spec:
  sourceType: grpc
  image: quay.io/kube-release-v1/catalog:v1.20
  displayName: My always up2date Catalog
  publisher: Red Hat
```

The image reference would resolve to `quay.io/kube-release-v1/catalog:v1.19` for a cluster running Kubernetes
1.19 and would render `quay.io/kube-release-v1/catalog:v1.20` for Kubernetes 1.20. 

The Kubernetes version can be obtained from one of these sources:

- using Clientset from k8s.io/client-go/kubernetes

   ```
   serverVersion, err := clientset.Discovery().ServerVersion()
   ```

- the api server `/version` endpoint using the `gitVersion` field (e.g. v1.17.1+6af3663)

Consider another example which uses an arbitrary kubernetes manifest as a source for template substitution:

```yaml
apiVersion: foo.example.com/v1
kind: Sample
metadata:
  name: MySample
  namespace: ns
spec:
  foo:
    bar: v2
```

The catalog source references the `Sample` manifest above like this:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: dynamic-catalog
  namespace: olm
  annotations:
    olm.catalogImageTemplate: "quay.io/sample/catalog:{group:foo.example.com,version:v1,kind:Sample,name:MySample,namespace:ns,jsonpath:spec.foo.bar}"
spec:
  sourceType: grpc
  image: quay.io/sample/catalog:v2
  displayName: My always up2date Catalog
  publisher: Red Hat
```

The image reference would resolve to `quay.io/sample/catalog:v2`. 

Regardless of template type, when the template value is unobtainable, the CatalogSource status should include 
messages such as `Cannot construct catalog image reference, variable "kube_major_version" couldn't be resolved`

Example if one or more errors occur:

```yaml
status:
  olm.catalogImageTemplate:
    messages:
    - `Cannot construct catalog image reference, variable "{kube_major_version}" couldn't be resolved` 
      `Cannot construct catalog image reference, variable "{group:foo.example.com,version:v1,kind:Sample,name:MySample,namespace:ns,jsonpath:spec.foo.bar}" couldn't be resolved` 
    resolvedImage: "quay.io/sample{kube_major_version}/catalog:{group:foo.example.com,version:v1,kind:Sample,name:MySample,namespace:ns,jsonpath:spec.foo.bar}"   
```

Example of success:
```yaml
status:
  olm.catalogImageTemplate:
    messages:
    - `catalog image reference was successfully resolved` 
    resolvedImage: "quay.io/kube-release-v1/catalog:v1.20"   
```

If a user decides to no longer participate in automatic catalog switching, they can simply remove the annotation
from the `CatalogSource`.


### User Stories [optional]

#### Story 1

As a cluster admin, I want to upgrade my cluster and seamlessly upgrade my third party catalog sources as well as the operators within the catalog
upon completing a cluster upgrade

#### Story 1

As a third party catalog creator, I want to provide customized catalogs that conform to the platform version and have the cluster use them automatically

### Implementation Details/Notes/Constraints

The CatalogSource will require pre-configuration with the appropriate templates to make this feature work. 


### Risks and Mitigations

- Misconfiguration or inability to fetch a value for a given template will NOT update the CatalogSource image as might be expected, so 
the user must consult the status to determine what template has failed.

## Design Details

### Test Plan

- Test image resolution by checking status updates to see if the template value matches the cluster conditions
- Test cluster upgrade to ensure that OLM operator upgrades the catalog (and therefore the third party operator)

### Acceptance Criteria

- dynamic catalog references enabled by variables in the annotations for a CatalogSource, which results 
an update of the catalog image reference
- supported variables include:
  - Kubernetes version (major, minor, patch, e.g. 1.22.0), 
- optional variables include
  - GVK with JSON path
- usage of unresolvable variables in the annotation of a CatalogSource causes the status section to be updated
with human-readable failure reasons and the catalog image URL is NOT updated. The status section should also
indicate the currently resolved image for reference.
- Any change in the resolved / rendered image URL defined in the annotation leads to a catalog update
(i.e. a switch to the new image) as well as status updates to indicate current image 


### Upgrade / Downgrade Strategy

- Use of template values (or removal of template values) are governed by the creator/maintainer of the CatalogSource. 
The effect of making the change is immediate (or at least as quickly as the reconcile logic can make it). 
- If the OLM operator (or whatever operator contains the controller) is downgraded to a version that no
longer supports templates, then the current image value defined in the CatalogSource will still be in effect.


### Version Skew Strategy

- The changes here should be self contained within the operator and how it handles updates to the CatalogSource during its reconcile cycle.
The OLM operator logic as it exists today will not be changed.

## Implementation History

- 2021-07-02 - initial proposal

## Drawbacks

- adds complexity to catalog image resolution

## Alternatives

- manual image reference changes could be made by cluster administrator upon cluster upgrade

## Infrastructure Needed

- proper automated testing would require a cluster upgrade which may not be feasible
