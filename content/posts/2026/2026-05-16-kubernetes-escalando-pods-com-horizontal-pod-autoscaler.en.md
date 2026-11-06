---
title: "Kubernetes: scaling pods automatically with the Horizontal Pod Autoscaler"
slug: "2026-05-16-kubernetes-scaling-pods-with-the-horizontal-pod-autoscaler"
description: "How to set up the Horizontal Pod Autoscaler (HPA) in Kubernetes with metrics-server, generate load and watch the pods scale, using a local cluster with kind."
images: ["/images/capa-kubernetes-escalando-pods-com-horizontal-pod-autoscaler.png"]
author: Jason
date: 2026-05-16
tags: [ "Kubernetes", "Containers" ]
type: post
---

The goal of this article is to show how to use the **Horizontal Pod
Autoscaler (HPA)** to automatically increase and decrease the number
of pods of an application according to CPU usage. We'll generate load
on the application and watch the HPA create new pods and then remove
them when the load is gone.

For the tests we'll use a local cluster with kind. If you need it, see
[how to create a local cluster with
kind]({{< relref "/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind" >}}).

## Prerequisites

-   A Kubernetes cluster; in our case we'll use kind.
-   kubectl installed.

## How the HPA works

The HPA is a controller that, every 15 seconds by default, queries the
pods' metrics and calculates how many replicas are needed to reach
the configured target. The formula is:

```
desired replicas = ceil(current replicas × (current metric value / desired value))
```

For example: with 2 replicas using 100% CPU on average and a target of
50%, the HPA calculates `ceil(2 × 100 / 50) = 4` replicas.

Two important points:

-   CPU usage is calculated **relative to the container's `requests`**.
    Without CPU `requests` defined, the HPA can't calculate the
    percentage.
-   The metrics come from **metrics-server**, which isn't installed in kind.

## Installing metrics-server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

In kind, the kubelet certificates aren't signed by a CA that
metrics-server recognizes. For test environments, we add the
`--kubelet-insecure-tls` flag:

```bash
kubectl patch deployment metrics-server -n kube-system --type=json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

**Don't use this flag in production.** In managed clusters, such as EKS,
metrics-server can be installed without it.

Wait for the pod to be ready and test it:

```bash
kubectl top nodes
```

If the command shows the nodes' CPU and memory usage, metrics-server
is working.

## Deploying the application

We'll use the `hpa-example` image, from the Kubernetes documentation, which
runs a CPU-heavy calculation on each request. Create a file
called app.yaml:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
          limits:
            cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  selector:
    run: php-apache
  ports:
  - port: 80
```

```bash
kubectl apply -f app.yaml
```

## Creating the HPA

The quickest way is the `kubectl autoscale` command:

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

But ideally you keep the HPA versioned in YAML. The equivalent, using the
`autoscaling/v2` API, is:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Check the HPA:

```bash
kubectl get hpa php-apache
```

The **TARGETS** column shows the current usage relative to the target, for
example `cpu: 0%/50%`. Right after creation it may show
`<unknown>` for a few seconds, until the first metrics collection.

## Generating load

In another terminal, start a pod that sends requests non-stop to the
application:

```bash
kubectl run -i --tty load-generator --rm --image=busybox:1.36 --restart=Never \
  -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

In the first terminal, watch the HPA:

```bash
kubectl get hpa php-apache --watch
```

In about a minute, CPU usage goes well above 50% and the HPA
starts increasing the number of replicas. You can see the new pods
with:

```bash
kubectl get pods -l run=php-apache
```

## Stopping the load

Stop the load generator with `Ctrl + C`. CPU usage drops quickly,
but the replicas **don't decrease right away**. By default the HPA waits for a
**5-minute** stabilization window before scaling down, to keep
the number of pods from going up and down with every load
fluctuation.

This behavior can be tuned in the HPA's `behavior` field. For
example, to remove at most 1 pod per minute, after 5 minutes of
stability:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60
```

## Best practices

-   **Set realistic `requests`.** Since the percentage is calculated over
    `requests`, a value that's too low makes the HPA scale too early, and
    a value that's too high makes it slow to react.
-   **Don't set `replicas` in the Deployment when using HPA with GitOps.**
    If ArgoCD or `kubectl apply` reapply the number of replicas from the
    YAML, they'll fight with the HPA. Read more about GitOps in the
    [ArgoCD]({{< relref "/posts/2024/2024-06-09-gerenciamento-de-infraestrutura-com-gitops-e-argocd" >}}) article.
-   **Configure readiness probes.** New pods should only receive
    traffic when they're ready. See the article about
    [probes]({{< relref "/posts/2025/2025-02-15-kubernetes-configurando-liveness-readiness-e-startup-probes" >}}).
-   **Remember the nodes.** The HPA creates pods, but if there's no room in the
    cluster they stay `Pending`. In production, combine the HPA with a
    node autoscaler, such as Cluster Autoscaler or Karpenter on EKS.

## Cleanup

```bash
kubectl delete hpa php-apache
kubectl delete -f app.yaml
```

## Conclusion

The HPA is the simplest way to make an application follow
demand in Kubernetes: more pods when the load goes up and fewer pods when
it goes down, saving resources. With metrics-server installed and
well-defined `requests`, the basic setup takes just a few
lines of YAML.

## References

-   [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
-   [HorizontalPodAutoscaler Walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
-   [metrics-server](https://github.com/kubernetes-sigs/metrics-server)
