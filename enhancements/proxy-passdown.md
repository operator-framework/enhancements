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
   2. Add Operator SDK features and documentation to facilitate
      implementation of proxy-aware operators.

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

For Go-based operators, a helper function `ReadProxyVarsFromEnv()` will
be created in operator-lib to read the proxy variables from the Operator
environment with documentation on how to inject these variables into
each of the container specs. Operator authors will be responsible for
invoking this function and adding the variables to container specs. The
documentation for this will live in the godoc, which should include
example usage.

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
```


Example usage with the [memcached operator](https://github.com/operator-framework/operator-sdk/blob/master/testdata/go/v3/memcached-operator/controllers/memcached_controller.go#L83)

```go
// Usage with appsv1 Objects
dep := r.deploymentForMemcached(memcached)
proxyVars := ReadProxyVarsFromEnv()
for _, container := range dep.Spec.Template.Spec.Containers {
    container.Env = append(container.Env, proxyVars)
}
```

For Ansible-based operators, the simplest way is to lookup and pass the
environment variables directly where the objects are created by Operator
authors. Using Ansible-based memcached for example, the task would become:
```yaml
---
- name: start memcached
  community.kubernetes.k8s:
    definition:
      kind: Deployment
      apiVersion: apps/v1
      metadata:
        name: '{{ ansible_operator_meta.name }}-memcached'
        namespace: '{{ ansible_operator_meta.namespace }}'
      spec:
        replicas: "{{size}}"
        selector:
          matchLabels:
            app: memcached
        template:
          metadata:
            labels:
              app: memcached
          spec:
            containers:
            - name: memcached
              command:
              - memcached
              - -m=64
              - -o
              - modern
              - -v
              image: "docker.io/memcached:1.4.36-alpine"
              ports:
                - containerPort: 11211
              env:
                 - Name: HTTPS_PROXY: 
                   Value: "{{ lookup('env', 'HTTPS_PROXY') | default('', True) }}"
                 - Name: https_proxy: 
                   Value: "{{ lookup('env', 'HTTPS_PROXY') | default('', True) }}"
                 - Name: HTTP_PROXY: 
                   Value: "{{ lookup('env', 'HTTP_PROXY') | default('', True) }}"
                 - Name: http_proxy: 
                   Value: "{{ lookup('env', 'HTTP_PROXY') | default('', True) }}"
                 - Name: NO_PROXY: 
                   Value: "{{ lookup('env', 'NO_PROXY') | default('', True) }}"
                 - Name: no_proxy: 
                   Value: "{{ lookup('env', 'NO_PROXY') | default('', True) }}"
```


For Helm-based operators, environment variables can be set as helm
variables using the watches file, and passed along to operands in the
templates.

```yaml
- group: example.com
  version: v1alpha1
  kind: Nginx
  chart: helm-charts/nginx
  overrideValues:
    httpsProxy: $HTTPS_PROXY
    httpProxy: $HTTP_PROXY
    noProxy: $NO_PROXY
```

### User Stories

#### Ansible

As an ansible-based operator author, I have documentation and an example of
passing proxy environment variables to my operands.

#### Helm

As a helm-based operator author, I have documentation and an example of
passing proxy environment variables to my operands.

#### Golang

As a go-based operator author, I have a helper function to read proxy
environment variables and docs to show how to pass them to operands in
`kubernetes < v1.22`

Update the godoc and FAQ to include examples using the new Server Side Apply in `kubernetes v1.22`

## Risks

1. If operator-authors consistently implement similar code to set the
   environment variables, we may be able to add more helper functions,
   but they will need to work with Unstructured and Server Side Apply.
