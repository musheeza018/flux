---
title: Core Concepts
description: Core Concepts of Flux.
weight: 10
---

These are some core concepts in Flux.

## GitOps

GitOps is a way of managing your infrastructure and applications so that whole system
is described declaratively and version controlled (most likely in a Git repository),
and having an automated process that ensures that the deployed environment
matches the state specified in a repository.

For more information, take a look at ["What is GitOps?"](https://www.gitops.tech/#what-is-gitops).

## Sources

A *Source* defines the origin of a repository containing the desired state of 
the system and the requirements to obtain it (e.g. credentials, version selectors). 
For example, the latest `1.x` tag available from a Git repository over SSH.

Sources produce an artifact that is consumed by other Flux components to perform
actions, like applying the contents of the artifact on the cluster. A source
may be shared by multiple consumers to deduplicate configuration and/or storage.

The origin of the source is checked for changes on a defined interval, if
there is a newer version available that matches the criteria, a new artifact
is produced.

All sources are specified as Custom Resources in a Kubernetes cluster, examples
of sources are `GitRepository`, `OCIRepository`, `HelmRepository` and `Bucket` resources. 

For more information, take a look at
[the source controller documentation](components/source/_index.md).

## Reconciliation

Reconciliation refers to ensuring that a given state (e.g. application running in the cluster, infrastructure)
matches a desired state declaratively defined somewhere (e.g. a Git repository).

There are various examples of these in Flux:

- `HelmRelease` reconciliation: ensures the state of the Helm release matches what is defined in the resource,
  performs a release if this is not the case (including revision changes of a HelmChart resource).
- `Bucket` reconciliation: downloads and archives the contents of the declared bucket on a given
  interval and stores this as an artifact, records the observed revision of the artifact
  and the artifact itself in the status of resource.
- `Kustomization` reconciliation: ensures the state of the application
  deployed on a cluster matches the resources defined in a Git or OCI repository or S3 bucket.

## Kustomization

The `Kustomization` custom resource represents a local set of Kubernetes resources
(e.g. kustomize overlay) that Flux is supposed to reconcile in the cluster.
The reconciliation runs every five minutes by default, but this can be changed with `.spec.interval`.
If you make any changes to the cluster using `kubectl edit/patch/delete`,
they will be promptly reverted. You either suspend the reconciliation or push your changes to a Git repository.

For more information, take a look at the [Kustomize FAQ](faq.md#kustomize-questions)
and the [Kustomization CRD](components/kustomize/kustomization.md).

## Bootstrap

The process of installing the Flux components in a GitOps manner is called a bootstrap.
The manifests are applied to the cluster, a `GitRepository` and `Kustomization`
are created for the Flux components, then the manifests are pushed to an existing Git repository
(or a new one is created). Flux can manage itself just as it manages other resources.
The bootstrap is done using the `flux` CLI or
using our [Terraform Provider](https://github.com/fluxcd/terraform-provider-flux).

For more information, take a look at [the bootstrap documentation](installation.md#bootstrap).

## Continuous Delivery

Continuous Delivery refers to the practice of delivering software updates frequently and reliably. 

For more information, take a look at continuous delivery as defined in the [CNCF](https://glossary.cncf.io/continuous-delivery/).

## Continuous Deployment

Continuous Deployment is the practice of automatically deploying code changes to production once they have passed through automated testing. 

For more information, take a look at continuous delivery as defined in the [CNCF Glossary](https://glossary.cncf.io/continuous-delivery/).

## Progressive Delivery

Progressive Delivery builds on Continuous Delivery by gradually rolling out new features or updates to a subset of users, allowing developers to test and monitor the new features in a controlled environment and make necessary adjustments before releasing them to everyone.

Developers can use techniques like feature flags, [canary releases](https://glossary.cncf.io/canary-deployment/), and A/B testing to minimize the chances of introducing bugs or errors that could harm users or interrupt business operations. These strategies enable a controlled and gradual rollout of new features, ensuring a smooth and successful release that enhances user trust and improves the overall user experience.

The Flux project offers a specialised controller called [Flagger](https://github.com/fluxcd/flagger) that implements various progressive delivery techniques. For more information, take a look at [Flagger deployment strategies](https://fluxcd.io/flagger/usage/deployment-strategies/).

## Getting started with Flux

This tutorial shows you how to bootstrap Flux to a Kubernetes cluster and deploy a sample application in a GitOps manner.

## Before you begin

To follow the guide, you need the following:

- **A Kubernetes cluster**. We recommend [Kubernetes kind](https://kind.sigs.k8s.io/docs/user/quick-start/) for trying Flux out in a local development environment.
- **A GitHub personal access token with repo permissions**. See the GitHub documentation on [creating a personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line).

## Objectives

- Bootstrap Flux on a Kubernetes Cluster.
- Deploy a sample application using Flux.
- Customize application configuration through Kustomize patches.

## Install the Flux CLI

The `flux` command-line interface (CLI) is used to bootstrap and interact with Flux.

To install the CLI with Homebrew run:

```sh
brew install fluxcd/tap/flux
```

For other installation methods, see the [CLI install documentation](installation.md#install-the-flux-cli).

## Export your credentials

Export your GitHub personal access token and username:

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

## Check your Kubernetes cluster

Check you have everything needed to run Flux by running the following command:

```bash
flux check --pre
```

The output is similar to:

```
► checking prerequisites
✔ kubernetes 1.22.2 >=1.20.6
✔ prerequisites checks passed
```

## Install Flux onto your cluster

For information on how to bootstrap using a GitHub org, Gitlab and other git providers, see [Installation](installation.md).

Run the bootstrap command:

```sh
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal
```

The output is similar to:

```
► connecting to github.com
✔ repository created
✔ repository cloned
✚ generating manifests
✔ components manifests pushed
► installing components in flux-system namespace
deployment "source-controller" successfully rolled out
deployment "kustomize-controller" successfully rolled out
deployment "helm-controller" successfully rolled out
deployment "notification-controller" successfully rolled out
✔ install completed
► configuring deploy key
✔ deploy key configured
► generating sync manifests
✔ sync manifests pushed
► applying sync manifests
◎ waiting for cluster sync
✔ bootstrap finished
```

The bootstrap command above does the following:

- Creates a git repository `fleet-infra` on your GitHub account.
- Adds Flux component manifests to the repository.
- Deploys Flux Components to your Kubernetes Cluster.
- Configures Flux components to track the path `/clusters/my-cluster/` in the repository.

## Clone the git repository

Clone the `fleet-infra` repository to your local machine:

```sh
git clone https://github.com/$GITHUB_USER/fleet-infra
cd fleet-infra
```

## Add podinfo repository to Flux

This example uses a public repository [github.com/stefanprodan/podinfo](https://github.com/stefanprodan/podinfo),
podinfo is a tiny web application made with Go.

1. Create a [GitRepository](../components/source/gitrepositories/) manifest pointing to podinfo repository's master branch:

    ```sh
    flux create source git podinfo \
      --url=https://github.com/stefanprodan/podinfo \
      --branch=master \
      --interval=30s \
      --export > ./clusters/my-cluster/podinfo-source.yaml
    ```

    The output is similar to:

    ```yaml
    apiVersion: source.toolkit.fluxcd.io/v1
    kind: GitRepository
    metadata:
      name: podinfo
      namespace: flux-system
    spec:
      interval: 30s
      ref:
        branch: master
      url: https://github.com/stefanprodan/podinfo
    ```

2. Commit and push the `podinfo-source.yaml` file to the `fleet-infra` repository:

    ```sh
    git add -A && git commit -m "Add podinfo GitRepository"
    git push
    ```

## Deploy podinfo application

Configure Flux to build and apply the [kustomize](https://github.com/stefanprodan/podinfo/tree/master/kustomize)
directory located in the podinfo repository.

1. Use the `flux create` command to create a [Kustomization](../components/kustomize/kustomization/) that applies the podinfo deployment.

    ```sh
    flux create kustomization podinfo \
      --target-namespace=default \
      --source=podinfo \
      --path="./kustomize" \
      --prune=true \
      --interval=5m \
      --export > ./clusters/my-cluster/podinfo-kustomization.yaml
    ```

    The output is similar to:

    ```yaml
    apiVersion: kustomize.toolkit.fluxcd.io/v1
    kind: Kustomization
    metadata:
      name: podinfo
      namespace: flux-system
    spec:
      interval: 5m0s
      path: ./kustomize
      prune: true
      sourceRef:
        kind: GitRepository
        name: podinfo
      targetNamespace: default
    ```

2. Commit and push the `Kustomization` manifest to the repository:

    ```sh
    git add -A && git commit -m "Add podinfo Kustomization"
    git push
    ```

    The structure of the `fleet-infra` repo should be similar to:

      ```
      fleet-infra
      └── clusters/
          └── my-cluster/
              ├── flux-system/                        
              │   ├── gotk-components.yaml
              │   ├── gotk-sync.yaml
              │   └── kustomization.yaml
              ├── podinfo-kustomization.yaml
              └── podinfo-source.yaml
      ```

## Watch Flux sync the application

1. Use the `flux get` command to watch the podinfo app.

    ```sh
    flux get kustomizations --watch
    ```

    The output is similar to:

    ```
    NAME          REVISION             SUSPENDED  READY   MESSAGE
    flux-system   main@sha1:4e9c917f   False      True    Applied revision: main@sha1:4e9c917f
    podinfo       master@sha1:44157ecd False      True    Applied revision: master@sha1:44157ecd
    ```

2. Check podinfo has been deployed on your cluster:

    ```sh
    kubectl -n default get deployments,services
    ```

    The output is similar to:

    ```
    NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/podinfo   2/2     2            2           108s

    NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
    service/podinfo      ClusterIP   10.100.149.126   <none>        9898/TCP,9999/TCP   108s
    ```

Changes made to the podinfo
Kubernetes manifests in the master branch are reflected in your cluster.

When a Kubernetes manifest is removed from the podinfo repository, Flux removes it from your cluster.
When you delete a `Kustomization` from the `fleet-infra` repository, Flux removes all Kubernetes objects previously applied from that `Kustomization`.

When you alter the podinfo deployment using `kubectl edit`, the changes are reverted to match
the state described in Git.

## Suspend updates

Suspending updates to a kustomization allows you to directly edit objects applied from a kustomization, without your changes being reverted by the state in Git.

To suspend updates for a kustomization, run the command `flux suspend kustomization <name>`.

To resume updates run the command `flux resume kustomization <name>`.

## Customize podinfo deployment

To customize a deployment from a repository you don't control, you can use Flux
[in-line patches](../components/kustomize/kustomization/#override-kustomize-config). The following example shows how to use in-line patches to change the podinfo deployment.

1. Add the following to the field `spec` of your `podinfo-kustomization.yaml` file:

    ```yaml clusters/my-cluster/podinfo-kustomization.yaml
      patches:
        - patch: |-
            apiVersion: autoscaling/v2beta2
            kind: HorizontalPodAutoscaler
            metadata:
              name: podinfo
            spec:
              minReplicas: 3     
          target:
            name: podinfo
            kind: HorizontalPodAutoscaler
    ```

1. Commit and push the `podinfo-kustomization.yaml` changes:

    ```sh
    git add -A && git commit -m "Increase podinfo minimum replicas"
    git push
    ```

After the synchronization finishes, running `kubectl get pods` should display 3 pods.

## Multi-cluster Setup

To use Flux to manage more than one cluster or promote deployments from staging to production, take a look at the 
two approaches in the repositories listed below.

1. [https://github.com/fluxcd/flux2-kustomize-helm-example](https://github.com/fluxcd/flux2-kustomize-helm-example)
2. [https://github.com/fluxcd/flux2-multi-tenancy](https://github.com/fluxcd/flux2-multi-tenancy)

