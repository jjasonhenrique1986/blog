---
title: "Kubernetes: understanding DaemonSet, Deployment and StatefulSet"
slug: "2024-10-12-kubernetes-understanding-daemonset-deployment-and-statefulset"
description: "Understand the differences between DaemonSet, Deployment and StatefulSet in Kubernetes, with examples on a local kind cluster."
images: ["/images/capa-kubernetes-entenda-daemonset-deployment-e-statefulset.png"]
author: Jason
date: 2024-10-12
tags: [ "Kubernetes", "EKS", "Containers" ]
type: post
---

The goal of this article is to explain and demonstrate the
DaemonSet, Deployment and StatefulSet objects in Kubernetes, the main
container orchestration tool. For the tests we'll use
a local Kubernetes cluster with kind.

To create your local cluster with kind, follow the post [Creating a
local Kubernetes cluster with
kind]({{< relref "/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind" >}}).
That cluster has one node as the ControlPlane and two other nodes as
the DataPlane.

## Prerequisites

To follow this article, you'll need:

-   A Kubernetes cluster; in our case, we'll use kind.
-   kubectl installed.

For the examples we'll create a namespace where our tests
will run. To create this namespace, run the command below:

```bash
kubectl create ns app
```

## DaemonSet

### What is it?

A **DaemonSet** is a type of Kubernetes controller that ensures
a copy of a given Pod runs on each node of a
cluster. It's especially useful for running services that need to be
present on every node, such as monitoring agents, logging agents or
networking systems.

### When to use it?

-   **Monitoring and logging**: if you need to collect logs or
    metrics from every node, a DaemonSet is ideal.
-   **Networking**: used to deploy networking tools, such as a
    CNI (Container Network Interface), for example weave-net, which must
    be present on every node.

### Example

For this first example we'll create a file called daemonset.yaml
with the content below:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: app
spec:
  selector:
    matchLabels:
      name: fluent-bit
  template:
    metadata:
      labels:
        name: fluent-bit
    spec:
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:latest
        resources:
          limits:
            memory: 200Mi
            cpu: 100m
```

To apply the example above, run the command below:

```bash
kubectl apply -f daemonset.yaml
```

In this example, the DaemonSet deploys a fluent-bit agent to
collect logs from each node in the cluster. This example is only meant to show
how a DaemonSet works; we won't test the
fluent-bit agent itself.

To check the pods, run the command below:

```bash
kubectl get pods -n app
```

It should return output similar to the image below:

![Pods of the fluent-bit DaemonSet running](/images/deploy-2.webp)

Since we have only two nodes running as DataPlane, only
two fluent-bit pods were created. To see which nodes the pods are
running on, add the -o wide flag, as in the image below:

![fluent-bit pods spread out, one on each worker node](/images/deploy-3.webp)

The command below shows how many daemonsets are running in the cluster:

```bash
kubectl get daemonset -n app
```

![Output of kubectl get ds showing the fluent-bit DaemonSet with 2 ready pods](/images/deploy-4.webp)

## Deployment

### What is it?

A **Deployment** is the most widely used controller in Kubernetes to
manage the creation and scaling of Pod replicas.

### When to use it?

-   **Stateless applications**: ideal for most web applications,
    APIs or microservices that don't keep internal state between different
    replicas.

### Example

In this second example we'll create a file called deployment.yaml with
the content below:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx  
  namespace: app  
  labels:
    app: nginx
spec:
  replicas: 10
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21.1
        ports:
        - containerPort: 80
```

To apply the manifest above, run the command:

```bash
kubectl apply -f deployment.yaml
```

Here, the Deployment defines ten replicas of an NGINX container. If a
Pod fails, the Deployment automatically creates a new replica to
ensure availability.

To check the pods, run the command below:

```bash
kubectl get pods -n app
```

![Ten pods of the nginx deployment running in the app namespace](/images/deploy-5.webp)

With Kubernetes' default configuration we can see that the pods are
spread across the nodes, as in the image below:

![nginx deployment pods spread across the cluster-jason-worker and cluster-jason-worker2 nodes](/images/deploy-6.webp)

The command below shows information about the deployment:

```bash
kubectl get deployment -n app
```

![Output of kubectl get deployment with nginx at 10 of 10 ready replicas](/images/deploy-7.webp)

## StatefulSet

### What is it?

A **StatefulSet** is similar to a Deployment, but it's used to
manage stateful applications, where data persistence is
important. The StatefulSet ensures Pods are deployed and
scaled in a predictable way.

### When to use it?

-   **Databases**: ideal for distributed databases, such as
    MySQL, PostgreSQL or any application that requires data
    persistence across pods. It's also used for applications that need
    a logical order.

### Example

In this third example we'll create a file called statefulset.yaml
with the content below:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: app
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

In this example, a StatefulSet is used to create three replicas of an
NGINX container, each one with its own persistent storage
volume.

To check the pods, run the command below:

```bash
kubectl get pods -n app
```

![StatefulSet pods web-0, web-1 and web-2 created in order](/images/deploy-8.webp)

Another thing we can see is the logical order in which the
pods come up: first web-0, then web-1 and then web-2.
This can be a requirement for some applications.

The command below shows information about the statefulset:

```bash
kubectl get statefulset -n app
```

## Bonus: ReplicaSet

A **ReplicaSet** is an object responsible for keeping a specific number
of replicas of a deployment. It's created
automatically when we create a deployment. Below is the command to
check the replicaset:

```bash
kubectl get replicaset -n app
```

![Output of kubectl get rs with the nginx replicaset and 10 replicas](/images/deploy-9.webp)

Another point about ReplicaSets is that each new deploy creates a new
ReplicaSet, and the replicas will be available in the new
replicaset. Below we'll simulate a new deployment by running the
command below:

```bash
kubectl set image deploy/nginx nginx=nginx:1.19 -n app
```

In the image below we can see that a new replicaset was created and the
deployment's pods were moved to this new replicaset.

![Two nginx replicasets after the update: the new one with 10 replicas and the old one with 0](/images/deploy-10.webp)

## Conclusion

The main goal of this article was to show what the Deployment,
DaemonSet, StatefulSet and ReplicaSet objects in Kubernetes are and how to use them.

I hope this article was useful to you, whether you've been working with
Kubernetes for a short or a long time.
