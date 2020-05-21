---
title: service-binding
authors:
  - "@sbose78"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-03-19
last-updated: 2019-05-08
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



## Motivation


Connecting applications to the services that support them—for example, establishing the exchange of credentials between a Java application and a database that it requires—is referred to as binding. The configuration and maintenance of this binding together of applications and backing services can be a tedious and inefficient process. Manually editing YAML files to define binding information is error-prone and can introduce difficult-to-debug failures.

The goal of the Service Binding Operator is to solve this binding problem. By making it easier for application developers to bind applications with needed backing services, the Service Binding Operator also assists operator providers in promoting and expanding the adoption of their operators


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

The Service Binding machinery respects binding-related guidance which backing service operators choose to 
provide. 


### Operator Providing Metadata in CRD Annotations

In this method,
the binding information is provided as annotations in the CRD of the operator
that manages the backing service. The Service Binding Operator extracts the
annotations to bind the application together with the backing service.

For example, this is a *bind-able* operator's annotations in its CRD for a
PostgreSQL database backing operator.
``` yaml
---
[...]
kind: CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1beta1
metadata:
  name: databases.postgresql.baiju.dev
  annotations:
    servicebinding.dev/username: "path={.status.data.dbConfiguration},objectType=ConfigMap"
    servicebinding.dev/password: "path={.status.data.dbConfiguration},objectType=ConfigMap"
    service.binding/uri: "path={.status.data.url}"
spec:
  group: postgresql.baiju.dev
  version: v1alpha1
```

### Operator Providing Metadata in OLM

This guidance enables operator providers to specify binding information an
operator's OLM (Operator Lifecycle Manager) descriptor. The Service Binding
Operator extracts to bind the application together with the backing service.
The information may be specified in the "status" and/or "spec" section of the
OLM in plaintext or as a reference to a Secret or a ConfigMap.

For example, this is a *bind-able* operator OLM Descriptor for a
PostgreSQL database backing operator.
``` yaml
---
[...]
statusDescriptors:

  description: Name of the Secret to hold the DB user and password
    displayName: DB Password Credentials
    path: dbCredentials
    x-descriptors:
      - urn:alm:descriptor:io.kubernetes:Secret
      - servicebinding:user
      - servicebinding:password

  description: Database connection IP address
    displayName: DB IP address
    path: dbConnectionIP
    x-descriptors:
      - servicebinding
```

### User Providing Metadata in the CR or the Kubernetes resource

```yaml
---
[...]

kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: foo
  namespace: bar
  annotations:
    service.binding/uri: "path={.spec.host}"
spec:
  host: example-sbose.apps.ci-ln-smyggvb-d5d6b.origin-ci-int-aws.dev.rhcloud.com
  path: /
  to:
    kind: Service
    name: example
    weight: 100
  port:
    targetPort: 80
  wildcardPolicy: None
```

A detailed guide to decorating backing services to make them binding friendly has been documented [here](https://github.com/application-stacks/service-binding-specification/blob/master/annotations.md).


## Advanced Configuration

### Custom Environment variables


To make binding applications (e.g., legacy Java applications that depend on JDBC connectioon strings)  together with backing services more flexible, the Service Binding Operator supports the optional use of custom environment variables. To use custom environment variables, an application developer creates a ServiceBinding that looks like the one shown 

```yaml
apiVersion: service.binding/v1alpha1
kind: ServiceBinding

metadata:
  name: binding-ecommerce
  namespace: foo

spec:

  applications:
  - resourceRef: payments-app
    group: ""
    version: v1
    resource: deployments

  services:
  - group: postgresql.baiju.dev
    version: v1alpha1
    kind: Database
    resourceRef: db-demo

  dataMapping:
    - name: JDBC_URL
      value: 'jdbc:postgresql://{{ .status.dbConnectionIP }}:{{ .status.dbConnectionPort }}/{{ .status.dbName }}'
    - name: DB_USER
      value: '{{ index .status.dbConfigMap "db.username" }}'
    - name: DB_PASSWORD
      value: '{{ index .status.dbConfigMap "db.password" }}'
```

## Example

### Binding an Imported Quarkus app deployed as Knative service with an In-cluster Operator Managed PostgreSQL Database

The following is a summary of the [scenario](examples/knative_postgresql_customvar/README.md).

  * Create a 'Service' which is this case is a Database

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

  * Create a binding

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
            resourceRef: knative-app
        services:
          - group: postgresql.baiju.dev
            version: v1alpha1
            kind: Database
            resourceRef: db-demo
        dataMapping:
          - name: JDBC_URL
            value: 'jdbc:postgresql://{{ .status.dbConnectionIP }}:{{ .status.dbConnectionPort }}/{{ .status.dbName }}'
          - name: DB_USER
            value: '{{ index .status.dbConfigMap "db.username" }}'
          - name: DB_PASSWORD
            value: '{{ index .status.dbConfigMap "db.password" }}'
      ```

  * Verify that a `Secret` named `binding-request` has been injected into the `KnativeService`.


## Permissions


### Read and Watch Resources which provide binding metadata
* CSVs
* CRDs
* All CRs, which is effecively implies all resources! This is because we don't know the Group and Resource of CRs in advance.

### Read Resources which contain binding information
* ConfigMaps 
* Secrets
* Routes
* Services
* All CRs, which is effecively implies all resources! This is because we don't know the Group and Resource of CRs in advance.

### Read and Update popular podSpec-based workloads
* deployments
* deploymentconfigs
* daemonsets
* replicasets
* statefulsets
* services ( Knative )

We may want to disallow daemonsets, replicasets and statefulsets as they aren't very commonly used without deployments/deploymentconfigs.

## Security

#### Why
Assuming the user gets to create a ServiceBinding CR, how do we avoid letting the user execute an escalation of privilege.

* John doesn't have view on Secrets.
* John creates a ServiceBinding, which leads to a backing service's secret being read, and contents written into a binding secret.
* ServiceBinding controller injects the binding secret into the application workload.
* If John has the privileges to print the environment variables in the Deployment's container, John gets access to secret's contents which were otherwise not visible to John.( aka escalation of privilege)
* If John was otherwise not allowed to modify a Deployment, John gets to do that as well (aka escalation of privilege)

#### How
To avoid an escalation of privilege, we plan to implement a validating webhook to verify the following:

* Does John have reasonable access to the backing services ( and it's sub-resources )?
* Does John have reasonable access to the application ?
* A validating webhook "validates" conditions before an object is accepted. In this case, subject access reviews (SARs) could be made use of, to validate user privileges.

## Bill of materials

Service Binding Controller Image `quay.io/redhat-developer/app-binding-operator` and it's associated `Deployment`'s `ServiceAccount`.


| Component                                                                 | Count 
| ------------------------------------------------------------------------------- | ----- 
| `Deployments` | 1
| `Roles`   |  1 
| `ServiceAccounts`  | 1
| `RoleBindings` |  1
