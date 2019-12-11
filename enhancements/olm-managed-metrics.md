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

The primary purpose of this enhancement is to propose a set of changes that would enable [Operator-Lifecycle-Manager (OLM)](https://github.com/operator-framework/operator-lifecycle-manager) managed operators to define `ServiceMonitors` and `PrometheusRule` objects to interact with the [Prometheus operator](https://github.com/coreos/prometheus-operator). 

This same feature could then be leveraged by Red Hat's OLM managed operators to expose metrics to [Telemeter](https://github.com/openshift/telemeter).

For the uninitiated, Red Hat uses metrics collected by Telemeter to:
* Proactively reach out to customers with "problem clusters"
* Troubleshoot customer issues
* Understand how individual applications are behaving at a wholistic level

## Motivation

OLM managed operators would significantly benefit from the ability to interact with Prometheus operators deployed on cluster. In particular, it is very difficult for Red Hat OLM managed operators to report metrics to Telemeter, which places these teams at a severe disadvantage when diagnosing customer issues.

It is imparative that OLM provide these teams with the means to interact with Prometheus operators on cluster and by extension report metrics to Telemeter.

### Goals

* Provide operators with the means to expose metrics to prometheus via `ServiceMonitors`
* Provide operators with the means to create `PrometheusRules`
* Ensure that only approved Red Hat operators are scraped by Telemeter

### Non-Goals

* Provide generic metrics about OLM managed operators

## Proposal

### Bundle ServiceMonitor and PrometheusRule Objects with a CSV

The bundle object should be updated to support Prometheus `ServiceMonitor` and `PrometheusRule` objects as it does with RBAC. When these objects are packaged with a CSV, OLM will be updated to deploy them along with the operator.

### OLM Telemeter Support

#### Telemeter Requirements

Once OLM supports bundling `ServiceMonitor` and `PrometheusRule` objects with the CSV, OLM must enable Red Hat owned operators to take advantage of Telemeter. Before moving forward, let's review the requirements that must be met for Telemeter to scrape an application's metrics:
1. The application must be deployed in a namespace that is both prefixed with `openshift-` and labbeled with`openshift.io/cluster-monitoring=true`
2. A [ServiceMonitor object](https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md#related-resources) must be created in the namespace mentioned above and point to the application's metrics endpoint
3. A role/rolebinding must be created that provides the prometheus service account with appropriate RBAC priviledges to discover the ServiceMonitor object in the newly created namespace.

Requirement 2 is fully met in [#Bundle ServiceMonitor/PrometheusRule Objects with a CSV](#bundle-serviceMonitor-prometheusRule-objects-with-a-csv), leaving our team with the duty to fulfill requirements 1 and 3.

#### Allow operator author to define a specific namespace

The first step towards allowing OLM managed operators to expose metrics to Telemeter involves deploying the operator in a namespace defined by the operator author. This can be accomplished by updating the `StrategyDetailsDeployment` type to include an optional field called `namespace` that defines where the operator's namespaced resources must be deployed. 

#### Creating RBAC and applying the `openshift.io/cluster-monitoring=true` label

We had considered creating any namespaces that begin with `openshift-` to include the `openshift.io/cluster-monitoring=true` label. However, this could be abused by external operators to report metrics to telemeter, potentially overloading the Telemeter prometheus instance. 

To protect Telemeter, we propose creating a new operator that is responsible for adding the `openshift.io/cluster-monitoring=true` label to namespaces and for generating the Telemter RBAC. This operator would:

* Be deployed in the `openshift-monitoring` namespace
* Have a list of approved OLM managed operators that may report metrics to Telemeter
* Watches for Events on namespaces
* If an event occurs on a namespace that starts with `openshift-` and the namespace only includes valid operators, add the `openshift.io/cluster-monitoring=true` annotation and create RBAC that allows the Telemeter service account to watch for events in the namespace.

## Implementation History

* 2019/12/11: Proposal created