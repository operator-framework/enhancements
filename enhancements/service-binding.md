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

## Making a service bindable

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
    servicebindingoperator.redhat.io/status.dbConfigMap.password: 'binding:env:object:secret'
    servicebindingoperator.redhat.io/status.dbConfigMap.username: 'binding:env:object:configmap'
    servicebindingoperator.redhat.io/status.dbName: 'binding:env:attribute'
    servicebindingoperator.redhat.io/spec.Token.private: 'binding:volumemount:secret'
spec:
  group: postgresql.baiju.dev
  version: v1alpha1
```

### Operator Providing Metadata in OLM

This feature enables operator providers to specify binding information an
operator's OLM (Operator Lifecycle Manager) descriptor. The Service Binding
Operator extracts to bind the application together with the backing service.
The information may be specified in the "status" and/or "spec" section of the
OLM in plaintext or as a reference to a secret.

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
      - binding:env:object:secret:user
      - binding:env:object:secret:password
  description: Database connection IP address
    displayName: DB IP address
    path: dbConnectionIP
    x-descriptors:
      - binding:env:attribute
```

### User Providing Metadata in the CR or the Kubernetes resource

```
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: foo
  namespace: bar
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


## In Action

* [Binding an Imported app with an In-cluster Operator Managed PostgreSQL Database](examples/nodejs_postgresql/README.md)

    * Create a 'Service' which is this case is a Database

      ```
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

    * Deploy an application called `nodejs-rest-http-crud` as a Kubernetes `Deployment`

    * Create a binding

      ```
      apiVersion: apps.openshift.io/v1alpha1
      kind: ServiceBindingRequest
      metadata:
        name: binding-request
        namespace: service-binding-demo
      spec:
        applications:
          - resourceRef: nodejs-rest-http-crud
            group: apps
            version: v1
            resource: deployments
        services:
          - group: postgresql.baiju.dev
            version: v1alpha1
            kind: Database
            resourceRef: db-demo
      ```



* [Binding an Imported app with an Off-cluster Operator Managed AWS RDS Database](examples/nodejs_awsrds_varprefix/README.md)

    * Create a 'Service' which is this case is a Database

      ``` 
      apiVersion: aws.pmacik.dev/v1alpha1
      kind: RDSDatabase
      metadata:
        name: mydb
        namespace: service-binding-demo

      spec:
        class: db.t2.micro
        engine: postgres
        dbName: mydb
        name: mydb
        password:
          key: DB_PASSWORD
          name: mydb
        username: postgres
        deleteProtection: true
        size: 10

      status:
        dbConnectionConfig: mydb
        dbCredentials: mydb
        message: ConfigMap Created
        state: Completed
      ```

    * Deploy an application called `nodejs-app` as a Kubernetes `Deployment`

    * Create a binding

      ```
      apiVersion: apps.openshift.io/v1alpha1
      kind: ServiceBindingRequest
      metadata:
        name: mydb.to.nodejs-app
        namespace: service-binding-demo
      spec:
        envVarPrefix: "MYDB"
        services:
          - group: aws.pmacik.dev
            version: v1alpha1
            kind: RDSDatabase
            resourceRef: mydb
        applications:
          - resourceRef: nodejs-app
            group: apps
            version: v1
            resource: deployments
      ```

* [Binding an Imported Quarkus app deployed as Knative service with an In-cluster Operator Managed PostgreSQL Database](examples/knative_postgresql_customvar/README.md)

    * Create a 'Service' which is this case is a Database

      ```
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

    * Deploy an `KnativeService` called `knative-app` 

    * Create a binding

      ```
      apiVersion: apps.openshift.io/v1alpha1
      kind: ServiceBindingRequest
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
        customEnvVar:
          - name: JDBC_URL
            value: 'jdbc:postgresql://{{ .status.dbConnectionIP }}:{{ .status.dbConnectionPort }}/{{ .status.dbName }}'
          - name: DB_USER
            value: '{{ index .status.dbConfigMap "db.username" }}'
          - name: DB_PASSWORD
            value: '{{ index .status.dbConfigMap "db.password" }}'
      ```

      * Verify that a `Secret` named `binding-request` has been injected into the `Deployment`.



* [Binding an Imported app to an Off-cluster Operator Managed IBM Cloud Service](examples/nodejs_ibmcloud_operator/README.md)

  
