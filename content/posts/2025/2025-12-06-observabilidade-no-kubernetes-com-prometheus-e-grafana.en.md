---
title: "Observability in Kubernetes with Prometheus and Grafana"
slug: "2025-12-06-observability-in-kubernetes-with-prometheus-and-grafana"
description: "How to install kube-prometheus-stack with Helm to monitor a Kubernetes cluster with Prometheus, Grafana and Alertmanager, with PromQL and alert examples."
images: ["/images/capa-observabilidade-no-kubernetes-com-prometheus-e-grafana.png"]
author: Jason
date: 2025-12-06
tags: [ "Kubernetes", "Observability", "Containers" ]
type: post
---

The goal of this article is to set up a monitoring stack for a
Kubernetes cluster using **kube-prometheus-stack**, a Helm chart from the
Prometheus community that installs Prometheus, Grafana,
Alertmanager and the metrics exporters in one go, with ready-made dashboards and alerts.

## Observability in a few words

Observability is the ability to understand what is happening inside
a system from the data it produces. This data is usually
divided into three pillars:

-   **Metrics**: numeric values over time, such as CPU usage,
    requests per second and error rate.
-   **Logs**: records of events produced by the applications.
-   **Traces**: the path of a request through several services.

In this article we'll focus on **metrics**, which are usually the first
step and the foundation for alerts.

## Prerequisites

-   A Kubernetes cluster. For testing, a local cluster with kind
    works well; see [how to create a local cluster with
    kind]({{< relref "/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind" >}}).
-   kubectl and Helm 3 installed.

## What kube-prometheus-stack installs

-   **Prometheus Operator**: manages Prometheus and Alertmanager through
    custom resources (CRDs) such as `ServiceMonitor` and
    `PrometheusRule`.
-   **Prometheus**: collects and stores the metrics.
-   **Alertmanager**: receives alerts from Prometheus and sends them to email,
    Slack, etc.
-   **Grafana**: visualization, with dozens of ready-made dashboards.
-   **node-exporter**: a DaemonSet that exposes node metrics (CPU,
    memory, disk).
-   **kube-state-metrics**: exposes the state of Kubernetes objects
    (pods, deployments, restarts, etc.).

## Installation

Add the repository and install the chart in the `monitoring` namespace:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

Watch the pods until they're all ready:

```bash
kubectl get pods -n monitoring -w
```

The resource names use the release name, `kube-prometheus-stack`.
To see the Services created:

```bash
kubectl get svc -n monitoring
```

## Accessing Grafana

The user is `admin` and the password is in a Secret created by the chart:

```bash
kubectl get secret -n monitoring kube-prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

Run the port-forward and open `http://localhost:3000`:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

Under **Dashboards** there are already several ready-made panels. Some worth
looking at first:

-   **Kubernetes / Compute Resources / Cluster**: an overview of the cluster's CPU and
    memory.
-   **Kubernetes / Compute Resources / Namespace (Pods)**: usage per pod
    in a namespace.
-   **Node Exporter / Nodes**: details of each node.

## Accessing Prometheus

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

Open `http://localhost:9090`. Under **Status → Targets** you can see everything
Prometheus is collecting.

On a kind cluster it's normal for some control plane targets to show
as **down**, such as etcd, the scheduler, the controller-manager and
kube-proxy. On these clusters those components expose metrics only on
`127.0.0.1` by default. This doesn't affect the rest of the monitoring.

## Some PromQL queries

In Prometheus's **Graph** tab (or Grafana's **Explore**) try
the queries below.

CPU usage per pod in a namespace, in cores:

```promql
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="monitoring", container!=""}[5m]))
```

Memory in use per pod:

```promql
sum by (pod) (container_memory_working_set_bytes{namespace="monitoring", container!=""})
```

Containers that restarted in the last hour (a kube-state-metrics metric):

```promql
increase(kube_pod_container_status_restarts_total[1h]) > 0
```

Targets that are down:

```promql
up == 0
```

## Monitoring your application with ServiceMonitor

For Prometheus to collect metrics from your own application, you don't need to
edit its configuration: just create a `ServiceMonitor`. Assuming
an application with a Service labeled `app: my-app` and a port
named `http-metrics` that exposes `/metrics`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: app
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: http-metrics
    path: /metrics
    interval: 30s
```

The `release: kube-prometheus-stack` label is **required** in this
installation: by default Prometheus only picks up ServiceMonitors that
have the Helm release label. This is one of the most common mistakes: the
ServiceMonitor is created and the metrics never show up.

## Creating an alert

Alerts are defined with the `PrometheusRule` resource, which follows the
same label rule. The example below fires when a container
restarts more than 3 times in 15 minutes:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
  - name: app.rules
    rules:
    - alert: ContainerRestartingTooOften
      expr: increase(kube_pod_container_status_restarts_total[15m]) > 3
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Container {{ $labels.container }} restarting in pod {{ $labels.pod }}"
        description: "The container restarted more than 3 times in the last 15 minutes."
```

```bash
kubectl apply -f app-alerts.yaml
```

The rule shows up under **Alerts** in Prometheus. To receive the
notifications, configure a receiver (Slack, email, etc.) in Alertmanager
through the chart's values.

If this alert fires, a common cause is a badly configured liveness probe.
Read more in the article about
[probes in Kubernetes]({{< relref "/posts/2025/2025-02-15-kubernetes-configurando-liveness-readiness-e-startup-probes" >}}).

## Cleanup

```bash
helm uninstall kube-prometheus-stack -n monitoring
kubectl delete namespace monitoring
```

Helm **doesn't remove the chart's CRDs** (`servicemonitors`,
`prometheusrules`, etc.). If you want to remove them too:

```bash
kubectl get crd -o name | grep monitoring.coreos.com | xargs kubectl delete
```

## Conclusion

With a single Helm chart we get a complete metrics stack, with
ready-made dashboards and alerts for the cluster. From here, the next
steps are creating ServiceMonitors for your applications, configuring
Alertmanager to send notifications and moving on to the other pillars,
such as logs (Loki, for example) and traces (OpenTelemetry).

## References

-   [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
-   [Prometheus Operator](https://prometheus-operator.dev/)
-   [PromQL queries](https://prometheus.io/docs/prometheus/latest/querying/basics/)
