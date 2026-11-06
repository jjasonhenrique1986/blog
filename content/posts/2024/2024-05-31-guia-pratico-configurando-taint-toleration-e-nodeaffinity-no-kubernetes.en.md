---
title: "Hands-on guide: configuring Taint, Toleration and NodeAffinity in Kubernetes"
slug: "2024-05-31-hands-on-guide-taint-toleration-and-nodeaffinity-in-kubernetes"
description: "A hands-on guide to Taint, Toleration and NodeAffinity to control which nodes pods run on in Kubernetes, using kind."
images: ["/images/capa-guia-pratico-configurando-taint-toleration-e-nodeaffinity-no.png"]
author: Jason
date: 2024-05-31
tags: [ "Containers", "Kubernetes", "EKS" ]
type: post
---

The goal of this post is to show how to make your application's pods
run on a specific node, using kind. This configuration also
applies to Kubernetes on a cloud provider, such as AWS EKS.

## Prerequisites

To follow the demos in this post, you'll need the following
prerequisites:

-   kubectl installed.
-   A local cluster using kind. For that, follow the post [Creating
    a local Kubernetes cluster with
    kind]({{< relref "/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind" >}}).

## Concepts

Before getting to the hands-on part, we need to level-set a few
concepts used in this configuration. Here they are:

### Taint

The main purpose of a taint in Kubernetes is to
block or repel certain pods from running on, or being scheduled
to, a given node. Taints have two main types of
effect:

-   **NoSchedule** -- once the taint is configured, no new pod
    will be placed on that node, but the pods already on it will keep
    running.

-   **NoExecute** -- once the taint is configured, no pod will
    run on that node: even the pods already on it will be
    terminated and placed on another node without that taint.

Besides the two options above there's another effect called
**PreferNoSchedule**, which is a softer, preference-based
effect. With it, the Kubernetes scheduler tries
not to place pods on the node with that taint, but it isn't guaranteed.

Below is the syntax used to configure a taint on a node:

```bash
kubectl taint node name-node key=value:effect

kubectl taint node node01 enable=true:NoExecute
```

To remove a taint, use the command below:

```bash
kubectl taint node name-node key=value:effect -

kubectl taint node node01 enable=true:NoExecute -
```

To check whether a node has a taint configured, use the command
below:

```bash
kubectl describe node name-node
```

### Toleration

The main purpose of a toleration in Kubernetes is to
allow certain pods of a deployment to be scheduled and run
on a node that has a taint configured. Below is an example of the
configuration to add to your
deployment's manifest to set up a toleration:

```yaml
tolerations:
- key: "enable"
  operator: "Equal"
  value: "true"
  effect: "NoExecute"
```

### NodeAffinity

NodeAffinity in Kubernetes ensures
that a given pod goes to a given node. Below is an example of the
configuration to add to your
deployment's manifest to set up a nodeAffinity:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
```

To configure a nodeAffinity you need to create a
label for the node or use a label that already exists.

## Creating the deployment

Now for the hands-on part. You need to create
a local cluster with kind. To check that the cluster is
up, run the command below:

```bash
kubectl get nodes
```

The first step is to create a namespace for our application with the
command below:

```bash
kubectl create namespace app
```

Now let's create an nginx deployment in this namespace with 15 replicas and
see how it spreads across the cluster's nodes. For that, we'll use
the manifest below:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: app
spec:
  replicas: 15
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

To generate this manifest, use the command below:

```bash
kubectl create deploy nginx --image=nginx --replicas=15 -n app --dry-run=client -o yaml > nginx.yaml
```

Now let's apply this manifest with the command below:

```bash
kubectl apply -f nginx.yaml
```

Now let's check where the pods are running with the command
below:

```bash
kubectl get pods -n app -o wide
```

You should see output like the image below:

![nginx deployment pods spread across the cluster-jason-worker and cluster-jason-worker2 nodes](/images/taint-2.png)

We can see the pods were spread across the
**cluster-jason-worker** and **cluster-jason-worker2** nodes.

## Configuring the taint and toleration

Now let's move on to the second step, which is configuring the
taint and then adding the toleration to our deployment.
Below is the command used to configure the taint on the
**cluster-jason-worker** node:

```bash
kubectl taint node cluster-jason-worker enable=true:NoExecute
```

To check whether the taint was applied successfully, run the command
below:

```bash
kubectl describe node cluster-jason-worker
```

And look for the **Taints** field, as in the image below:

![kubectl describe node output showing the enable=true:NoExecute taint on cluster-jason-worker](/images/taint-3.webp)

Now let's change our deployment by adding the
tolerations block, as in the file below:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: app
spec:
  replicas: 15
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
      tolerations:
      - key: "enable"
        operator: "Equal"
        value: "true"
        effect: "NoExecute"
```

Now let's run the command below to replace the pods with
the new configuration:

```bash
kubectl replace -f nginx.yaml
```

To confirm the configuration was really applied to the cluster, run the
command below to view the manifest:

```bash
kubectl get deploy nginx -o yaml -n app
```

Now let's look at the pod distribution again, as
below:

![nginx pods in the app namespace running on both workers after applying the taint](/images/taint-4.webp)

We can see there are still pods on both nodes (cluster-jason-worker and
cluster-jason-worker2). That's because the toleration allows the
pod to run on the tainted node, but doesn't require it. To require it, we use
nodeAffinity, which we'll see in the next section.

## Configuring a nodeAffinity

As seen in the previous section, even with taint and toleration there are still
pods on both nodes. To configure the nodeAffinity we'll use
the existing **kubernetes.io/hostname** label. To see the
labels already on your nodes, run the command below:

```bash
kubectl get nodes --show-labels
```

Now let's make another change to our deployment, adding the
affinity block, as in the file below:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: app
spec:
  replicas: 15
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - cluster-jason-worker
      containers:
      - image: nginx
        name: nginx
      tolerations:
      - key: "enable"
        operator: "Equal"
        value: "true"
        effect: "NoExecute"
```

Let's replace the pods again and check that they're
only on the **cluster-jason-worker** node, as in the image below:

![nginx pods in the app namespace running only on the cluster-jason-worker node](/images/taint-5.webp)

Another test we can do to validate the configuration is
to scale the number of pods to 20 and check that the pods
stay on the same node, by running the command below:

```bash
kubectl scale deploy/nginx -n app --replicas 20
```

We can see 5 more pods were created on the same node, as in the
image below:

![More nginx replicas in the app namespace, all on the cluster-jason-worker node](/images/taint-6.webp)

## Deleting the local cluster

To delete the cluster you created with kind, follow the commands below:

```bash
kind get clusters
```

To get the cluster name.

```bash
kind delete clusters name-cluster
```

The command above deletes the cluster.

## Conclusion

The goal of this post was to show how to make your application's pods
go to a specific node, using kind. The same configuration works
on EKS, applying the taints and labels to the nodes of a node group.

## References

Below is the documentation I used as the basis for this post:

-   [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
-   [Affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)
