---
title: "Installing and testing the Falco tool"
slug: "2023-11-16-installing-and-testing-the-falco-tool"
description: "Installing and testing Falco, a Sysdig tool that detects threats in near real time in Kubernetes, containers and the cloud."
images: ["/images/capa-instalando-e-testando-a-ferramenta-falco.png"]
author: Jason
date: 2023-11-16
tags: [ "Kubernetes", "Security", "Containers" ]
type: post
---

Falco is a tool developed by Sysdig whose job is
to detect threats in near real time in the cloud, in workloads running
on Kubernetes and in containers.

Falco can monitor events from several sources, such as the Linux
kernel, the Kubernetes API and containers. It also integrates with
messaging tools, such as Slack and Teams, and with Prometheus. For
that, you need a Falco daemon called falcosidekick.

For more information about Falco, see the links below:

-   <https://github.com/falcosecurity/falcosidekick>
-   <https://falco.org/docs/>

Falco is also one of the topics covered in the CKS
(Certified Kubernetes Security Specialist) certification exam.

## Prerequisites

For this Falco demo we'll need the following
prerequisites:

-   A local Kubernetes cluster, which can be created with kind (see the
    [kind quick
    start](https://kind.sigs.k8s.io/docs/user/quick-start/) or my
    step by step [Creating a local Kubernetes cluster with
    kind]({{< relref "/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind" >}})).
-   [Helm installed](https://helm.sh/docs/intro/install/).
-   [kubectl installed](https://kubernetes.io/docs/tasks/tools/).

## Installation

Before installing Falco you need to add the Falco
repository to Helm by running the commands below:

``` bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
```

You should also create a separate namespace for the Falco installation
with the command below:

``` bash
kubectl create namespace falco
```

Now let's install Falco by running the command below:

``` bash
helm install falco -n falco --set driver.kind=ebpf --set tty=true falcosecurity/falco
```

To check whether Falco came up correctly, run the command below:

``` bash
kubectl get pods -n falco
```

It should show output like the image below:

![kubectl get pods output with the three Falco pods running](/images/falco-2.webp)

A Falco DaemonSet will be created, which runs a pod on each node of the
cluster. In my case, there are 3 Falco pods. To check,
run the command below:

``` bash
kubectl get daemonset -n falco
```

Now let's check the Falco pods' logs by running
the command below:

``` bash
kubectl logs -f daemonset/falco -n falco
```

You should see output similar to the one below:

![Falco pod logs showing version 0.36.2 and the rules being loaded](/images/falco-3.webp)

## Demo

Now let's simulate Falco in action by creating a pod, connecting
to that pod with kubectl and checking Falco's logs.

Below is the command to create a pod for this test:

``` bash
kubectl run alpine --image alpine -- sh -c "sleep infinity"
```

Now let's connect to the pod we created with the command below:

``` bash
kubectl exec -it alpine -- sh
```

Checking Falco's logs, there will be output similar to the one below:

``` text
15:27:42.692441989: Notice A shell was spawned in a container with an attached terminal (evt_type=execve user=root user_uid=0 user_loginuid=-1 process=sh proc_exepath=/bin/busybox parent=runc command=sh terminal=34816 exe_flags=EXE_WRITABLE container_id=cf009dcc3814 container_image=<NA> container_image_tag=<NA> container_name=<NA> k8s_ns=<NA> k8s_pod_name=<NA>)
```

From the event above we can see that someone connected to the
terminal of the container with ID cf009dcc3814.

## Conclusion

The goal of this article was to install and demonstrate the
Falco tool using a local Kubernetes cluster.

Falco can also be installed on a virtual machine running
Linux, as I showed in the following article:
[Testing Falco on a virtual machine]({{< relref "/posts/2023/2023-11-22-testando-o-falco-em-uma-maquina-virtual" >}}).
