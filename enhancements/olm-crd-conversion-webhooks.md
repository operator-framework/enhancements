---
title: olm-crd-conversion-webhook-support
authors:
  - "@sdhaliwa"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-06-10
last-updated: 2020-07-05
status: implementable
---

# OLM CRD Conversion Webhook Support

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

This enhancement covers the necessary steps to add OLM support for [ CRD conversion](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/#webhook-conversion) webhooks.

## Definitions

- Webhook: Webhooks are HTTP callbacks, providing a way for notifications to be delivered to an external web server i.e. a simple event-notification via HTTP POST. Examples:  [ CRD conversion](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/#webhook-conversion) webhooks, [validating](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook) and [mutating](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook) admission webhooks.

- Conversion Webhook: Webhook that can convert an object from one version to another. 

  When API clients, like kubectl or your controller, request a particular version of your resource, the Kubernetes API server needs to return a result that’s of that version. However, that version might not match the version stored by the API server.

  In that case, the API server needs to know how to convert between the desired version and the stored version. Since the conversions aren’t built in for CRDs, the Kubernetes API server calls out to a webhook to do the conversion instead. 

## Motivation

Many Operators are shipping conversion webhooks to deliver conversion of an object from one version to another. The CustomResourceDefinition API supports a versions field that is used to support multiple versions of custom resources that have been developed. Versions can have different schemas and a conversion webhook is used to convert custom resources between versions.

OLM needs to support operators with crd conversion webhooks and help manage and rotate corresponding certificates.

### Goals

- Operator authors can include crd conversion webhooks in their bundle as a part of a CSV.
- Operator author will not need to manage [cert](https://www.ssl.com/faqs/what-is-a-certificate-authority/) rotation for webhooks shipped with their operators.

### Non-goals

- This enhancement aims to support AllNamespace/ Global Singleton Operators only.
- Specifying PodDisruptionBudgets with webhooks is out of scope for this enhancement. 

## Proposal

Operator authors will be able to:

- Provide a CRD name in the WebhookDescription, and this CRD would have the conversion webhook defined. OLM would update the CRD with correct CA cert.


### CSV support for CRD Conversion Webhook

- The WebhookDescription struct would include one extra field called ‘ConversionCrds’ in the WebhookDescription struct which would hold the CRD names. 

```
type WebhookDescription struct {
    .
    .
    .
    // ConversionCrd must match the CustomResourceDefinition name
    ConversionCRDs           []string              `json:"conversionCRDs"`
}
```

- Cluster admins will be allowed to install CRD conversion webhooks for only AllNamespace Operators.

  CRD conversion webhooks intercept requests for all namespaces, so only AllNamespace operators will ship with conversion webhooks and only those can be installed to leverage these webhooks. Once an AllNamespace operator is installed, it would claim ownership of the API in every namespace, and OLM will not allow installation of any other Operator that claims the same API.

- There can’t be multiple conversion webhooks on a single cluster for a given CRD.

- No new resources are added, the Webhook resource is extended to include conversion CRD name.

### Webhook Upgrades

- If the CRD conversion webhook is removed in a new version of the CSV, the on cluster Webhook should be deleted.

### Securing webhook via certificates

The certificate creation and lifecycle management is planned to utilize the same certs package that is already in OLM managing the certificates for [api services](https://github.com/operator-framework/operator-lifecycle-manager/blob/21d67c12beef9dd72a3fe3d09a976d5289a42f60/doc/design/building-your-csv.md#apiservice-resource-creation). 

### Deployment Installation Order

OLM would:

- Create deployments defined in CSV
  - While creating the deployment, OLM would identify that CRD(s) is(/are) conversion webhook CRD(s)
  - This conversion webhook CRD would be updated to include proper CA cert.  


