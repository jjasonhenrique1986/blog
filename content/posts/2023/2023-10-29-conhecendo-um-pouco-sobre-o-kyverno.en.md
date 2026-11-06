---
title: "Getting to know Kyverno"
slug: "2023-10-29-getting-to-know-kyverno"
description: "An introduction to Kyverno, a Kubernetes dynamic admission controller: installation and creating a validation policy."
images: ["/images/capa-conhecendo-um-pouco-sobre-o-kyverno.png"]
author: Jason
date: 2023-10-29
tags: [ "Kubernetes", "Security", "Containers" ]
type: post
---

The goal of this article is to show how to use the Kyverno tool
in a Kubernetes environment. Kyverno is a Kubernetes dynamic admission
controller that lets you apply policies
inside your cluster.

In the links below you can find more information about this tool:

-   <https://kyverno.io/>
-   <https://github.com/kyverno/kyverno/>

## Prerequisites

To test Kyverno, these are the prerequisites:

-   A Kubernetes cluster. For that, you can use
    [kind]({{< relref "/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind" >}}) or [Rancher
    Desktop](https://rancherdesktop.io/).
-   [kubectl installed](https://kubernetes.io/docs/tasks/tools/).
-   [Helm installed](https://helm.sh/docs/intro/install/).

## Installation

You can install Kyverno in two ways: one using
Helm and the other applying a Kubernetes manifest with the
kubectl command.

For both installation methods, it's recommended to create a namespace
for Kyverno with the command below:

``` bash
kubectl create ns kyverno
```

## Installing with Helm

To install Kyverno with Helm, run the
commands below:

``` bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno -n kyverno
```

The same repository also has the `kyverno/kyverno-policies` chart, which
installs a set of ready-made policies on top of an existing Kyverno installation.

## Installing with kubectl

To install with kubectl, run the command below:

``` bash
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.10.0/install.yaml -n kyverno
```

## Kyverno concepts

After installing Kyverno, you'll have two kinds available
to create your policies:

-   **ClusterPolicy** -- which can be applied to the whole Kubernetes cluster.
-   **Policy** -- which can be applied to a specific namespace.

There are also two types of policies you can create:

-   **Validation** -- where the policy validates something, for example whether
    a deployment has a certain label.
-   **Mutation** -- where the policy changes a certain setting of
    a deployment.

Within these two types of policies, there are two actions that can be
taken:

-   **Audit** -- where it's only logged whether a given
    deployment follows a given Kyverno policy or not.
-   **Enforce** -- if a deployment doesn't meet a given
    Kyverno policy, it won't be created.

## Testing a Validation policy

To test a validation policy, let's create the ClusterPolicy
below, which checks whether the owner and environment labels exist, first
using the **Audit** action and then the **Enforce** action.

``` yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
  annotations:
    policies.kyverno.io/title: Require Labels
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/minversion: 1.6.0
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: Pod, Label
    policies.kyverno.io/description: >-
      This policy validates that the label `owner` and `environment` is specified with some value.      
spec:
  validationFailureAction: Audit
  background: true
  rules:
  - name: check-for-labels
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "The label `owner` and `environment` is required."
      pattern:
        metadata:
          labels:
            owner: "?*"
            environment: "?*"
```

To apply the clusterpolicy above, create a file called
**require-labels.yaml** and run the command below:

``` bash
kubectl apply -f require-labels.yaml
```

To check whether the policy was created, run the command below:

``` bash
kubectl get clusterpolicy
```

It should show output similar to the image below:

![kubectl output showing the require-labels policy in Audit mode and with Ready status](/images/kyverno-2.png)

To test it, let's create a deployment file without the owner and
environment labels, as in the file below:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: app
  name: app
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: app
    spec:
      containers:
      - image: nginx
        name: nginx
```

To apply the deployment above, just create the nginx.yaml file and
run the command below:

``` bash
kubectl apply -f nginx.yaml
```

To see whether or not the deployment meets the policy we created, we can
run the following commands:

``` bash
kubectl describe clusterpolicy require-labels
kubectl get events -n app
```

Where **app** is the namespace where the deployment was applied. The events
should show output similar to the image below:

![PolicyViolation events generated by Kyverno because the deployment and the pod don't have the owner and environment labels](/images/kyverno-3.webp)

Now let's change the clusterpolicy so that the owner and environment labels
are mandatory. If these labels aren't present,
the deployment won't be created.

Before changing the clusterpolicy, delete the deployment with the command
below:

``` bash
kubectl delete -f nginx.yaml
```

To switch the clusterpolicy to **Enforce**, change the
setting below from Audit to Enforce, as below:

``` yaml
spec:
  validationFailureAction: Enforce
```

Then save the file and apply the policy again with the
command below:

``` bash
kubectl apply -f require-labels.yaml
```

After that, try to apply the deployment again with
kubectl. You should see the following error message, as in the image
below:

![Kyverno admission webhook error blocking the creation of the deployment without the owner and environment labels](/images/kyverno-4.webp)

To fix this error, add the owner and
environment labels, as below:

``` yaml
  template:
    metadata:
      labels:
        app: app
        owner: jason
        environment: develop
```

## Conclusion

The goal of this article was to show how to use the
Kyverno tool. In the next article I show how to use Kyverno with
a Mutation policy:
[Creating a Mutation policy with Kyverno]({{< relref "/posts/2023/2023-11-02-criando-uma-policy-do-tipo-mutation-com-o-kyverno" >}}).

If you want to go deeper into the tool, the links below can help:

-   <https://playground.kyverno.io/#/>
-   <https://kyverno.io/resources/>
-   <https://kyverno.io/policies/>
