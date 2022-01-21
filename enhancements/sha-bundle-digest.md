---
title: SHA-Bundle-Digest
authors:
  - '@theishshah'
reviewers:
  - '@camilamacedo86'
  - '@joelanford'
  - '@jmrodri'
  - '@ryantking'
  - '@tlwu2013'
approvers:
  - '@tlwu2013'
  - '@jmrodri'
  - '@joelanford'

creation-date: 2022-01-14

last-updated: 2022-01-14

status: implementable
---

# Support SHA-based Image Tags

## Summary

Operators built with `operator-sdk` use Docker images that are referenced via an image URL and tag name. This EP adds
the ability to create operators that use image digests instead of tags for operator images as well as related images. An
image digest is a `SHA-256` sum that is created by hashing the contents of a Docker image manifest. A Docker image
manifest is a JSON file that defines the image, most notably its layers. See [Appendix A](#appendix-a) for an example
manifest for `golang:1.17.6`. An alternative method to specify an image is to use its digest directly instead of a tag.
For example, an image for go 1.17.6 would be pulled with the Docker command `docker pull golang:1.17.6`. The exact same
image can be pulled by its digest as well:
`docker pull golang@sha256:197db40b76cce7e70d59e59cc567896395730c6491e7b3da5d1f4c22efa72404`. An important distinction
is that since digests are a hash of the image manifests, any change to the image, will cause it to have a distinct
digest. This means that if a new image is tagged as `golang:1.17.6`, the first `docker pull` command will pull that new
image while the second `docker pull` command will return the same image as before since it uses the digest to reference
it.

## Motivation

Many users run Kubernetes in an air-gapped environment (also known as an offline or disconnected environment), which is
when a cluster network does not have any access to the public internet, usually for security purposes. In this setup,
since the image is hosted in a registry within the air-gapped environment, it will have a different name. For example,
if you push `golang:1.17.6` to an internal registry, it will have a name along the lines of
`registry.internaldomain.net/library/golang:1.17.6`. Since the name, which is part of the image manifest, is different,
the image digest will be different. This is a problem for security-conscience users who want to ensure that they are
using the exact image they expect since by definition the digests of the same image in two different registries will be
different. This EP resolves that by supporting an operator configuration that works directly with image digests instead
of tags. Users will then be able to push images to their internal registry using the image's digest instead of its tag,
which will keep the digest from changing, and then the operator will pull the image based on its digest.

### Goals

- Give users the ability to enable digest-based image resolution.
- Allow users to specify related images to resolve digests for.
- Use image digests in the output `ClusterServiceVersion` (see [Appendix B](#appendix-b)).
  - Images specified in deployments will have digests.
  - The `spec.relatedImage` section of the CSV will contain a mapping of images to their digests.

### Non-Goals

- Provide an automated way to migrate existing Operator-SDK-based operators from image tags to image digests.
- Identify locations in user code that reference images by tag.

## Proposal

- Add `--use-image-digests` to `operator-sdk generate bundle` command. When enabled, the CLI will:
  1. Search the environment defined in `manager.yaml` for all variables that begin with `RELATED_IMAGE_` to build a list
     of all related images that need to be added to the CSV (see [Appendix C](#appendix-c)).
  2. Resolve the digests for the operator image and any additional related images it finds in the environment.
  3. Output the operator image(s) with digests instead of tags in the CSV.
  4. Add related images with their digests to the CSV.
- Add a variable to the `Makefile` called `USE_IMAGE_DIGESTS`.

### Risks and Mitigations

Since the feature is opt-in, it carries the risk of the end-user enabling it without a correct understanding of the
feature and its implications. This can be mitigated with clear documentation both on the website and in the CLI help
section.

## Design Details

- As of this EP, the new `--use-image-digests` is the only flag to pass to the `operator-sdk generate bundle` command.
  In order to allow users full control over the command's flags, the `Makefile` should use an intermediary variable to
  hold the flags that a user can overwrite entirely. For example:

```Makefile
GENERATE_BUNDLE_FLAGS ?= ""
USE_IMAGE_DIGESTS ?= false
ifeq ($(GENERATE_BUNDLE_FLAGS),"")
    ifeq($(USE_IMAGE_DIGESTS),true)
        GENERATE_BUNDLE_FLAGS += "--use-image-digests"
    endif
endif
bundle:
    operator-sdk generate bundle $(GENERATE_BUNDLE_FLAGS)
```

- The `RELATED_IMAGE_*` variables should be defined in the `manager.yaml` file as outlined
  [here](https://master.sdk.operatorframework.io/docs/best-practices/common-recommendation/#other-common-suggestions).
- In order to resolve digests from tags, the CLI will use the `imageresolver` package in
  [operator-manifest-tools](https://github.com/operator-framework/operator-manifest-tools).
- The output cluster CSV will have the `relatedImage` annotation that is explained in the
  [OpenShift Documentation](https://docs.openshift.com/container-platform/4.9/operators/operator_sdk/osdk-generating-csvs.html#olm-enabling-operator-for-restricted-network_osdk-generating-csvs).

### Test Plan

The resulting artifact of this EP will be a CSV with image digests instead of tags in the image definitions as well as
digest annotations. This feature will need an integration test that verifies that the knobs we provide to enable and
configure image digests generate an expected CSV. The expected CSV will be based on the example memcached CSV with the
proposed changes (see [Appendix B](#appendix-b)).

## Implementation History

There has been no implementation to this point.

## Drawbacks

The original motivation for this feature is used in a disconnected environment with an SDK-built operator, OLM, and OCP.
Performing a full E2E test in a complex environment that will need to be specifically built out for this task is out of
scope for this SDK EP.

## Alternatives

We are not considering any alternative solutions at this time.

## Open Questions

- Operator-manifest-tools relies on an external binary, skopio. In order to remove the external binary, can we PR
  operator-manifest-tools to replace the calls to the skopeo CLI with library calls?
  - Non-blocking question, a future improvment will be to add library support instead of using an external binary.

## Appendix A

The following is an example of an image manifest for `golang:1.17.6`:

```json
[
  {
    "Config": "8b86bf336a01235faf28137dae90772076e6f431a2951259d949eb9153012755.json",
    "RepoTags": ["golang:1.17.6"],
    "Layers": [
      "644b6e1dc104cf7c01e3d906bad42ca6386e62e03ba55c23c9229eeca2aedf4e/layer.tar",
      "dbaefc91c060fe8e827580d1018824c5479bb8eeadf644b44f9bbdc6cc0ad314/layer.tar",
      "7164fcc370aec09c29fb107f4b404d667a046d7eb652130cfcad3b493d7f9485/layer.tar",
      "ded273d40fe1a66f4cc0e6cc15236a8800c4e465df4df2b682422a5477910951/layer.tar",
      "eac35aa9198a17ca6942cda2efacf8b052fcfd7c9046cace7290a0b80bd43d7b/layer.tar",
      "34cd146535bd6d8585e4bc411593bc82a336a335fbcc0a1cf78b1ecfe635c549/layer.tar",
      "0aafe4d111d6dbd2414e183de18f5736f470de5b0e750b05bbe4063f00366386/layer.tar"
    ]
  }
]
```

The manifest is built out of a list of layers, repo tags, and a config file. Since the config file and layers themselves
are hashed, if any of them change or the repo tags, the manifest digest will change.

## Appendix B

A `ClusterServiceVersion` is a [CRD](https://olm.operatorframework.io/docs/concepts/crds/clusterserviceversion/) defined by [Operator Lifecycle Manager](https://github.com/operator-framework/operator-lifecycle-manager) that defines metadata about the entire operator.

````

An example CSV for the Memcached example operator is available [here](https://github.com/operator-framework/operator-sdk/blob/master/testdata/go/v3/memcached-operator/bundle/manifests/memcached-operator.clusterserviceversion.yaml).

This EP will change two things on the CSV: use digests in deployment container image locations and add the `spec.relatedImages` section with all other images, such as base images, that need to have a digest resolved from a tag. An abbreviated version of the CSV with the aforementioned changes looks like this:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
metadata:
  # ...
spec:
  # ...
  install:
    spec:
      # ...
      deployments:
        - name: memcached-operator-controller-manager
          spec:
            # ...
            template:
              # ...
              spec:
                containers:
                  - args:
                    # ...
                    image: gcr.io/kubebuilder/kube-rbac-proxy@sha256:e10d1d982dd653db74ca87a1d1ad017bc5ef1aeb651bdea089debf16485b080b
                    # ...
                  - args:
                    # ...
                    image: quay.io/example/memcached-operator@sha256:d044b878fbcbd0c06c2bcc050f3f9ca070ffb089e6bac01accddcd0ba85b7b9e
                    # ...
      # ...
    # ...
  # ...
  relatedImages:
    - name: golang:1.17
      image: golang:8d717e8a7fa8035f5cfdcdc86811ffd53b7bb17542f419f2a121c4c7533d29ee
    - name: gcr.io/distroless/static:nonroot
      image: gcr.io/distroless/static:741704ac0e5e6ff758a8c46f0f028375252fdf147248d95514ce11ff57fdece9
````

## Appendix C

Example `manager.yaml` containing correctly populated RELATED IMAGE environment variables.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  # ...
spec:
    # ...
    template:
    # ...
    spec:
        containers:
            env:
                name:  RELATED_IMAGE_single_host_gateway
                value: quay.io/eclipse/che--traefik:v2.5.0-eb30f9f09a65cee1fab5ef9c64cb4ec91b800dc3fdd738d62a9d4334f0114683
        # ...
        - args:
            # ...
        # ...
      # ...
    # ...
  # ...
```
