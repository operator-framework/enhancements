---
title: service-binding
authors:
  - "@sbose78"
reviewers:
  - "@bparees"
approvers:
  - TBD
creation-date: 2020-03-19
last-updated: 2021-03-03
status: proposed
---

# Service Binding for operator-backed services

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

The Service Binding Controller enables applications to use external services by automatically collecting and sharing binding information (credentials, connection details, volume mounts, secrets, etc.) with the application. In effect, the Service Binding Operator defines a contract between a “bindable” backing service (for example, a database operator) and an application requiring that backing service.

Note that in addition to the initial sharing of binding information, the binding is also “managed” by the Service Binding Operator. This statement means that, if credentials or URLs undergo modification by the backing service operator, those changes are automatically reflected in the application.

This enhancement proposal provides a background on the current state of the implementation of the [Service Binding Operator project](https://github.com/redhat-developer/service-binding-operator/), makes a case for onboarding the project into the Operator Framework organization and eventually have the controllers shipped as part of OLM.



## Motivation

Connecting applications to the services that support them—for example, establishing the exchange of credentials between a Java application and a database that it requires—is referred to as binding. The configuration and maintenance of this binding together of applications and backing services can be a tedious and inefficient process. Manually editing YAML files to define binding information is error-prone and can introduce difficult-to-debug failures.

The goal of the Service Binding Operator is to solve this binding problem. By making it easier for application developers to bind applications with needed backing services, the Service Binding Operator also assists operator providers in promoting and expanding the adoption of their operators.


## The ServiceBinding custom resource

Manually binding an application together with a backing service without the Service Binding Operator is a time-consuming and error-prone process. 

The steps needed to perform the binding include:

* Locating the binding information in the backing service’s resources.
* Creating and referencing any necessary secrets.
* Modifying the application’s DeploymentConfig, Deployment, Replicaset, KnativeService, or anything else that uses a standard PodSpec to reference the binding request.

In contrast, by using the Service Binding Operator, the only action that an application developer must make during the import of the application is to make clear the intent that the binding must be performed. This task is accomplished by creating the `ServiceBinding`. The Service Binding controller takes that intent and performs the binding on behalf of the application developer.

The Service Binding Operator accomplishes this task by automatically collecting binding information and sharing it with an application and an operator-managed backing service. This binding is performed through a new custom resource called a `ServiceBinding`.


```yaml
apiVersion: service.binding/v1alpha1
kind: ServiceBinding

metadata:
  name: binding-ecommerce
  namespace: foo

spec:

  applications:
  - name: payments-app
    group: ""
    version: v1
    resource: deployments

  services:
  - group: postgresql.baiju.dev
    version: v1alpha1
    kind: Database
    name: db-demo

```

## Adding binding guidance to Services


The Service Binding machinery respects binding-related guidance which backing service operators choose to provide. 


### Backing Service providing binding metadata


If the backing service author has provided binding metadata in the corresponding CRD,
then Service Binding acknowledges it and automatically creates a binding secret with
the information relevant for binding.

The backing service may provide binding information as
* Metadata in the CRD as annotations
* Metadata in the OLM bundle manifest file as CSV Descriptors
* Secret or ConfigMap

If the backing service provides binding metadata, you may use the resource as is
to express an intent to bind your workload with one or more service resources, by creating a `ServiceBinding`.

### Backing Service not providing binding metadata


If the backing service hasn't provided any binding metadata, the application author may annotate the Kubernetes resource representing the backing service such that the managed binding secret generated has the necessary binding information.

As an application author, you have a couple of options to extract binding information
from the backing service:

* Decorate the backing service resource using annotations.
* Define custom binding variables.

In the following section, details of the above methods to make a backing service consumable for your application workload, is explained.

### Annotate Service Resources


The application author may consider specific elements of the backing service resource interesting for binding

* A specific attribute in the `spec` section of the Kubernetes resource.
* A specific attribute in the `status` section of the Kubernetes resource.
* A specific attribute in the `data` section of the Kubernetes resource.
* A specific attribute in a `Secret` referenced in the Kubernetes resource.
* A specific attribute in a `ConfigMap` referenced in the Kubernetes resource.

As an example, if the Cockroachdb authors do not provide any binding metadata in the CRD, you, as an application author may annotate the CR/kubernetes resource that manages the backing service ( cockroach DB ).

The backing service could be represented as any one of the following:
* Custom Resources.
* Kubernetes Resources, such as `Ingress`, `ConfigMap` and `Secret`.
* OpenShift Resources, such as `Routes`.

## Compose custom binding variables

If the backing service doesn't expose binding metadata or the values exposed are not easily consumable, then an application author may compose custom binding variables using attributes in the Kubernetes resource representing the backing service.

The *custom binding variables* feature enables application authors to request customized binding secrets using a combination of Go and jsonpath templating.

Example, the backing service CR may expose the host, port and database user in separate variables, but the application may need to consume this information as a connection string.




``` yaml
apiVersion: binding.operators.coreos.com/v1alpha1
kind: ServiceBinding
metadata:
  name: multi-service-binding
  namespace: service-binding-demo
spec:

  application:
    name: java-app
    group: apps
    version: v1
    resource: deployments

 services:
  - group: postgresql.baiju.dev
    version: v1alpha1
    kind: Database
    name: db-demo   <--- Database service
    id: postgresDB <--- Optional "id" field
  - group: ibmcloud.ibm.com
    version: v1alpha1
    kind: Binding
    name: mytranslator-binding <--- Translation service
    id: translationService

  mappings:
    ## From the database service
    - name: JDBC_URL
      value: 'jdbc:postgresql://{{ .postgresDB.status.dbConnectionIP }}:{{ .postgresDB.status.dbConnectionPort }}/{{ .postgresDB.status.dbName }}'
    - name: DB_USER
      value: '{{ .postgresDB.status.dbCredentials.user }}'

    ## From the translator service
    - name: LANGUAGE_TRANSLATOR_URL
      value: '{{ index translationService.status.secretName "url" }}'
    - name: LANGUAGE_TRANSLATOR_IAM_APIKEY
      value: '{{ index translationService.status.secretName "apikey" }}'

    ## From both the services!
    - name: EXAMPLE_VARIABLE
      value: '{{ .postgresDB.status.dbName }}{{ translationService.status.secretName}}'

    ## Generate JSON.
    - name: DB_JSON
      value: {{ json .postgresDB.status }}

```

This has been adopted in [IBM CodeEngine](https://cloud.ibm.com/docs/codeengine?topic=codeengine-kn-service-binding).

## Example

### Binding an Imported Quarkus app deployed as Knative service with an In-cluster Operator Managed PostgreSQL Database

The following is a summary of the [scenario](examples/knative_postgresql_customvar/README.md).

  * Create a 'Service', which in this case is a Database

      ```yaml
      apiVersion: postgresql.baiju.dev/v1alpha1
      kind: Database
      metadata:
        name: db-demo
        namespace: service-binding-demo
      spec:
        image: docker.io/postgres
        imageName: postgres
        dbName: db-demo
      ```

  * Deploy a `Knative` `Service` called `knative-app` 

  * Create a `ServiceBinding`

      ```
      apiVersion: service.binding/v1alpha1
      kind: ServiceBinding
      metadata:
        name: binding-request
        namespace: service-binding-demo

      spec:
        applications:
          - group: serving.knative.dev
            version: v1beta1
            resource: services
            name: knative-app
        services:
          - group: postgresql.baiju.dev
            version: v1alpha1
            kind: Database
            name: db-demo
        mapping:
          - name: JDBC_URL
            value: 'jdbc:postgresql://{{ .status.dbConnectionIP }}:{{ .status.dbConnectionPort }}/{{ .status.dbName }}'
          - name: DB_USER
            value: '{{ index .status.dbConfigMap "db.username" }}'
          - name: DB_PASSWORD
            value: '{{ index .status.dbConfigMap "db.password" }}'
      ```

  * Verify that a `Secret` named `binding-request` has been injected into the `KnativeService`.


## Permissions


### Read Resources which contain binding information
* ConfigMaps 
* Secrets 
* Services
* Custom resource types which may participate in binding.

### Read and Update popular podSpec-based workloads.
* deployments
* deploymentconfigs
* daemonsets
* replicasets
* statefulsets
* services ( Knative )

We may want to disallow daemonsets, replicasets and statefulsets out of the box as they aren't very commonly used without deployments/deploymentconfigs.

## Security

### Why
Assuming the user gets to create a ServiceBinding CR, how do we avoid letting the user execute an escalation of privilege.

* John doesn't have view on Secrets.
* John creates a ServiceBinding, which leads to a backing service's secret being read, and contents written into a binding secret.
* ServiceBinding controller injects the binding secret into the application workload.
* If John has the privileges to print the environment variables in the Deployment's container, John gets access to secret's contents which were otherwise not visible to John.( aka escalation of privilege)
* If John was otherwise not allowed to modify a Deployment, John gets to do that as well (aka escalation of privilege)

### How
To avoid an escalation of privilege, we plan to implement a validating webhook to verify the following:

* Does John have reasonable access to the backing services ( and it's sub-resources )?
* Does John have reasonable access to the application ?
* A validating webhook "validates" conditions before an object is accepted. In this case, subject access reviews (SARs) could be made use of, to validate user privileges.

## Organization

* It is hereby proposed that the project currently hosted at [redhat-developer/service-binding-operator](https://github.com/redhat-developer/service-binding-operator/) be onboarded into [operator-framework/operator-lifecycle-manager](https://github.com) organization.

* The project would be governed by the processes of the Operator Framework community.


## Delivery

* The controller and associated resources as specified in the following section would be shipped with OLM. Details on the implementation of the same is not in scope of this enhancement proposal.
* The Service Binding project would continue shipping standalone releases.


## Bill of materials

Service Binding Controller Image `quay.io/redhat-developer/app-binding-operator` and it's associated `Deployment`'s `ServiceAccount`.


| Component                                                                 | Count 
| ------------------------------------------------------------------------------- | ----- 
| `Deployments` | 1
| `Roles`   |  1 
| `ServiceAccounts`  | 1
| `RoleBindings` |  1
