---
title: olm-admission-webhook-support
authors:
  - "@awgreene"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-03-12
last-updated: 2020-03-12
status: implementable
---

# OLM Admission Webhook Support

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

This enhancement covers the necessary steps to add the capability for OLM to support [validating](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook) and [mutating](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook) admission webhooks.

## Motivation

More and more Operators are shipping webhooks to deliver on core parts of their functionality. Today, operators shipped with OLM are required to manually create and manage their webhooks independently of OLM due to lack of support. In this process, each operator re-creates the logic to implement a webhook and the logic to rotate and manage the coresponding certificates. OLM can help automate much of this process and provide a first-class experience to users, further strengthening OLM's value proposition.

### Goals

- Operator authors can include validating and mutating admission webhooks in their bundle as a part of a CSV.
- Operator author will not need to manage cert rotation for webhooks shipped with their operators.
- Via the SubscriptionConfig field in the Subscription, Cluster Admins will be able to integrate with tools like [service-ca operator](https://github.com/openshift/service-ca-operator) or [cert-manager](https://github.com/jetstack/cert-manager) projects.

### Non-Goals

- OLM will not provide support for conversion webhooks at this time.
- OLM will not support off-cluster webhooks.
- OLM will not provide a method to disable OLM Certificate Management across the cluster.
- OLM Webhook Support will not work out of the Box with the [CSVless Bundle Proposal](https://github.com/operator-framework/enhancements/pull/8).

## Proposal

This proposal outlines a plan that enables OLM to support operators that include validating or mutating admission webhooks as a part of the operator bundle. When this plan is implemented, operator authors will be able to:

- Ship their operator with a validating or mutating webhook without implementing logic to create or manage cert information as OLM will take care of that requirement instead.

Likewise, Cluster Admins will be able to:

- Disable OLM management of certificates and outsource this to a tool such as the service-ca controller or cert-manager.

Before we discuss cert generation or rotation, let's do a quick review of resources created as a part of a Webhook:

- Webhook
- Service
- Deployment
- RBAC
- Certs (Present in the Webhook and Deployment)

Currently, operator authors can already define the Deployment and RBAC resources in the CSV. Furthermore, OLM could be updated to generate the service required to forward requests intercepted by the webhook to the appropriate deployment and could manage on-cluster cert rotation. This leaves the operator author with responsibility to define the webhook itself within the CSV.

In order to support both validating and mutating webhooks OLM will need to provide operator authors with the ability to:

- Specify all fields in the [ValidatingWebhook struct](https://github.com/kubernetes/api/blob/b3bd583303d6e0723bbd276c4b00d1b65c1ff8db/admissionregistration/v1/types.go#L173-L300)
- Specify all fields in the [MutatingWebhook struct](https://github.com/kubernetes/api/blob/b3bd583303d6e0723bbd276c4b00d1b65c1ff8db/admissionregistration/v1/types.go#L302-L447)

Close examination of the `ValidatingWebhook` and `MutatingWebhook` structs reveals that the `MutatingWebhook` struct contains all fields featured in the `ValidatingWebhook` struct and an additional `ReinvocationPolicy` field. As such, the `CSV` struct should be updated to provide users with the ability to define all fields in a `MutatingWebhook` struct.

### CSV support for Validating and Mutating Webhook

CSV support for webhooks will begin with the introduction of the `Webhook` and `WebhookDescription` structs. These structures provides OLM with sufficient information to create either mutating or validating admission webhooks. These structs should look similar to the following examples:

```go
// Webhook is an exact replica of the MutatingWebhook struct with the exception that the WebhookClientConfig and NamespaceSelector fields are missing, as OLM will generate these resource itself.
// This struct MUST contain all fields that are present in the ValidataingWebhook and MutatingWebhook structs.
type Webhook struct {
	Name                    string                                          `json:"name"`
	Rules                   []admissionregistrationv1.RuleWithOperations    `json:"rules"`
	FailurePolicy           *admissionregistrationv1.FailurePolicyType      `json:"failurePolicy,omitempty"`
	MatchPolicy             *admissionregistrationv1.MatchPolicyType        `json:"matchPolicy,omitempty"`
	ObjectSelector          *metav1.LabelSelector                           `json:"objectSelector,omitempty"`
	SideEffects             *admissionregistrationv1.SideEffectClass        `json:"sideEffects"`
	TimeoutSeconds          *int32                                          `json:"timeoutSeconds,omitempty"`
	AdmissionReviewVersions []string                                        `json:"admissionReviewVersions"`
	ReinvocationPolicy      *admissionregistrationv1.ReinvocationPolicyType `json:"reinvocationPolicy,omitempty"`
}

// WebhookDescription is used to provide OLM with information about the Webhook being shipped with the operator
type WebhookDescription struct {
	// Type is used to identify the admission webhook as a validating or mutating webhook. Accepted values include `validating` and `mutating`
	Type                    WebhookAdmissionType                            `json:"type"`
	// DeploymentName must match one of the deployments included in the CSV.
	DeploymentName          string                                          `json:"deploymentName,omitempty"`
	// ContainerPort specifies which port the service should forward requests to.
	ContainerPort           int32                                           `json:"containerPort,omitempty"`
	// Webhook contains the information that must be sent to the validating or mutating webhook. 
	Webhook                 Webhook                                      `json:"webhook"`
}
```

It important to note that the Webhook structure does not provide users with the ability to set the WebhookClientConfig and NamespaceSelctor fields. Both of these fields will be generated via OLM. In detail:

- OLM will create the WebhookConfigClient after generating the Service and the CA-Cert.
- The NamespaceSelector field will be generated based on the OperatorGroup that the operator is deployed in. When creating the OperatorGroup, OLM will apply a label to all namespaces that are included in that operator group. Then, when creating the Webhook, OLM will update the namespace selector to use that label.

It is also important to call out that OLM will not allow webhooks to define rules that allows the webhook to intercept actions against CSVs and webhooks. Consider the worst case scenario where a broken webhook intercepts all calls against any resource and has a `Fail` [FailurePolicy](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#failure-policy). In this scenario, all calls to the K8s API would be rejected. As such OLM will implement constraints that prevent webhooks from including rules that include webhooks. Furthermore, if a webhook includes a rule that allows it to intercept calls to delete a CSV, it could effectively prevent a cluster admin from disabling the webhook described earlier, as OLM would attempt to reinstall the deleted webhook. As such, OLM will not allow webhooks to include rules that allows the webhook to intercept API requests targeting CSVs.

### Securing webhook via certificates

The certificate creation and lifecycle management is planned to utilize the same certs package that is already in OLM managing the certificates for api services. OLM has tested this feature extensively at this point and could easily port it to webhooks.

### Integration with Cert Management tools

There is a great deal of community support for integration with certificate management projects such as the [service-ca operator](https://github.com/openshift/service-ca-operator) or [cert-manager](https://github.com/jetstack/cert-manager). Cluster Admins should be given the opportunity to integrate with either of these tools be disabling the OLM management of certs and adding appropriate annotations to the deployment and webhook resources via the [SubscriptionConfig resource](https://github.com/operator-framework/operator-lifecycle-manager/blob/8985872bfd5888c8d7d49e4dfc9be162a87691e7/pkg/api/apis/operators/v1alpha1/subscription_types.go#L41-L87).

### Deployment Installation Order

There are a number of customer usecases in which the rollout order of deployments matters. Consider the following scenarios:

- An operator may be written to assume that any CR it reconciles has been acted upon by a webhook defined in its CSV. In this situation, OLM should provide the Operator Author with the ability to install the admission webhook before the operator.
- An operator may be written such that it expects to perform some action prior to starting the webhook. This logic may involve creating a CR that has special values that would be changed by the webhook defined in the CSV. In this situation, OLM should provide the Operator Author with the ability to install the operator prior to the webhook.

Both of the usecases described above can be addressed by updating OLM to:

- Create Deployments defined in the CSV in order.
- When creating the deployments defined in the CSV:
  - Waiting for the Deployment to reach the `READY` state.
  - Checking if the deployment owns any webhook or APIServices resources and if so, create the required resource.

### Webhook Upgrades

OLM must support the addition, subtraction, and upgrade of webhooks between versions of a CSV:

- If an admission webhook is removed in a new version of the CSV, the on cluster Webhook should be deleted.
- If a new webhook is added in a new version of the CSV, the webhook should be deployed.
- If the webhook is updated in a new version of the CSV, the existing webhook should be disabled before upgrading the operator.

### Risks and Mitigations

- Webhooks allow operators to intercept core resource and could wreak havoc on clusters if incorrectly configured. As such, users should be notified when an operator bundle includes a webhook.

- It is possible that a CR could be created between the time that a CRD is introduced and the time that the webhook, designed to validate or mutate those CRs, is created. The Operator Author should have some logic to validate that the webhook has intercepted the resource before reconciling it.

- When upgrading a CSV that includes a webhook, it is possible that the webhook could prevent the operator from being upgraded successfully. As such, OLM will disable webhooks associated with a CSV when performing an update. This can introduce a window where a CR can be created that is not intercepted by the webhook defined in the CSV.

- A webhook could be implemented that prevents the cluster from deleting the webhook. As such, OLM will not allow any webhook to intercept delete calls to an operator. Furthermore, if a webhook intercepts delete calls to a CSV, it could effectively prevent a user from disabling the webhook described above. As such, OLM will not allow Webhooks to intercept Delete calls to CSVs.

## Design Details

### Graduation Criteria / Deprecation Plan

Not applicable, this feature will be introduced as a part of the `ClusterServiceVersion` resource

### Upgrade / Downgrade Strategy

When upgrading or downgrading an operator via the CSV, a Webhook will never be removed but the resources that were created to support the webhook will. These resources will be left behind in an attempt to prevent cluster state from being lost.

### Version Skew Strategy

Not applicable.

## Implementation History

- Proposal 03/12
