---
title: operator-status
authors:
  - "@awgreene"
reviewers:
  - "@ecordell"
approvers:
  - "@kevinrizza"
creation-date: 2020-05-19
last-updated: 2020-06-10
status: implementable
see-also:
  - "N/A"  
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# operator-status

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

The goal of this enhancement is to allow operators managed by the [Operator Lifecycle Manager (OLM)](https://github.com/operator-framework/operator-lifecycle-manager) operator to communicate conditions and states that OLM would otherwise be unable to infer. The presence of certain conditions should influence OLM behavior to protect the cluster.

## Motivation

OLM currently infers the state of an operator by inspecting the Kubernetes resources that define the operator. OLM does not provide operators with the means to communicate complex states.

Operators managed by OLM would benefit from the ability to communicate their state directly to OLM through a supported channel. Operators could communicate complex conditions and influence how OLM manages the operator, dramatically improving user experience.

A core requirement of this enhancement is to provide operators with the ability to communicate "Upgrade Readiness", providing operators with the ability to signal to OLM that they should not be upgraded at this time. Consider the following scenario based on OLM today:

>An operator installed with automatic updates is executing a critical process that could place the cluster in a bad state if OLM upgrades the operator. While this critical process is running, a new version of the operator becomes available. Since the operator was installed with automatic updates, OLM upgrades the operator without allowing the critical process to finish and places the cluster in an unrecoverable state.

OLM could handle the scenario described above if the operator had the ability to communicate that it should not be upgraded as it ran the critical process.

### Goals

- Provide operators with the means to explicitly communicate a condition that influences OLM behavior, such as the "Upgrade Readiness" condition defined above.
- Preserve existing behavior if the operator does not take advantage of this feature.

### Non-Goals

- Provide operators with the means to cancel an upgrade that is in process.
- OLM cannot guarantee that some outside force will not interrupt the services of the operator and will not provide the operator with the ability to communicate that it should not be "Interrupted".

## Proposal

### User Stories

#### Story 1

As an operator author, I want my operator to be able to communicate conditions to OLM that it cannot infer otherwise.

#### Story 2

As an operator author, I want my operator to be able to communicate that OLM should not attempt to upgrade my operator until I tell it otherwise.

## Design Details

OLM will introduce the `Probe` CustomResourceDefinition which will communicate an operator's state through the use of [conditions](https://github.com/kubernetes/enhancements/blob/3aa92e20bdfa2e60e3cb1a2b92cdca61847d7ad2/keps/sig-api-machinery/1623-standardize-conditions/README.md#kep-1623-standardize-conditions). For those unfamiliar with conditions, they are used by many Kubernetes Resources to provide users with additional insight into the state of the resource and typically adhere to the following format:

| Field | Description |
| ----- | ----------- |
| Type | Type of condition in CamelCase or in foo.example.com/CamelCase. |
| Status | Status of the condition, one of True, False, Unknown. |
| ObservedGeneration | If set, this represents the .metadata.generation that the condition was set based upon. |
| LastTransitionTime | Last time the condition transitioned from one status to another. |
| Reason | The reason for the condition's last transition in CamelCase. |
| Message | A human readable message indicating details about the transition. |

OLM will define a set of "OLM Supported Conditions" which will influence the behavior of OLM when present. The list of "OLM Supported Conditions" can be seen in the table below:

| Condition Type | Effect |
| -------------- | ------ |
| Upgradeable | When `false`, OLM will not upgrade the operator.|
| Important | OLM will highlight this condition to the Cluster Admin.|

Operators will communicate their conditions through one of two methods:

1. By setting conditions on CustomResource that the operator owns.
2. By setting conditions on a `Probe` CustomResource directly.

### Why allow operators to communicate status through CustomResources

There are many reasons why an operator may want to communicate its status through owned CustomResources:

- Operators already write to the status of their CustomResources to communicate "Actual State".
- By writing the condition to the status of an owned CustomResource, it becomes very clear which CR is responsible for changing OLM behavior.
- Introducing an OLM owned CRD forces operators to include OLM specific code, which an author may not want to maintain.

#### Drawbacks to Communicating status through CustomResources

Allowing operators to write to the status of any CR creates a potential scaling issue if enough CRs are on cluster. OLM will avoid these performance issues by introducing the Status Probe Operator (SPO) which will reconcile the `Probe` resource. If an operator chooses to write to their own CRs, SPO will create a pod that will keep the status of the `Probe` resource synced with the conditions in the CRs.

### Opting into OLM Supported Conditions

Operators may opt-in to "OLM Supported Conditions" by adding annotations to their owned CRD(s), where the key is the defined by OLM and the value is a boolean expression over condition types applied to the CR at `.status.conditions`. The "OLM Supported Condition" annotation key will adhere to the following format: `olm.operatorframework.io/condition.<ConditionName>`.

Let's consider the case where an operator is upgradeable when the migrating condition on a CR is false, the "OLM Supported Condition" annotation should be `olm.operatorframework.io/condition.Upgradeable: "!Migrating"`. Alternatively, the operator author could indicate that it is upgradeable when both the `ReadyToGo` and `FinishedMigrating` conditions are true by adding the `olm.operatorframework.io/condition.Upgradeable: "FinishedMigrating && ReadyToGo"` annotation.

### Example Workflow

Let's examine the complete workflow for an operator that owns the `Foo` crd where:

- The operator should not be upgraded when a `Foo` CR has the `Migrating` condition set to true.
- OLM should surface any CR condition whose `Type` matches `BadConnectivity` or `UnhealthyDatabase` in the `Probe` resource.

In this case, the operator's CRD should include the following annotations:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata: {
  name: foo
  annotations: {
    "olm.operatorframework.io/condition.Upgradeable" : "!Migrating" # The operator is upgradeable when the CR is not being migrated.
    "olm.operatorframework.io/condition.Important" : "BadConnectivity || UnhealthyDatabase" # The Important condition should only include the `OR` boolean operator.
  }
}
```

When this annotation is present on a CRD, OLM will append the following `Probe` CR to the `installPlan`:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Probe
metadata:
  name: foo-operator.v1.0.0
  namespace: operators
spec:
  manager: crdAnnotations # Changes to the CRD's annotations should override the spec. OLM will watch the CRD for changes to the "OLM Supported Condition" annotations and update the spec accordingly.
  probeResources:
  - resource: foo.example.com
    upgradeable: "!Migrating"
    important: "BadConnectivity || UnhealthyDatabase"
```

When the`probe` resource above is created, OLM will spin up an SPO pod which collects the information from conditions of `foo.example.com` CRs and communicate it to OLM via the `probe`'s status accordingly. If an operator were to define multiple resources that it communicates its status through, a single SPO pod would monitor all of those resources. The permissions of the probe will match the same permissions of the operator.

>Note: If the probe's spec is nil, SPO will not create a pod and the operator will be expected to update the CRs status itself.

In this example, the SPO will be responsible for maintaining the following fields in the status of the `probe` resource:

```yaml
status:
  conditions:
    upgradeable: # A boolean that represents if the operator may be upgraded - aggregated from all probeResources below.
  probeResources: # The list of probe resources.
```

Consider an upgrade where the a single `Foo` CR exists on cluster:

```yaml
apiVersion: foo.example.com/v1
kind: Foo
metadata:
  name: foo-example
  namespace: default
spec:
  ...
  ...
  ...
status:
  conditions:
  - type: Migrating # Condition 1
    status: "True"
    reason: "CriticalProcess"
    message: "Migrating old database schema to new format"
  - type: UnhealthyDatabase # Condition 2
    status: "True"
    reason: "LivenessProbeFailure"
    message: "The foo-database failed its liveness probe."
  - type: BadConnectivity # Condition 3
    status: "True"
    reason: "404"
    message: "Unable to reach the database"
  - type: SomethingUnrelated # Condition 4
    status: "True"
    reason: "Unmatched"
    message: "OLM doesn't care about this condition"
```

When SPO registers this CR, the Probe CR will be updated like so:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Probe
metadata:
  name: foo-operator.v1.0.0 # Same name as the CSV.
  namespace: operators
spec:
  manager: crdAnnotations # When set to `crdAnnotations` OLM will override any changes to the probe's spec to match those defined via annotations on the CRDs described above.
  probeResources:
  - name: foo.example.com
    upgradeable: "!Migrating"
    important: "BadConnectivity || UnhealthyDatabase"
status:
  conditions:
  - type: Upgradeable # Overall Upgradeable condition
    status: "False"
    reason: "NotUpgradeable"
    message: "The operator has communicated that the operator is not upgradeable"
  probeResources:
  - kind: foos # ObjectReference
    name: "foo-example"
    namespace: default
    reasons: # The Reason the CR is mapping to an "OLM Supported Condition"
    - "!Migrating"
    - "UnhealthyDatabase"
    - "BadConnectivity"
```

From the presence of the false `Upgradeable` condition, OLM will understand that it should not upgrade the operator. The Conditions used to evaluate the overall `Upgradeable` status of the Operator are stored within the `status.probeResources` array. Notice that the Condition that did not map to an "OLM Supported Condition" was not tracked in the `Probe`.

>Note: If a single CR communicates that the operator is not upgradeable, OLM will not upgrade the operator.

The complexity of communicating state of an operator can be reduced by having the operator introduce a new CRD which is used to communicate its conditions or by writing to the probe resource directly. This creates a single source of truth for the state of the operator.

### Writing to the Probe Resource Directly

An Operator can manage its status by writing directly to the `Probe` resource associated with its operator. The [operator-sdk](https://github.com/operator-framework/operator-sdk) project will provide a library that includes a set of functions geared towards managing the status [subresource](https://book-v1.book.kubebuilder.io/basics/status_subresource.html) of the `Probe` CRD introduced by OLM.

The functions provided within the library will detect if the `Probe` CRD is present on the cluster using the Discovery API. If the `Probe` resource exists, the library will provide a set of functions allowing a user to `GET` or `UPDATE` the operator's `Probe`.

The library will also communicate to OLM that the operator is managing the `Probe` resource by setting the `spec.manager` field to `operator`. When the `spec.manager` field is set to `operator`, the library will throw an error if the `spec.ProbeResources` field is not `nil`.

At the time of this enhancement, the operator will only need to manage its `Upgradeable` condition.

#### Example: Operator Managed Probe Resource

Let's consider what the `Probe` resource may look like for the Foo operator:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Probe
metadata:
  name: foo-operator.v1.0.0
  namespace: operators
spec:
  manager: operator  # When set to `operator`, the operator must manage the state of the probe resource.
status:
  conditions:
  - type: Upgradeable # Overall Upgradeable condition
    status: "False"     # The operator is not upgradeable
    reason: "FooProcess"
    message: "The operator is not upgradeable because FooProcess is running"
```

In this case, the operator is communicating that OLM should not upgrade the operator at this time. Likewise, the operator could convey that OLM may upgrade the operator with the following `Probe`:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Probe
metadata:
  name: foo-operator.v1.0.0
  namespace: operators
spec:
  manager: operator
status:
  conditions:
  - type: Upgradeable # Overall Upgradeable condition
    status: "True"    # The operator is upgradeable
    reason: "ReadyForUpgrade"
    message: "The operator is ready to be upgraded"
```

### Test Plan

OLM's e2e testing suite will be upgraded to handle the usecase:

- If an operator does not opt-in to "OLM Supported Conditions", OLM reverts to existing behavior.
- If an operator opts-in to the `Upgradeable` "OLM Supported Conditions":
  - Upgrades are not blocked if no CRs exist.
  - Upgrades are not blocked if a CR exists but the the defined condition expression cannot be calculated.
  - Upgrades are not blocked if a CR exists and the boolean expression is false.
  - Upgrades are blocked if a CR exists and the boolean expression is true.

### Risks and Mitigations

- OLM cannot assume that a Condition on a CR with the type `Upgradeable` maps to the `Upgradeable` "OLM Supported Condition". It is impossible for OLM to guarantee that an "OLM Supported Condition" type is not already being used as a condition type by some operator in the world. By making this feature opt-in only, OLM avoids this conflict.
- Operators may opt-in to this feature but never update the endpoint as "upgradeable". In these situations, cluster admins must uninstall the operator and upgrade the operator manually. In the future, OLM could offer a soft and hard grace period.

## Alternatives

There were a number of viable alternatives that were considered when creating this enhancement.

### Alternative 1: Probe CRD

In this approach, OLM would introduce a new namespaced [CustomResourceDefinition](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/) called the Probe CRD. The Probe object would have no spec. The Probe's [Status subresource](https://book-v1.book.kubebuilder.io/basics/status_subresource.html) would include a map of conditions based off the  [Kubernetes APIServiceCondition](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#podcondition-v1-core). The Maps key's will be consistent with the "Name" of the condition being communicated and the values.

The Probe CRD would be used by operator to:
- Communicate conditions and states that should influence how OLM manages the operator, such as preventing OLM from upgrading the operator.
- Communicate messages to cluster admins, such as slow off-cluster queries or poorly configured CRs. These communications will not influence OLM behavior.

Similarly, OLM may use the Probe CRD to:
- Communicate to cluster admins the state of an operator, such as why an operator is unable to be upgraded.
- Communicate with the operator about upcoming actions.

#### Pros

- A single point of communication between OLM and Operators.
- Feature is opt-in, operators can choose not to take advantage of this feature.
- This approach can be expanded to support new communication channels easily.
- Allows communication of conditions that influence OLM or provide helpful messages to Cluster Admins.

#### Cons

- Operators are required to know when they are managed by OLM and write to an OLM resource.
- Expands the scope of APIs introduced by OLM.
- Could confuse users since the OLM owned Operator CRD already conveys status.

### Alternative 2: Operators write to the status of the CSV or Operators API

This approach considered allowing Operators to write to the status of the [CSV](https://github.com/operator-framework/api/blob/6468209c61b24bed76ed4701eba1a8b7f81b2377/crds/operators.coreos.com_clusterserviceversions.yaml#L7) or the [Operators](https://github.com/operator-framework/api/blob/6468209c61b24bed76ed4701eba1a8b7f81b2377/crds/operators.coreos.com_operators.yaml#L7) API rather than introducing a new API managed by OLM and was quickly abandoned for the reasons discussed below in Pros/Cons.

#### Pros

- No new APIs.
- A single source of truth for operator status.
- Could easily expand to support other communication channels.

#### Cons

- Operators are required to know when they are managed by OLM and write to an OLM resource.
- Requires Operator Authors to learn OLM specific readiness reporting conventions.
- Operators granted the ability to update a CSV's status can change fields which influences OLM behavior. For example, the operator might accidentally change the `Phase` of a CSV.

### Alternative 3: Upgrade Readiness Endpoints

This approach did not consider how an operator would report generic operator status and instead focused on upgrade readiness. In this approach, operators would communicate upgrade readiness by exposing an `upgradeReadiness` endpoint. This endpoint could have communicated upgrade readiness by:

- printing 0, indicating that the operator cannot be upgraded at this time.
- printing 1, indicating that the operator is ready to be upgraded.

The endpoint would have been exposed via a [Service resource](https://kubernetes.io/docs/concepts/services-networking/service/#service-resource) that OLM would probe for upgrade readiness prior to upgrading the operator to the next version. In the absence of a defined "Upgrade Readiness Service", OLM would have fallen on default behavior.

This approach would have had to have supported all bundle mediatypes, but let's consider how this would have looked in both CSV and CSVless bundles.

#### CSV Bundles

OLM's [ClusterServiceVersion](https://github.com/operator-framework/api/blob/197407cd70e8ddfef85d21216085ed52fbb4bb2d/pkg/operators/v1alpha1/clusterserviceversion_types.go#L500) type would have be updated to include an upgradeableService for each [StrategyDeploymentSpec](https://github.com/operator-framework/api/blob/197407cd70e8ddfef85d21216085ed52fbb4bb2d/pkg/operators/v1alpha1/clusterserviceversion_types.go#L63).

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
metadata:
  name: foo-operator.v1.0.0
  namespace: olm
spec:
  description: |
    An example operator
  displayName: foo-operator
  install:
    spec:
      deployments:
      - name: foo-operator
        upgradeReadinessServices:              # An array of readiness services
        - name: foo-operator-readiness-service # The Service name
          spec:                                # Standard Service spec
            selector:                            
              app: foo-operator
            ports:
            - protocol: TCP
              port: 80
              targetPort: 9376
        spec:
          replicas: 1
          selector:
            matchLabels:
                app: foo-operator
          template:
            metadata:
              labels:
                app: foo-operator
            spec:
              containers:
              - name: foo-operator
                image: foo
                command:
                  - sleep
                  - "3600"
    strategy: deployment
  installModes:
  - supported: false
    type: OwnNamespace
  - supported: true
    type: SingleNamespace
  - supported: true
    type: MultiNamespace
  - supported: true
    type: AllNamespaces
  maturity: alpha
  provider:
    name: Red Hat
  version: 1.0.0
```

Alternatively, the operator author could omit the Upgradeable Readiness Service Spec and simply specify the name. When using this approach, the service will typically be present in the operator bundle:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
metadata:
  name: foo-operator.v1.0.0
  namespace: olm
spec:
  description: |
    An example operator
  displayName: foo-operator
  install:
    spec:
      deployments:
      - name: foo-operator
        readinessServices:                     # An array of upgradeReadiness services
        - name: foo-operator-readiness-service # The name of the Service
        spec:
          replicas: 1
          selector:
            matchLabels:
                app: foo-operator
          template:
            metadata:
              labels:
                app: foo-operator
            spec:
              containers:
              - name: foo-operator
                image: foo
                command:
                  - sleep
                  - "3600"
    strategy: deployment
  installModes:
  - supported: false
    type: OwnNamespace
  - supported: true
    type: SingleNamespace
  - supported: true
    type: MultiNamespace
  - supported: true
    type: AllNamespaces
  maturity: alpha
  provider:
    name: Red Hat
  version: 1.0.0
```

The operator bundle would have included the following Service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: foo-operator-readiness-service
spec:
  selector:
    app: foo-operator
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

#### CSVless Bundles

In the case of a CSVless bundle, an annotation could have been applied to the `olm.yaml` which indicates the name of the upgradeReadinessService and the deployment name:

```yaml
  operators.operatorframework.io.deployment.<DeploymentName>.readiness.service: "<ServiceName>"
```

Once the `olm.yaml` is created with the key/value above, the following resources would have needed to be present in the operator bundle:

- A service whose name matches the `ServiceName` in your annotation which routes to the readiness endpoint.
- A deployment
  - Whose name matched the DeploymentName used in your annotation
  - Had a `podTemplate` with a label selected by the service

#### Pros

- Allows users to define multiple readiness checks for both CSV and CSVLess bundles.
- Exposing a binary output at an endpoint exposed in your operator was similar to the existing and well adopted [readiness probe format](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) which the operator author is likely familiar with.

#### Cons

- Defining a communication channel (in this case an "Upgrade Readiness Service") is unique for each manifest format.
- This approach did not support generic status reporting.
- This approach did not lend itself well to covering other unexpected usecases. Adding new communication channels would prove difficult and require users to learn how to define each communication channel as they became available.
- The Service used to communicate to OLM could forward requests to any pod that matches the Service's LabelSelector, which could be an issue if multiple pods with the SelectorLabels existed.

### Alternative 4: Allow Operators to define endpoints that OLM would probe

The fourth alternative closely mimicked the third, but focused on pulling the [Prober](https://github.com/kubernetes/kubernetes/blob/release-1.18/pkg/kubelet/prober/prober.go#L48) code into OLM and use it to conduct "Readiness Probes" against endpoints that the Operator defined.

This approach once again sought to take advantage of the well known standard used by [Liveness, Readiness, and StartUp Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) . Specifically, this approach supports [HTTP Gets, EXEC, and TCPSocket](https://github.com/kubernetes/api/blob/6f652b6ce59c386f4a431eb031d0339620aaff5e/core/v1/types.go#L2279-L2292) options.

This approach would have had to have supported all bundle mediatypes, but let's consider how this would have looked in both CSV and CSVless bundles.

#### CSV Bundles

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
metadata:
  name: foo-operator.v1.0.0
  namespace: olm
spec:
  description: |
    An example operator
  displayName: foo-operator
  install:
    spec:
      deployments:
      - name: foo-operator
        upgradeReadinessProbes:                 # An array of readiness probes
        - name: foo-operator-readiness-endpoint # The endpoint name
          exec:
            command:
            - cat
            - /tmp/readiness
        - name: foo-operator-second-readiness-enpoint
          httpGet:
            path: /readiness
            port: 8080
            httpHeaders:
            - name: Custom-Header
              value: Awesome
        spec:
          replicas: 1
          selector:
            matchLabels:
                app: foo-operator
          template:
            metadata:
              labels:
                app: foo-operator
            spec:
              containers:
              - name: foo-operator
                image: foo
                command:
                  - sleep
                  - "3600"
    strategy: deployment
  installModes:
  - supported: false
    type: OwnNamespace
  - supported: true
    type: SingleNamespace
  - supported: true
    type: MultiNamespace
  - supported: true
    type: AllNamespaces
  maturity: alpha
  provider:
    name: Red Hat
  version: 1.0.0
```

#### CSVLess Bundles

Cracks in this approach began to appear when considering how it would have been implemented in CSVLess bundles. The primary benefit of adopting the Prober code into OLM was its support for [HTTP Gets, EXEC, and TCPSocket](https://github.com/kubernetes/api/blob/6f652b6ce59c386f4a431eb031d0339620aaff5e/core/v1/types.go#L2279-L2292) options. Unfortunately, the ability to support the various Prober options spawned far too many combinations of KeyValues that would have had to exist in the `olm.yaml` file. Ultimately, OLM would probably have had to create a new way of defining possible options.

#### Pros

- CSV Bundles followed a well defined probe format which supports [HTTP Gets, EXEC, and TCPSocket](https://github.com/kubernetes/api/blob/6f652b6ce59c386f4a431eb031d0339620aaff5e/core/v1/types.go#L2279-L2292) options.
- Enabled OLM to identify which POD is reporting an upgradeReadiness failure, as a service could route to one of any number of pods.

#### Cons

- Requires OLM to create an import and maintain prober code which executes the probe.
- CSVLess Bundle support introduced additional complexity both in design and on user adoption.
