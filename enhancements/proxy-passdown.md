---
title: proxy-friendly-operators
authors:
  - "@asmacdo"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2021-07-02
last-updated: 2021-07-02
status: implementable
---

# Operators Support Network Proxy

## Summary

Document, simplify, and encourage Operator authors to pass proxy
environment variables to operands.

## Motivation

Working with a proxy is a feature that needs to be added to each
operator separately, enabling the end-user story:
"As an operator-admininstrator, I
can deploy operators that respect a network proxy."

For Cluster administrators to set up a network proxy:
   1. Pods requiring network must respect proxy environment variables
      `HTTP_PROXY`, `HTTPS_PROXY`, and `NO_PROXY`
   2. Operators must be provided environment variables
   3. Operators must pass the environment variables to operands

All of this is already possible but operator authors must implement this
functionality themselves. This should be a "best practice"
and documentation for this in our repository.

## Proposal

### Goals

   1. Establish a best practice for operator authors:  Operators receive proxy
      environment variables from the operator deployment, and pass proxy
      environment variables to operands that require network.
   2. Operator SDK features and documents to facilitate implementation of
      proxy-friendly operators.

### Non-Goals

   1. It would be nice to encourage proxy friendly operators by
      including that functionality as a badge or part of the maturity
      model on operatorhub, but that is out of scope for this proposal.
   2. Automatic scaffolding of proxy passdown cannot be done because the
      logic must occur in operator-specific code.
   3. It is beyond the scope of this proposal to set up trusted certs
      required for https connections to a proxy.


### How the Operator receives Env Vars

In all cases, proxy environment variables to be passed to operands must
be included on the Operator deployment. This can be handled by the
administrator, or automatically set by OLM.

### How the Operator passes Env vars to Operands

For Go-based operators, set of functions will be created in operator-lib
(and offered upstream to controller-runtime) to read the proxy variables
from the Operator environment and inject these variables into each of
the container specs of various kubernetes objects. Operator authors will
be responsible for invoking these functions. We will need functions for:

   0. operatorlib.ReadProxyVarsFromEnv() // Helper function
   1. operatorlib.AddProxyToDeployment(dep *appsv1.Deployment)
   2. operatorlib.AddProxyToDaemonSets(daem *appsv1.DaemonSet)
   3. operatorlib.AddProxyToReplicaSets(rep *appsv1.ReplicaSet)
   4. operatorlib.AddProxyToPods(pod *appsv1.Pod)

Simplified Code:

```go
// Read environment variables for "HTTP_PROXY, HTTPS_PROXY, and
// NO_PROXY"
func ReadProxyVarsFromEnv() ([]corev1.EnvVar) {
   httpProxyEnvVar := "HTTP_PROXY"
   httpproxy := os.Getenv("HTTP_PROXY")
   return []corev1.EnvVar{
       {
           Name:  httpProxyEnvVar,
           Value: httpproxy,
       },
       {
           Name:  strings.ToLower(httpProxyEnvVar),
           Value: httpproxy,
       },
    }
}

func AddProxyToDeployment(dep *appsv1.Deployment) {
    // Create []corev1.EnvVar from Operator environment
    proxyVars := ReadProxyVarsFromEnv()
    // Pass proxy vars to each container
   for _, container := range dep.Spec.Template.Spec.Containers {
       container.Env = append(container.Env, proxyVars)
   }
}
```


Example usage with the [memcached operator](https://github.com/operator-framework/operator-sdk/blob/master/testdata/go/v3/memcached-operator/controllers/memcached_controller.go#L83)

```go
dep := r.deploymentForMemcached(memcached)
operatorlib.AddProxyToDeployment(dep)
```


For Ansible-based operators, the simplest way is to lookup and pass the
environment variables directly where the objects are created by Operator
authors. For example, the Ansible memcached `deployment.yaml` would
include:
```yaml
   env:
      HTTPS_PROXY: {{ lookup('env', 'HTTPS_PROXY') | default('', True) }}'
```

For Helm-based operators, environment variables can be set as helm
variables using the watches file, and passed along to operands using
templates.

```yaml
- group: example.com
  version: v1alpha1
  kind: Nginx
  chart: helm-charts/nginx
  overrideValues:
    httpsProxy: $HTTP_PROXY
```

### User Stories

#### Ansible

As an ansible-based operator author, I have documentation and an example of
passing proxy environment variables to my operands.

#### Helm

As a helm-based operator author, I have documentation and an example of
passing proxy environment variables to my operands.

#### Golang

As a go-based operator author, I have functions that read and
pass proxy environment variables to various operand types as well as
documentation for how to use them.


## Risks

1. We may not have captured all of the go utility methods, we may need to
add more helpers later.
