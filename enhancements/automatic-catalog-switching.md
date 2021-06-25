---
title: automatic-catalog-switching
authors:
  - "@jchunkins"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2021-06-21
last-updated: 2021-06-21
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
in whatever way they choose. These well known template names will be replaced by a lookup 
mechanism to discover its corresponding value. The current proposal concentrates on 
template names that correspond to portions of a version (following semver major, minor, and patch). 
There are other possible template values provided but version is the primary focus.
Note that all templates must be surrounded by `{` and `}`, contain no spaces, and be all lower case.

### required templates

- {kube_major_version}
  - example resolved value: 1
- {kube_minor_version}
  - example resolved value: 19
- {kube_patch_version}
  - example resolved value: 0
- {olm_major_version}
  - example resolved value: 0
- {olm_minor_version}
  - example resolved value: 18
- {olm_patch_version}
  - example resolved value: 1
- {platform_architecture}
  - example resolved value: x86_64

### suggested downstream templates (only available in downstream versions of OLM)

- {ocp_major_version}
  - example resolved value: 1
- {ocp_minor_version}
  - example resolved value: 1
- {ocp_patch_version}
  - example resolved value: 1
- {ocp_update_channels}
  - example resolved value: fast

### Upstream OLM

Consider the following example for upstream OLM, where a catalog can be switched out based on the Kubernetes version in use:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: dynamic-catalog
  namespace: olm
spec:
  sourceType: grpc
  image: quay.io/kube-release-v{kube_major_version}/catalog:v{kube_major_version}.{kube_minor_version}
  displayName: My always up2date Catalog
  publisher: Red Hat
```

The image reference would resolve to quay.io/kube-release-v1/catalog:v1.19 for a cluster running Kubernetes
1.19 and would render quay.io/kube-release-v1/catalog:v1.20 for Kubernetes 1.20. 

The Kubernetes version can be obtained from one of these sources:

- using Clientset from k8s.io/client-go/kubernetes

   ```
   serverVersion, err := clientset.Discovery().ServerVersion()
   ```

- the api server `/version` endpoint using the `gitVersion` field (e.g. v1.17.1+6af3663)

When the Kubernetes version (or other template value) is unobtainable, the CatalogSource status should be set to an unhealthy state
with a message such as `Cannot construct catalog image reference, variable "kube_major_version" couldn't be resolved`

Once a catalog image has been resolved (even under error conditions), it should be added as a status condition as `resolved_image` so users know what value is
currently active.

### Downstream OLM

Consider the following example for downstream OLM where a catalog can be switched out based on the OCP version in use:


```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: dynamic-catalog
  namespace: olm
spec:
  sourceType: grpc
  image: quay.io/openshift-v{ocp_major_version}/catalog:v{ocp_major_version}.{ocp_minor_version}
  displayName: My always up2date Catalog
  publisher: Red Hat
```

In addition to supporting the upstream OLM templates, downstream OLM can also support its own templates.  

The image reference would resolve to quay.io/openshift-v4/catalog:v4.8 for OLM running on OCP 4.8 and would 
render quay.io/openshift-v4/catalog:v4.9 for OLM running on OCP 4.9. 

The OCP version can be obtained from:

- the `clusterversions.config.openshift.io` Custom Resource using the state parameter `.status.history[?(@.state=="Completed")].version`

When the Kubernetes or OCP version (or other template value) is unobtainable, the CatalogSource status should be set to an unhealthy state
with a message such as `Cannot construct catalog image reference, variable "ocp_major_version" couldn't be resolved`

Once a catalog image has been resolved (even under error conditions), it should be added as a status condition as `resolved_image` so users know what value is
currently active.

### User Stories [optional]

#### Story 1

As a cluster admin, I want to upgrade my cluster and seamlessly upgrade my third party catalog sources as well as the operators within the catalog
upon completing a cluster upgrade

#### Story 1

As a third party catalog creator, I want to provide customized catalogs that conform to the platform version and have the cluster use them automatically

### Implementation Details/Notes/Constraints

The CatalogSource will require pre-configuration with the appropriate templates to make this feature work. 


### Risks and Mitigations

- Misconfiguration or inability to fetch the version will put the CatalogSource into an unhealthy state.
- {platform_architecture} may not be obtainable in a heterogeneous cluster

## Design Details

### Test Plan

- Test image resolution by checking status updates to see if the template value matches the cluster conditions
- Test cluster upgrade to ensure that OLM operator upgrades the catalog (and therefore the third party operator)

### Acceptance Criteria

- dynamic catalog references enabled by variables in the catalog image URL spec
- upstream supported variables include:
  - Kubernetes version (major, minor, patch, e.g. 1.22.0), 
  - OLM version (major, minor, e.g. 0.18.1)
  - Platform architecture (e.g. x86_64)
- downstream supported variables include:
  - OCP version (major, minor, patch, e.g 4.9.0)
  - OCP update channels (e.g. fast)
  
- usage of unresolvable variables in the catalog image URL spec causes the catalog operator to stop reconciliation of the CatalogSource and set it into a human-readable error condition
- Any change in the resolved / rendered image URL spec leads to a catalog update (switch to the new image) as well as status updates to indicate current image in use


### Upgrade / Downgrade Strategy

- Use of template values (or removal of template values) are governed by the creator/maintainer of the CatalogSource. 
The effect of making the change is immediate (or at least as quickly as the reconcile logic can make it). 
- If the OLM operator is downgraded to a version that no longer supports templates, then the CatalogSource will by definition be
broken because it will never be a able to resolve the image reference.


### Version Skew Strategy

- The changes here should be self contained within the OLM operator and how it handles the CatalogSource during its reconcile cycle.

## Implementation History

- 2021-06-25 - initial proposal

## Drawbacks

- adds complexity to catalog image resolution

## Alternatives

- manual image reference changes could be made by cluster administrator upon cluster upgrade

## Infrastructure Needed

- proper automated testing would require a cluster upgrade which may not be feasible

