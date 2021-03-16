---
title: operatorhub-bundle-validation
authors:
  - "@camilamacedo86"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2021-02-19
last-updated: 2021-03-14
status: implementable
---

# Add new checks to the OperatorHub Validator

This proposal describes new checks to be implemented into the [OperatorHubValidator][operatorhubvalidator]. This validator is defined in [operator-framework/api][oper-api].

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

**1. Should we create a new Validator for this new checks ?** 

The new checks are valid in the context of what would be required and/or recommended to publish an operator bundle on OperatorHub.io. Also, we do not have many of them to starts to have big concerns which shows fail in the [PO](https://wiki.c2.com/?PrematureOptimization) in this stage.

However, would be expected following good practices where the types of checks are properly encapsulate and can easily keep maintained and extracted if required.

**2. Are these the ONLY checks to ensure that the operator bundles work well as expected?**

Definitely not. The operator bundle manifests has a lot of information which could be used by *"linters"* checks with the purpose of avoid issues scenarios and give a fast response for the operator authors and jobs.

This proposal only addresses some priority needs which are; 
- check and notify [channel naming convention][channel-names-doc] 
- check and notify operator name versioning
- avoid the issues with the most critical scenarios related to K8S API deprecations which will cause high impact after 1.22 release out.

**3. What are the changes proposed here which affect SDK?**

This proposal is to define new checks for a Validator which is used and provided by SDK already. Then, for users are able to use its new checks SDK requires only bump the version of [operator-framework/api][oper-api] and add a new option flag `--optional-values` to get and pass options values to the Validator.

Note that as described bellow the idea here is only add a flag which accepts a string map. SDK does not validate the values informed it will only pass this data obtained to the [operator-framework/api][oper-api]. See [here](https://github.com/operator-framework/operator-sdk/blob/v1.5.0/internal/cmd/operator-sdk/bundle/validate/optional.go#L114)

**Why this proposal suggested a string map for flag `--optional-values`?**

See that SDK allows we use it with any validator implemented in [operator-framework/api][oper-api]. So, we might want to have other optional values for others ones (e.g ocp=4.8). Then, SDK does not required have an flag option for each case. Also, not that we can call the `operator-sdk bundle validate` for a suite of tests which means that we could indeed in the future pass a list of key=values where each validator used would check if has or not a key valid for it. 

## Summary

`OperatorHubValidator` checks if the operator is respecting the required criteria to be published in the [OperatorHub.io](https://operatorhub.io/). This proposal describes criteria to be added to it. See [here](https://github.com/operator-framework/api/blob/v0.6.0/pkg/validation/internal/operatorhub.go) its checks code implementation.

Many tools and projects use `OperatorHubValidator` already. Note that the new checks described by this proposal will be available via SDK:

```sh
$ operator-sdk bundle validate ./bundle --select-optional name=operatorhub
``` 

The new validator also needs to allow the checks occurs to verify the operator bundle configuration for a specific Kubernetes version where it is intend to be published:

```sh
$ operator-sdk bundle validate ./bundle \
    --select-optional name=operatorhub \
    --optional-values="k8s=1.22"
```

**NOTES** For further information check [here](https://sdk.operatorframework.io/docs/olm-integration/generation/#validation) in the SDK docs and [here](https://olm.operatorframework.io/docs/tasks/validate-package/#validation) in the OLM docs. Also, the new checks will be required for audit purposes where a tool would be able to check all bundles of a provided OLM index catalogue. 

## Motivation

- Allow an operator developer or scheduled jobs a means to check required criteria for publishing an operator on OperatorHub.
- Validate the operator bundle manifests and notify users and maintainers with warning and errors about what is not compliant to the OperatorHub criteria
- Allow operator developers, CI jobs such as the pipeline/CVP, and admins of OLM catalogues audit and validate the bundles regarding the specific criteria for Kubernetes versions, which are supported by the bundle or which the user is intend to distribute the operator.

### Goals

- Perform static checks on the operator bundle manifests to avoid the scenarios where the operator bundle:
    - is published but will not be installable
    - is not prepared to work well on all Kubernetes cluster versions and lacks any configuration to address its limitation
    - is using deprecated or no longer supported K8S APIs
    - is not following good practices and conventions
- [operator-framework/api][oper-api] with new checks in `OperatorHubValidator`
- SDK tool using [operator-framework/api][oper-api] and return errors or warnings regarding the new checks detailed in this proposal via the command `operator-sdk bundle validate ./bundle --select-optional name=operatorhub` and allowing we pass a list of key=maps for the Validators via `--optional-values`
- Pipeline able to identify the scenarios covered by these new checks 

### Non-Goals

- Add checks which are specific for some kind of vendor
- Provide a way for the users check specific criteria or perform custom checks

## Proposal

The `1.22` release will bring a big affect to the Pipeline and to the OpenShift catalog. Note that   

The checks proposed in this documentation will allow the operator authors and admin catalogs identify all. 

### User Stories 

- As an operator developer, I'd like to use the SDK tool to validate my operator bundle before publishing on OperatorHub. Validating will allow me to quickly get its errors and warnings locally so that, I can provide the required fixes and improvements before trying to publish it.
- As an operator framework maintainer, I'd like to see the `OperatorHubValidator` in [operator-framework/api][oper-api] performing the proposed new checks so that, I can easily keep it maintained and use it in any project and tool which pleases me.
- As an operator framework maintainer, I'd like to see CI checking the operator bundle criteria when a user pushes a Pull Request to add the operator bundle on the OperatorHub so that, we can easily ensure a better quality of the projects before they are published.  
- As an operator developer or operator framework maintainer, I'd like to use the validator to check if my operator bundle will work on versions of Kubernetes that I am intend to publish it to.

### Implementation Details/Notes/Constraints

#### New Criteria valid which will not checked against to a specific Kubernetes version

**minKubeVersion**

The motivation for the following check is to ensure that operators authors understand that when the `minKubeVersion` is not defined it means allow to distribute the operator bundle in any Kubernetes version available and then, tha the operator projects has no limitations regarding the supported kubernetes versions which probably is not the common scenario.

**NOTE** By looking at the OpenShift catalogue `4.7` we can found `125` head operator bundles, and `41` of them provided the `minKubeVersion` version. It means that `84` operator bundles have been distributed as supporting all Kubernetes Versions, which has a significant potential of not being accurate.

| Check | Type | Description | 
| ------ | ----- | ----- |
| To check if the `minKubeVersion` is informed | warning | Check if `spec.minKubeVersion` in CSV has a value if not inform the user that it would be recommended 

The motivation for the following checks is only to ensure that the syntax informed is valid and avoid issues usually caused by a typo mistake.

| Check | Type | Description | 
| ------ | ----- | ----- |
| To check if the value defined in `minKubeVersion` is valid| error | Check if `spec.minKubeVersion` has an valid syntax and if it is a valid option. Otherwise, return an error with the information accordingly. For that we only need to check if the value is respecting the [Semantic Versioning](https://semver.org/). We can use the golang lib [semver](https://github.com/blang/semver) `semver.Parse(value)`. More info: [k8s docs](https://kubernetes.io/docs/setup/release/version-skew-policy/#supported-versions). 

**Checking recommendations and conventions**

The motivation for the following check is to ensure that operators authors knows that operator bundles names should follow a name convention with versioning. Also, it will check if the channels are or not respecting the [Naming Convention Rules][channel-names-doc]. However, the operator authors still able to choose the names as please them.

This check is also valid for any analytic tool that we might to provide where we would able to check all operator bundles from a catalog which are not following the convention as to starts to promote them along its users.

| Check | Type | Description | 
| ------ | ----- | ----- |
| To check if the operator bundle versioning convention is followed | warning | Check if the operator bundle name (`metadata.name`) in CSV matches with `operatorname-v0.0.0`. We can use regex and check if the value is respecting the [Semantic Versioning](https://semver.org/). We can use the golang lib [semver](https://github.com/blang/semver) `semver.Parse(value)`. Otherwise we should inform that it is not following the recommended versioning conventional.
| To check if the operator bundle channel names are following up the definition. See [here](https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/channel-naming.md#recommended-channel-naming) | warning | Check if the channels name are not in `preview`,`stable`,`fast` or in `preview-*`,`stable-*`,`fast-*`. We can use regex and raise the warning with the link for the doc. Also, the warning should return all values that does not match with criteria. See that the channel names can be found in the index image (e.g [here](https://github.com/operator-framework/operator-sdk/blob/v1.4.2/testdata/go/v2/memcached-operator/bundle/metadata/annotations.yaml#L2)) or in the package (e.g [here](https://github.com/operator-framework/community-operators/blob/master/upstream-community-operators/mysql/mysql.package.yaml#L1))

#### New Criteria valid which will be checked against to a specific Kubernetes version

For the checks which should be done accordingly for some specific version(s) it will use the Kubernetes version which has been provided to the validator via flag or via the bundle operator configuration. For example, if we run:

```sh
 operator-sdk bundle validate ./bundle \
     --select-optional name=operatorhub \
     --optional-values="k8s=1.22"
```

Then, it means that the check will validate the operator bundle configuration accordingly to the Kubernetes version informed where it is intend to be published to. In this case, the validator will consider the Kubernetes version `=1.22`. 

If the `--optional-values="k8s=1.22"` not be used then, we will use the `minKubeVersion` in the CSV to do the checks. So, the criteria is `>=minKubeVersion`. By last, if the `minKubeVersion` is not provided then, we should consider as described above that the operator bundle is intend to work well in any Kubernetes version.

**CRD and Webook APIS deprecated and unsupported versions**

Note that in these cases, users should be informed that the `apiextensions/v1beta1` and that the `admissionregistration.k8s.io/v1beta1` was deprecated in Kubernetes `1.16` and will/was removed in `1.22`.

- To check if the operator bundle is using a CRD API version which is deprecated or not supported we will need to see if the CRD manifests in the bundle are using `apiVersion: apiextensions.k8s.io/v1beta1`. (See [here](https://github.com/operator-framework/operator-sdk/blob/v1.4.2/testdata/go/v2/memcached-operator/bundle/manifests/cache.example.com_memcacheds.yaml#L1)). Note that an operator bundle can have Many CRD(s).

- To check if the operator bundle is using the Webhook API version which is deprecated we will need to see if the CSV has the spec `webhookdefinitions.<*>.v1beta1`. (e.g See [here](https://github.com/operator-framework/operator-sdk/blob/v1.4.2/testdata/go/v2/memcached-operator/bundle/manifests/memcached-operator.clusterserviceversion.yaml#L203-L205). Note that an operator bundle can have Many Webhook(s) and with more than one type and this field is a list of.

The error raised will be accordingly with the Kubernetes version that we have, see:

| Conditional |Type |
| ------ | ----- |
| operator bundle supports k8s => `1.22` and do not uses `apiextensions/v1` ÒR has [webhookdefinitions](https://github.com/operator-framework/operator-sdk/blob/v1.4.2/testdata/go/v2/memcached-operator/bundle/manifests/memcached-operator.clusterserviceversion.yaml#L203-L205) and do not have the API `v1` listed | error |
| operator bundle supports k8s `=> 1.16 <= 1.22` and uses `apiextensions/v1beta1` ÒR `admissionregistration.k8s.io/v1beta1` | warning |

**Why we need to check if the api v1 is not used instead of only check if it has any CRD or Webhook with the deprecated `v1beta1`?**
		
The operator author might provide a solution that works for both scenarios which means that will work when installed in the previous version of `Kubernetes < 1.16` and also in the upper versions `=> 1.22`. However, for the operator works well in any Kubernetes version `>=1.22`, we know that it is mandatory it has the latest APIs versions `v1`. See [here](https://github.com/kubernetes-sigs/kubebuilder/blob/v3.0.0-beta.0/test/e2e/v3/plugin_cluster_test.go#L99-L101), for example, that in the e2e test for Kubebuilder, we check what is the cluster version to apply and test the API versions accordingly. And then, note that we can find both APIs in the webhooks fields of the CSV as well such as:

```
  webhookdefinitions:
  - admissionReviewVersions:
    - v1beta1
    - v1   
``` 

**IMPORTANT** 

> Kubernetes version `1.22` was not released until now which means that the deprecated APIs still supported for all clusters. Also, at the same way none OpenShift version which is using Kubernetes API `1.22` was released. Then, we cannot raise errors until it happens for the scenarios where we are `minKubeVersion` is empty and we are assuming all possible versions.
> However, for the checks where the `--optional-values="k8s=1.22"` was used we know that the goal is to verify if the operator bundle will work òn `1.22` then, we can already raise the `error` result. 

Then, it means that:
- `--optional-values="k8s=value"` flag with a value `=> 1.16 <= 1.22` the validator will return result as warning.
- `--optional-values="k8s=value"` flag with a value `=> 1.22` the validator will return result as error.
- `minKubeVersion >= 1.22` return the error result.

**Currently (Before `1.22` release)**
- `minKubeVersion` empty/nil return the warning result.

**After `1.22` k8s release**
- `minKubeVersion >= 1.22 or empty/nil`  return the error result.

Note that the OperatorHubValidator needs to be able to receive a string map and check if it contains any valid key for the specific check to address this requirement.

### Risks and Mitigations

The Kubernetes `1.22` release will impact the Pipeline. After the implementation of the check: 

**After `1.22` k8s release**
- `minKubeVersion >= 1.22 or empty/nil`  return the error result.

All bundles which are using the no longer supported Kubernetes API will start to fail in the above check which indeed is the purpose of this proposal. However, the risk still solved since the Pipeline can indeed inform a Kubernetes version lower such as `--optional-values="k8s=1.21"` if required until all projects are properly update.

Then, this proposal has a very low risks and/or mitigations which are usually faced to address any change. However, see that the usage of the validator can be easily disabled from any place in case it is required until a bug fix can be patched as well. For example, the user or a job can decide not use the command until a bug fix to be addressed.

## Design Details

### Test Plan

The rules implemented should be covered by unit tests in the [operator-framework/api][oper-api].

[operatorhubvalidator]: https://github.com/operator-framework/operator-registry/blob/master/docs/design/operator-bundle.md
[oper-api]: https://github.com/operator-framework/api
[channel-names-doc]: https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/channel-naming.md#naming-convention-rules
