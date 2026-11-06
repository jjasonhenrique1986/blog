---
title: "Getting to know the Octant tool"
slug: "2023-01-09-getting-to-know-the-octant-tool"
description: "Get to know Octant, an open source project that lets you view and manage a Kubernetes cluster through a graphical interface."
images: ["/images/capa-conhecendo-a-ferramenta-octant.png"]
author: Jason
date: 2023-01-09
lastmod: 2026-11-04
tags: [ "Kubernetes", "Observability", "Containers" ]
type: post
---

> **Update (November 2026):** Octant was discontinued and its
> repository was archived in January 2023 (now at
> [vmware-archive/octant](https://github.com/vmware-archive/octant)), so
> it no longer receives fixes. For a Kubernetes graphical interface, see
> [Headlamp](https://github.com/kubernetes-sigs/headlamp), a
> kubernetes-sigs project, or [k9s](https://github.com/derailed/k9s), in the terminal.

Octant is an open source project that lets you view
your Kubernetes cluster through a graphical interface, so you can do all the
management of the cluster and the applications through it.

Octant was maintained by VMware. Below are links with more information:

- https://octant.dev/
- https://github.com/vmware-archive/octant

## Prerequisites

To test this tool you need a local Kubernetes
cluster. You can use kind or Rancher Desktop for testing. Below are
links with more information:

- https://kind.sigs.k8s.io/ (see also [how to create a local cluster with kind]({{< relref "/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind" >}}))
- https://rancherdesktop.io/

You also need kubeconfig to be configured, since the
tool reads information from that file, as in the image below:

![Octant preferences screen showing the path of the kubeconfig in use](/images/octant-2.webp)

For more information, see the [kubeconfig
documentation](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/).

## Installation

To use it, you need to install a client on your local machine,
which supports Windows, Linux and Mac.

To download it, just go to the link below:

-  https://github.com/vmware-archive/octant/releases/tag/v0.25.1

At the time of writing, this is the tool's latest version.

## Overview

This section gives an overview of the tool.

![Octant Workloads screen with the Kubecost deployments and the memory and CPU usage of each one](/images/octant-3.webp)

In the image above we can see all the workloads
running in a given namespace, as well as the CPU and
memory usage of each workload.

![Octant showing the kubecost-cost-analyzer deployment and the graph of related resources](/images/octant-4.webp)

![Octant Resource Viewer tab with the graph of the kubecost-cost-analyzer deployment, replicaset, pods and configmaps](/images/octant-5.webp)

Above are other examples of views.

It lets you see the Kubernetes objects a given
workload is using. Below are some examples:

![Overview of the kubecost namespace in Octant with the list of deployments and pods](/images/octant-6.webp)

![List of services in the kubecost namespace in Octant, with type, IP and ports](/images/octant-7.webp)

![Summary tab of the kubecost-cost-analyzer deployment in Octant with configuration, status and pods](/images/octant-8.webp)

You can set up a port-forward with a single click

![Deployment pod template in Octant with the cost-model container and the port forward buttons for ports 9003 and 9090](/images/octant-9.png)

It shows the environment variables

![Container environment variables listed in Octant, below the ports with port forward](/images/octant-10.webp)

Viewing the YAML used by the deployment

![YAML tab of the kubecost-cost-analyzer deployment in Octant](/images/octant-11.webp)

Viewing logs and connecting to the pod through the tool

![Logs tab of a kubecost-cost-analyzer pod in Octant](/images/octant-12.webp)

![Terminal tab of a kubecost-cost-analyzer pod in Octant, with a shell open in the container](/images/octant-13.webp)

You can see the events, as in the image below:

![Events screen of a namespace in Octant](/images/octant-14.webp)

It lets you apply YAML through the tool

![Octant Apply YAML window to apply manifests to the cluster](/images/octant-15.webp)

You can also integrate the tool with other plugins, as in the link
below:

- https://github.com/topics/octant-plugin

## Conclusion

The goal of this post was to show how to use the Octant tool and
how it can help manage a Kubernetes cluster through a
graphical interface, increasing your productivity.

There are similar tools, such as Headlamp, k9s (in the terminal) and
Lens. Kubernetes Dashboard, which I used to mention here, was also
[archived](https://github.com/kubernetes-retired/dashboard). Below are links
with more information:

- [Headlamp](https://github.com/kubernetes-sigs/headlamp)
- [k9s](https://github.com/derailed/k9s)
- [Lens](https://k8slens.dev/)
