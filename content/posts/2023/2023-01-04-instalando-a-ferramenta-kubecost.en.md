---
title: "Installing the Kubecost tool"
slug: "2023-01-04-installing-the-kubecost-tool"
description: "Step by step installation of Kubecost, a cost monitoring and management tool for Kubernetes and the cloud."
images: ["/images/capa-instalando-a-ferramenta-kubecost.png"]
author: Jason
date: 2023-01-04
lastmod: 2026-11-03
tags: [ "AWS", "FinOps", "Kubernetes" ]
type: post
---

> **Update (November 2026):** Kubecost was acquired by IBM in
> 2024. The official documentation now lives in
> [IBM Docs](https://www.ibm.com/docs/en/kubecost/self-hosted/3.x) and the chart
> is in the [kubecost/kubecost](https://github.com/kubecost/kubecost) repository,
> already at version 3.x. The `kubecost.yaml` manifest used in the installation below
> is no longer available: follow the Helm installation in the official
> documentation. The screens and commands in this post are from the version at the time.

The purpose of this post is to show how to install Kubecost, a cost
monitoring and management tool for a Kubernetes environment and
also for the cloud, supporting the main cloud providers (AWS,
Azure and
GCP).

Kubecost has two kinds of license (free and paid). In the free version, the
collected information is only available for 15 days.

For more information about Kubecost, see the links below:

- https://docs.kubecost.com/
- https://github.com/kubecost/cost-analyzer-helm-chart

## Prerequisites

As a prerequisite, you need a Kubernetes cluster to
install Kubecost. For this demo you can use kind
or Rancher Desktop.

To create a cluster with kind, see the [official kind
website](https://kind.sigs.k8s.io/) or my step by step on
[how to create a local cluster with kind]({{< relref "/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind" >}}).

For Rancher Desktop, see the [official Rancher
Desktop website](https://rancherdesktop.io/).

## Installation

In my scenario, since this is a test environment, I'll install it
with kubectl, but it can also be done with Helm.

Create a namespace called kubecost, where the application will be deployed,
with the command below:

``` bash
kubectl create ns kubecost
```

Run the command below to install it:

``` bash
kubectl apply -f https://raw.githubusercontent.com/kubecost/cost-analyzer-helm-chart/master/kubecost.yaml -n kubecost
```

The following objects will be created, as in the image below:

![kubectl apply output creating the Kubecost objects in the kubecost namespace](/images/kubecost-2.webp)

Below is an overview of the objects created by Kubecost:

![kubectl get all output in the kubecost namespace with pods, services, daemonset and deployments running](/images/kubecost-3.webp)

The default installation includes a prometheus-server, which is responsible for
collecting information from the cluster. You can use your own
Prometheus server, but it's recommended to use this separate
Prometheus.

## Accessing the Kubecost console

To access the Kubecost console we'll use the kubectl
port-forward command to test locally. Here's the command:

``` bash
 kubectl port-forward service/kubecost-cost-analyzer 9090:9090 -n kubecost
```

Below are some screenshots of the Kubecost console:

![Kubecost Overview screen with cluster costs, possible savings and the cost trend chart](/images/kubecost-4.webp)

![Kubecost Overview screen with the cost per namespace and the Cloud Costs Breakdown section](/images/kubecost-5.webp)

## Conclusion

The purpose of this post was to show how to install Kubecost. In the next
post I show the main features of the Kubecost
console: [Overview of the Kubecost tool]({{< relref "/posts/2023/2023-01-07-overview-da-ferramenta-kubecost" >}}).
