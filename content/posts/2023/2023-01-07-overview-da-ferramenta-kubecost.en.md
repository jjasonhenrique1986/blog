---
title: "Overview of the Kubecost tool"
slug: "2023-01-07-overview-of-the-kubecost-tool"
description: "An overview of the Kubecost console for monitoring and managing Kubernetes and cloud provider (AWS, Azure and GCP) costs."
images: ["/images/capa-overview-da-ferramenta-kubecost.png"]
author: Jason
date: 2023-01-07
lastmod: 2026-11-03
tags: [ "AWS", "FinOps", "Kubernetes" ]
type: post
---

> **Update (November 2026):** Kubecost was acquired by IBM in
> 2024. The official documentation now lives in
> [IBM Docs](https://www.ibm.com/docs/en/kubecost/self-hosted/3.x) and the chart
> is in the [kubecost/kubecost](https://github.com/kubecost/kubecost) repository,
> already at version 3.x. The screens and commands in this post are from the version at the time.

The goal of this post is to give an overview of the management console
of Kubecost, a tool that helps manage and monitor the cost
of a Kubernetes environment and also of the cloud provider (AWS,
Azure and GCP) you use.

## Prerequisites

As a prerequisite, you need to have already installed Kubecost.
To install it, follow the post [Installing the Kubecost
tool]({{< relref "/posts/2023/2023-01-04-instalando-a-ferramenta-kubecost" >}}).

## Overview of the Kubecost console

In the sections below I'll give a short overview of the Kubecost
management console.

### Overview section

Below are some screenshots of the Overview section:

![Kubecost Overview section with Kubernetes costs, cloud costs, possible savings and cluster efficiency](/images/kubecost-6.webp)

![Kubecost Overview section with the cost breakdown by namespace and the trend chart by namespace](/images/kubecost-7.webp)

The following information is available:

- Total Kubernetes cost over the last 7 days;
- Total cloud cost over the last 7 days (this requires integrating with
  the cloud provider; see the [Kubecost cloud integration
  documentation](https://docs.kubecost.com/install-and-configure/install/cloud-integration/multi-cloud));
- Total cost over the last 7 days;
- Possible Savings are recommendations that can be applied to the
    environment;
- Efficiency is calculated based on CPU and memory usage
    over a period of time;
- Number of nodes and pods per cluster;
- Namespaces with the highest spend;

### Savings section

Below are some screenshots of the Savings section:

![Kubecost Savings section with the estimated monthly savings and recommendations, such as adjusting container requests and resizing disks](/images/kubecost-8.webp)

![Kubecost recommendation to resize local disks with low utilization](/images/kubecost-9.webp)

The Savings section makes recommendations that save money in the
Kubernetes environment and in the cloud:

- Request/Limit recommendations for CPU and memory;
- Shows unused resources in the cloud;
- Shows abandoned workloads;
- Node disk resizing or node resizing;

### Assets and Allocations sections

Below are some screenshots of the Allocations section:

![Kubecost Allocations section with the accumulated cost by namespace in a chart and a table](/images/kubecost-10.webp)

![Detail of the kubecost namespace in the Allocations section, with cost per resource, CPU and RAM efficiency and suggested savings](/images/kubecost-11.webp)

The Allocations section shows CPU, RAM,
network and load balancer cost information, broken down by namespace.

Screenshots of the Assets section:

![Kubecost Assets section with the cost of the last 7 days grouped by resource type](/images/kubecost-12.webp)

![Kubecost Assets section detailing the cost of the cluster's disks](/images/kubecost-13.webp)

The Assets section shows the following information:

- Costs based on resources, for example Node, Disk,
    Network;
- Data by date (keep in mind that the free version only shows
    15 days of information);
- How much each node or each disk in the environment is costing;

### Health section

Below are screenshots of the Health section:

![Kubecost Health section with a health score of 94 and the list of cluster checks](/images/kubecost-14.webp)

![Continuation of the Kubecost Health section with the reliability, storage and network checks and their severities](/images/kubecost-15.webp)

The Health section runs a best-practices checklist based on
reliability and performance. This checklist produces a score from 0 to
100. Below are some of these checks:

- Whether there are pods with CrashLoopBackOff status;
- Whether there are pods with pending status;
- Pods with low CPU usage;
- Memory Pressure and Disk Pressure;
- Cluster cost increased by more than 10%;
- Out Of Memory events;

### Alerts and Reports sections

The Reports section lets you create reports based on the Asset and
Allocation sections, as in the image below:

![Kubecost Reports section with the saved reports for allocation by namespace and assets by type](/images/kubecost-16.webp)

The Alerts section lets you send alerts to a
Slack channel. See the image below:

![Kubecost Alerts section with the health alerts configuration and the Slack webhook](/images/kubecost-17.webp)

Below are the types of alerts that can be created:

![List of Kubecost alert types: Allocation Budget, Allocation Efficiency, Allocation Recurring Update, Allocation Spend Change, Asset Budget and Cloud Report](/images/kubecost-18.webp)

## Conclusion

The purpose of this post was to give an overview of the Kubecost
management console. For more information, see the
[Kubecost documentation](https://docs.kubecost.com/).
