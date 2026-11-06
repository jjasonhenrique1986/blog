---
title: "Kubernetes: configuring Liveness, Readiness and Startup Probes"
slug: "2025-02-15-kubernetes-configuring-liveness-readiness-and-startup-probes"
description: "Understand the difference between Liveness, Readiness and Startup Probes in Kubernetes and see hands-on examples of each one on a local kind cluster."
images: ["/images/capa-kubernetes-configurando-liveness-readiness-e-startup-probes.png"]
author: Jason
date: 2025-02-15
tags: [ "Kubernetes", "Containers" ]
type: post
---

The goal of this article is to explain and demonstrate Kubernetes
**probes**: Liveness, Readiness and Startup. They are how
Kubernetes knows whether your container is healthy and whether it's
ready to receive traffic. Configuring probes well avoids two
very common problems: a stuck application that is never restarted and an
application receiving requests before it's ready.

For the tests we'll use a local cluster with kind. If you don't have
one yet, see [how to create a local cluster with
kind]({{< relref "/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind" >}}).

## Prerequisites

-   A Kubernetes cluster; in our case we'll use kind.
-   kubectl installed.

Let's create a namespace for the tests:

```bash
kubectl create ns probes
```

## Concepts

The probes are run by the **kubelet** on each node. Each type of probe
has a different consequence when it fails:

| Probe | Question it answers | What happens when it fails |
|---|---|---|
| **Liveness** | Is the container alive? | The kubelet **restarts** the container |
| **Readiness** | Is the container ready to receive traffic? | The pod is **removed from the Service endpoints** (it isn't restarted) |
| **Startup** | Has the application finished starting? | Until it passes, liveness and readiness are **disabled**; if it hits the limit, the container is restarted |

Probes can check the container in four ways:

-   **httpGet**: makes an HTTP request; a status between 200 and 399 is success.
-   **tcpSocket**: tries to open a TCP connection on the port.
-   **exec**: runs a command inside the container; exit code 0 is success.
-   **grpc**: uses the gRPC health checking protocol.

And the main parameters (with their default values):

-   `initialDelaySeconds` (0): how long to wait before the first check.
-   `periodSeconds` (10): interval between checks.
-   `timeoutSeconds` (1): maximum response time.
-   `failureThreshold` (3): consecutive failures to consider the probe failed.
-   `successThreshold` (1): consecutive successes to consider it
    successful again. For liveness and startup this value must be 1.

## Liveness Probe

To see liveness in action we'll use a container that creates a file,
waits 30 seconds and deletes that file. The probe checks whether the file
exists, so after 30 seconds it starts failing.

Create a file called liveness.yaml with the content below:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec
  namespace: probes
spec:
  containers:
  - name: app
    image: busybox:1.36
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

Apply the manifest:

```bash
kubectl apply -f liveness.yaml
```

After about a minute, check the pod's events:

```bash
kubectl describe pod liveness-exec -n probes
```

The events will show the message `Liveness probe failed` and,
right after, that the container will be restarted. Watching the pod, the
**RESTARTS** column goes up with each cycle:

```bash
kubectl get pod liveness-exec -n probes -w
```

## Readiness Probe

Now let's look at readiness. We'll start an nginx with a readiness
probe that queries the `/ready` path, which doesn't exist yet. nginx
answers 404, the probe fails and the pod never becomes ready.

Create a file called readiness.yaml:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: probes
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: probes
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f readiness.yaml
kubectl get pods -n probes -l app=web
```

The pod's status is **Running**, but the READY column shows **0/1**.
Notice that it's **not restarted**, it just doesn't receive traffic. We can
confirm that the Service has no ready endpoints:

```bash
kubectl get endpointslices -n probes -l kubernetes.io/service-name=web
```

Now let's create the file the probe is looking for:

```bash
kubectl exec -n probes deploy/web -- sh -c 'echo ok > /usr/share/nginx/html/ready'
```

In a few seconds the pod goes to **1/1** and its IP shows up in the
Service's EndpointSlice. That's exactly the behavior we want
during a deploy: the new pod only receives requests when it's
ready.

## Startup Probe

Some applications take a long time to start, for example Java applications
that load a lot of dependencies. If liveness starts checking before
the application is up, the container gets restarted in a loop.

The startup probe solves this: until it succeeds, the other
probes don't run. In the example below the application has up to
`30 x 10 = 300` seconds to start. After that, liveness takes over
with a shorter interval:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-example
  namespace: probes
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
    startupProbe:
      httpGet:
        path: /
        port: 80
      failureThreshold: 30
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /
        port: 80
      periodSeconds: 10
```

## Best practices

-   **Liveness shouldn't depend on external services.** If the
    database goes down and liveness checks the database, all pods will be
    restarted at the same time, and that doesn't fix the problem.
-   **Readiness can check dependencies**, since the consequence is
    only taking the pod out of load balancing.
-   **Use a startup probe for slow applications** instead of greatly increasing
    liveness's `initialDelaySeconds`.
-   **Create lightweight endpoints**, such as `/healthz` and `/ready`, that answer
    quickly and don't do heavy processing.

## Cleanup

```bash
kubectl delete ns probes
```

## Conclusion

Probes are a simple setting that makes a big difference to an
application's availability. Liveness ensures a stuck
container gets restarted, readiness ensures only ready pods receive
traffic and startup protects applications that take a while to come up.

If you want to better understand the controllers that use these probes, see
also the article about
[DaemonSet, Deployment and StatefulSet]({{< relref "/posts/2024/2024-10-12-kubernetes-entenda-daemonset-deployment-e-statefulset" >}}).

## References

-   [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
