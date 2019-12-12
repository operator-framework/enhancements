---
title: olm-operators-can-integrate-with-openshift-monitoring
authors:
  - "@awgreene"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2019-12-11
last-updated: 2019-12-11
status: implementable
---

# olm-operators-can-integrate-with-openshift-monitoring

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

The primary purpose of this enhancement is to propose a set of changes that would enable [Operator-Lifecycle-Manager (OLM)](https://github.com/operator-framework/operator-lifecycle-manager) managed operators to define `ServiceMonitors` and `PrometheusRule` objects to interact with the [Prometheus Operator](https://github.com/coreos/prometheus-operator). This feature could then be used on OpenShift clusters to expose metrics to [Telemeter](https://github.com/openshift/telemeter).

For the uninitiated, Red Hat uses metrics collected by Telemeter to:

- Proactively reach out to customers with "problem clusters"
- Troubleshoot customer issues
- Understand how individual applications are behaving at a wholistic level

## Motivation

OLM managed operators would significantly benefit from the ability to interact with Prometheus Operators deployed on cluster. By presenting OLM managed operators with the ability to seamlessly integrate with the Prometheus Operator, Cluster Admins would be given a single portal to gauge the health of operators installed on their clusters.

This feature present unique value on OpenShift Clusters, where Telemeter could provide Red Hat with the ability to identify and proactively fix customer issues caused by Red Hat owned operators.

### Goals

- Provide operators with a first-class way of creating `ServiceMonitors` objects
- Provide operators with a first-class way of creating `PrometheusRules` objects
- Provide operators with a first-class way of specifying a namespace that they must be deployed to

### Non-Goals

- Provide generic metrics about OLM managed operators

## Proposal

### OLM support for new object in bundles

The bundle object currently has the following format:

```bash
$ tree
bundle
    ├── manifests
    │   ├── example.crd.yaml
    │   ├── example.csv.yaml
    │   └── example.rbac.yaml
    └── metadata
        └── annotations.yaml
```

Notice that OLM understands how to create and manage CSVs, CRDs, and RBAC files. This enhancement proposes expanding this feature to support `Namespaces`, `ServiceMonitors` and `PrometheusRules` objects:

```bash
$ tree
bundle
    ├── manifests
    │   ├── example.crd.yaml
    │   ├── example.csv.yaml
    │   ├── example.ns.yaml
    │   ├── example.prometheusrule.yaml
    │   ├── example.rbac.yaml
    │   └── example.servicemonitor.yaml
    └── metadata
        └── annotations.yaml
```

With this change, OLM will need to be updated to manage the `Namespace`, `ServiceMonitor` and `PrometheusRule` objects through the lifecycle of an operator.

#### Namespace Support

As OLM gains support for `Namespace` objects, The `ClusterServiceVersion` will be updated to include an optional `NamespaceRequirements` object field. The `NamespaceRequirements` will look like the following:

```golang
type NamespaceRequirements struct {
  //+optional
  // name represents where the operator must be deployed.
  name string

  //+optional
  // labels are labels that must be present on the given namespace.
  labels []string
}
```

>Note: If `NamespaceRequirements.namespace` is set, the UI will only show the specified namespace as an install option.

#### ServiceMonitor and PrometheusRule Support

As OLM gains support for these Prometheus objects, there is a chance that installed operators may overload the Prometheus instance. It is important that a Cluster Admin is alerted that an operator will report metrics prior to deploying the operator. As such, we propose adding a `monitoringSupported` boolean field to the `ClusterServiceVersion` object. OLM will not install either of the Prometheus object unless the `monitoringSupported` field is set to true. Additionally, Cluster Admins will need to manually the operator's `InstallPlans` in the following scenarios:

- A fresh install for this operator is being created where this field is set to `true`
- An existing operator had this field set to `false` and is being upgraded to a new version that sets this field to `true`

> Note: The `InstallPlan` will include a helpful message for Cluster Admins if either of the situations above are encountered.

### OLM Telemeter Support

#### Telemeter Requirements

Once OLM is capable of managing `ServiceMonitor` and `PrometheusRule` objects along with the lifecycle of an operator, OLM must allow Red Hat owned operators to take advantage of Telemeter. Before moving forward, let's review the requirements that must be met for Telemeter to scrape an application's metrics:

1. The application must be deployed in a namespace that is both prefixed with `openshift-` and labeled with `openshift.io/cluster-monitoring=true`
2. A [ServiceMonitor object](https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md#related-resources) must be created in the namespace mentioned above and point to the application's metrics endpoint
3. A role/rolebinding must be created that provides the Prometheus Operator ServiceAccount with the appropriate RBAC priviledges to discover the `ServiceMonitors` and `PrometheusRules` objects in the newly created namespace.

Both requirements 1 and 2 are fully met by the changes proposed above leaving our team with the duty to fulfill requirement 3.

#### Fulfilling Requirement 3: Updating the Prometheus Operator ServiceAccount

In order for the Prometheus Operator to recognize take advantage of this new feature, its ServiceAccount must have the proper RBAC permissions to view events on the `ServiceMonitors` and `PrometheusRules` objects in the operator's namespace.

On upstream clusters, there is no way for OLM to identify which Prometheus Operator ServiceAccount should be granted the RBAC priviledges. As such, the cluster admin will need to update the correct Prometheus Operator's OperatorGroup to include the correct namespace.

On OpenShift cluster, OLM can simply update the default Prometheus instance with the correct RBAC priviledges when the `openshift.io/cluster-monitoring=true` label is present on the namespace.

#### Risks and Mitigations

It is possible that the Prometheus team will not allow OLM managed operators to use the default prometheus operator to record metrics. If they are concerned about OLM managed operators overloading the prometheus instance, one solution that addresses this concern includes creating a new operator that is responsible for adding the `openshift.io/cluster-monitoring=true` label to namespaces and for generating the Telemter RBAC. This operator would:

- Be deployed in the `openshift-monitoring` namespace
- Have a list of approved OLM managed operators that may report metrics to Telemeter
- Watch for events on namespaces
- If an event occurs on a namespace that starts with `openshift-` and the namespace only includes valid operators, add the `openshift.io/cluster-monitoring=true` annotation and create RBAC that allows the Telemeter service account to watch for events in the namespace.

Alternatively, we could only add the `openshift.io/cluster-monitoring=true` on operators that originated from the `redhat-operators` and `certified-operators` CatalogSources.

## Implementation History

- 2019/12/11: Proposal created
- 2019/12/12: Proposal updated based on feedback
