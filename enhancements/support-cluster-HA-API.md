---
title: support-cluster-HA-API
authors:
  - "@varshaprasad96"
reviewers:
  - "@jmrodri"
  - "@ecordell"
  - "@estroz"
approvers:
  - "@jmrodri"
  - "@ecordell"
  - "@estroz"
creation-date: 2021-02-01
status: implementable
---

# Support for Cluster High-availability Mode API in Operator-lib

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

The goal of this enhancement is to describe the support for [cluster high-availability mode API][enhancement_cluster_operators] introduced by OpenShift. Operator-lib will contain the necessary helper functions which will enable users to identify if the OpenShift cluster can support high availability deployment mode of their operator or not.

## Motivation

OpenShift intends to provide an API to expose the high-availability expectations to operators in a cluster. The API will accomplish this by embedding additional fields in the [informational resource][informational_resource]. Operator-lib will contain the necessary helper functions which utilize the OpenShift API and provide users with the ability to identify if the cluster can support high-availability deployment of their operator or not. Operator authors can leverage this functionality to obtain information about the cluster topology and have their operators react accordingly.

### Goals:

- List the helper functions which will be provided in operator-lib that read the OpenShift informational resource and return appropriate values.

### Non-Goals:

- Discuss how OpenShift determines if the cluster can support HA/non-HA mode.
- Discuss the behavior of operators in HA/non-HA mode.

### User Stories

#### Story 1

As an Operator author, I would like to know if the cluster can support HA or non-HA modes using a function that can work in any Kubernetes cluster, so that I can configure my operator's behavior accordingly.

## Design Details

As an operator author it is helpful to know the cluster topology so that one can modify the behavior of operands. Operator-lib will provide users with a `GetClusterConfig` function that returns a `ClusterConfig` struct.

The `ClusterConfig` struct can be defined to be:

```go
import (
    ...
    apiv1 "github.com/openshift/api/config/v1"
    ...
)

// ClusterConfig contains the configuration information of the
// cluster.
type ClusterConfig struct {
    // ControlPlaneTopology contains the expectations for operands that 
    // run on control plane nodes.
    ControlPlaneTopology apiv1.TopologyMode

    // InfrastructureTopology contains the expectations for infrastructre
    // services.
    InfrastructureTopology apiv1.TopologyMode
}
```

The `ClusterConfig` contains the high availability expectations for control plane and infrastructure nodes. The `apiv1.TopologyMode` is defined by OpenShift API and refers to the [topology mode][api_topology_mode_pr] in which the cluster is running. It can have the following two values:

1. `ControlPlaneTopology` - refers to the expectations for operands that normally run on [control plane][control_plane] (or master) nodes.
2. `InfrastructureTopology` - refers to the expectations for infrastructure services that do not run on control plane nodes, usually indicated by a node selector for a `role` value other than `master`.

The possible values for both can be either `HighlyAvailable` or `SingleReplica`. More details on when the cluster is considered to support highly available or single replica deployments can be found [here][enhancement_cluster_operators].

**Note**
In future, the `ClusterConfig` can be extended to store additional information on cluster topology. 

Operator-lib will provide a helper function which will return the `ClusterConfig` mentioned above by reading the `infrastructure.config.openshift.io` crd present in cluster. The user will be expected to provide the location of the `kubeconfig` file, with which an existing  cluster can be accessed using the [Go client][upstream_go_client].

```go
// ErrAPINoAvailable is the error returned when the infrastructure CRD
// is not found in cluster. This means that this is a non-openshift cluster.
var ErrAPINoAvailable = fmt.Errorf("infrastructure crd cannot be found in cluster")

// GetClusterConfig will return the HA expectations of the cluster
func GetClusterConfig(kubeconfigPath string) (ClusterConfig, error) {
    // Implementation will involve reading the required fields
    // in status subresource of `infrastructure.config.openshift.io`
    // crd. If the crd is not present this will return an 
    // `ErrAPINoAvailable` error.
}
```

**NOTE:**
Operators cannot modify the informational resources present in OpenShift clusters. The function can only be used to retrieve information about the high availability expectations of the cluster and not to modify it.

### Scenarios:

1. Operator author tries to fetch the `ClusterConfig` for a non-openshift cluster:

In this case, an `ErrAPINoAvailable` error is returned. The error can be wrapped, such that users can utilize the `errors.Is` method to identify this specific case.

2. Operator author tries to use this function with older versions of OCP clusters which do not have HA expectations embedded in CRD:

In this case, the error returned will be in the same format as stated in previous scenario but would specifically mention that the requested information resource cannot be found in CRD.

### Restrictions of this library:

1. While `operator-sdk` will not scaffold this function in operators, the function is available to operator developers through operator-lib.

2. This function will always return an `ErrAPINoAvailable` error on non-OpenShift clusters.

[informational_resource]: https://docs.openshift.com/container-platform/4.6/installing/install_config/customizations.html#informational-resources_customizations
[control_plane]: https://docs.openshift.com/container-platform/4.1/architecture/control-plane.html#defining-masters_control-plane
[enhancement_cluster_operators]: https://github.com/openshift/enhancements/blob/master/enhancements/cluster-high-availability-mode-api.md#cluster-operators
[operator_lib]: https://github.com/operator-framework/operator-lib
[openshift-api]: https://github.com/openshift/api
[api_topology_mode_pr]: https://github.com/openshift/api/pull/827
[upstream_go_client]: https://kubernetes.io/docs/tasks/administer-cluster/access-cluster-api/#go-client