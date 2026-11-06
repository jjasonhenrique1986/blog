---
title: "Creating a local Kubernetes cluster with kind"
slug: "2023-11-15-creating-a-local-kubernetes-cluster-with-kind"
description: "Step by step to create a local Kubernetes cluster with kind for testing and studying."
images: ["/images/capa-criando-um-cluster-local-de-kubernetes-com-o-kind.png"]
author: Jason
date: 2023-11-15
tags: [ "Kubernetes", "Containers" ]
type: post
---

The goal of this article is to walk through how to run a
Kubernetes cluster locally with kind, so you can run
tests or study Kubernetes.

Keep in mind that this tutorial focuses on creating a local Kubernetes
cluster on a Linux machine (in my case, a Debian distribution).

## Installing Docker

The first step is to install Docker so we can run kind.
To install Docker, run the command below:

``` bash
curl -fsSL https://get.docker.com | sh
```

To check whether Docker was installed successfully, run the command
below:

``` bash
docker 
```

## Installing kubectl

We need to install the kubectl command-line utility
to manage the Kubernetes cluster from the command line.
Below are the commands to run:

``` bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
```

To validate the kubectl installation, run the command below:

``` bash
kubectl
```

## Installing kind

To install kind, run the commands below:

``` bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/
```

To check the kind installation, run the command below:

``` bash
kind 
```

## Creating the Kubernetes cluster

Now let's create a Kubernetes cluster locally
with kind. I'll create the cluster declaratively
using a YAML file, as below:

``` yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cluster-name
runtimeConfig:
  "api/alpha": "false"
nodes:
- role: control-plane
- role: worker
- role: worker
```

Create a file called **create_cluster.yaml**
with the content above and run the command below:

``` bash
kind create cluster --config create_cluster.yaml
```

You should see output like the one below:

![kind create cluster output creating the cluster-jason cluster and setting the kubectl context](/images/kind-2.webp)

To check whether the cluster was created correctly, run the commands
below:

```bash
kubectl get nodes
kubectl get pods -A
```

To delete the cluster, just run the command below:

```bash
kind delete clusters cluster-name
```

## Installing Helm

Now let's install Helm, a Kubernetes package manager
for installing applications. First, download the
latest version's package from the [Helm releases on
GitHub](https://github.com/helm/helm/releases).

Then run the commands below:

```bash 
tar -zxvf package-name
sudo mv linux-amd64/helm /usr/local/bin
```

To check whether Helm was installed successfully, run the command
below:

```bash
helm
```

## Conclusion

The goal of this article was to create a tutorial for creating a
local Kubernetes cluster along with some essential packages to
manage the cluster. Below are some links to help you if you have
questions:

-   <https://helm.sh/docs/intro/install/>
-   <https://kind.sigs.k8s.io/docs/user/quick-start/>
-   <https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/>
