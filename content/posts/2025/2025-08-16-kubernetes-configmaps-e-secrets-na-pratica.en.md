---
title: "Kubernetes: ConfigMaps and Secrets in practice"
slug: "2025-08-16-kubernetes-configmaps-and-secrets-in-practice"
description: "How to separate configuration from code in Kubernetes using ConfigMaps and Secrets: creating them, using them as environment variables and volumes, and security precautions."
images: ["/images/capa-kubernetes-configmaps-e-secrets-na-pratica.png"]
author: Jason
date: 2025-08-16
tags: [ "Kubernetes", "Containers", "Security" ]
type: post
---

The goal of this article is to show how to use **ConfigMaps** and
**Secrets** in Kubernetes. They let you separate configuration from the
container image: the same image can run in development and in
production by changing only the configuration.

For the tests we'll use a local cluster with kind. If you need it, see
[how to create a local cluster with
kind]({{< relref "/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind" >}}).

## Prerequisites

-   A Kubernetes cluster; in our case we'll use kind.
-   kubectl installed.

```bash
kubectl create ns config
```

## Concepts

-   **ConfigMap**: stores **non-sensitive** configuration data, such as
    URLs, log level and configuration files.
-   **Secret**: stores **sensitive** data, such as passwords, tokens and
    certificates.

An important point: by default Secrets are only encoded in
**base64**, and base64 **is not encryption**. Anyone with
read permission on the Secret can see the value. We'll come back to this
point at the end.

Both can be consumed by the pod in two ways: as **environment
variables** or as **files mounted in a volume**.

## Creating a ConfigMap

We can create it from literal values and from files. First create
a file called app.properties:

```bash
cat > app.properties <<EOF
cache.ttl=300
feature.new-checkout=true
EOF
```

Now create the ConfigMap with two literal values and the file:

```bash
kubectl create configmap app-config -n config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info \
  --from-file=app.properties
```

To see the content:

```bash
kubectl get configmap app-config -n config -o yaml
```

The same ConfigMap can be written in YAML, which is ideal to keep
versioned in Git:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: config
data:
  APP_ENV: production
  LOG_LEVEL: info
  app.properties: |
    cache.ttl=300
    feature.new-checkout=true
```

## Creating a Secret

```bash
kubectl create secret generic db-credentials -n config \
  --from-literal=DB_USER=app \
  --from-literal=DB_PASSWORD='S3nh@Forte'
```

See that the value is only base64:

```bash
kubectl get secret db-credentials -n config -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

The command above shows the password in plain text. That's why you should **not version
Secrets as YAML in Git** without some kind of encryption.

## Using them in the pod

Create a file called pod.yaml. In it we use the ConfigMap and the Secret in
three ways: all keys as variables (`envFrom`), a specific key
from the Secret as a variable (`secretKeyRef`) and the `app.properties` file
mounted in a volume:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: config
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: DB_USER
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: DB_PASSWORD
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
      - key: app.properties
        path: app.properties
```

```bash
kubectl apply -f pod.yaml
```

Checking the environment variables and the file:

```bash
kubectl exec -n config app -- env | grep -E 'APP_ENV|LOG_LEVEL|DB_USER'
kubectl exec -n config app -- cat /etc/app/app.properties
```

## Updating the configuration

There's an important difference here:

-   **Environment variables are not updated** in a pod that's already
    running. You need to recreate the pod, for example with
    `kubectl rollout restart deployment/<name>` when using a
    Deployment.
-   **Files mounted through a volume are updated** automatically
    by the kubelet after a few seconds. This doesn't apply when the volume
    is mounted with `subPath`.

Let's test it by changing the file in the ConfigMap:

```bash
kubectl patch configmap app-config -n config --type merge \
  -p '{"data":{"app.properties":"cache.ttl=600\nfeature.new-checkout=true\n"}}'
```

Wait a bit and read the file again. The value of `cache.ttl` will
change to 600, but the `LOG_LEVEL` variable, for example, would only change by
recreating the pod. The application also needs to reread the file to use the
new value.

## Immutable ConfigMaps and Secrets

For configuration that shouldn't change, you can mark the object as
immutable. This prevents accidental changes and reduces the load on the API
server, since the kubelet stops watching for changes:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2
  namespace: config
data:
  APP_ENV: production
immutable: true
```

To change an immutable object you have to delete it and create it again. A
common practice is to put the version in the name (`app-config-v2`, `v3`...) and
update the Deployment to point to the new name.

## Secret precautions

-   **Restrict access with RBAC.** Whoever can read Secrets in a
    namespace can see all of its passwords. See how to configure it
    in the article about
    [RBAC in Kubernetes]({{< relref "/posts/2024/2024-11-15-gerenciando-acesso-no-kubernetes-com-rbac" >}}).
-   **Enable encryption at rest in etcd.** In managed clusters
    such as EKS, this can be done with AWS KMS keys.
-   **Consider an external secrets manager**, such as AWS Secrets
    Manager, integrated with the cluster through the External Secrets Operator or the
    Secrets Store CSI Driver. That way the source of truth stays outside the
    cluster.
-   **Prefer volumes over environment variables** for sensitive data,
    since variables can end up showing in logs and error dumps.

## Cleanup

```bash
kubectl delete ns config
rm app.properties
```

## Conclusion

ConfigMaps and Secrets are the foundation for keeping configuration out of the
image. Use ConfigMaps for regular data and Secrets for sensitive data,
remembering that a Secret on its own isn't encrypted: combine it with RBAC,
encryption in etcd and, if possible, an external secrets manager.

## References

-   [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
-   [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
