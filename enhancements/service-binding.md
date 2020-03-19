---
title: service-binding
authors:
  - "@sbose78"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-03-19
last-updated: 2019-03-19
status: implementable
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


## The ServiceBindingRequest custom resource

Manually binding an application together with a backing service without the Service Binding Operator is a time-consuming and error-prone process. 

The steps needed to perform the binding include:

* Locating the binding information in the backing service’s resources.
* Creating and referencing any necessary secrets.
Manually editing the application’s DeploymentConfig, Deployment, Replicaset, KnativeService, or anything else that uses a standard PodSpec to reference the binding request.

In contrast, by using the Service Binding Operator, the only action that an application developer must make during the import of the application is to make clear the intent that the binding must be performed. This task is accomplished by creating the ServiceBindingRequest. The Service Binding Operator takes that intent and performs the binding on behalf of the application developer.

The Service Binding Operator accomplishes this task by automatically collecting binding information and sharing it with an application and an operator-managed backing service. This binding is performed through a new custom resource called a ServiceBindingRequest.


```
apiVersion: apps.openshift.io/v1alpha1
kind: ServiceBindingRequest

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

```

## Advanced Configuration

### Custom Environment variables


To make binding applications (e.g., legacy Java applications that depend on JDBC connectioon strings)  together with backing services more flexible, the Service Binding Operator supports the optional use of custom environment variables. To use custom environment variables, an application developer creates a ServiceBindingRequest that looks like the one shown 

```
apiVersion: apps.openshift.io/v1alpha1
kind: ServiceBindingRequest

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

  customEnvVar:
    - name: JDBC_URL
      value: 'jdbc:postgresql://{{ .status.dbConnectionIP }}:{{ .status.dbConnectionPort }}/{{ .status.dbName }}'
    - name: DB_USER
      value: '{{ index .status.dbConfigMap "db.username" }}'
    - name: DB_PASSWORD
      value: '{{ index .status.dbConfigMap "db.password" }}'
```


Here's an example on [binding an Imported Java Spring Boot app to an In-cluster Operator Managed PostgreSQL Database](https://github.com/redhat-developer/service-binding-operator/blob/master/examples/java_postgresql_customvar/README.md)
