---
title: "Infrastructure management with GitOps and ArgoCD"
slug: "2024-06-09-infrastructure-management-with-gitops-and-argocd"
description: "A GitOps tutorial: installing ArgoCD on a Kubernetes cluster with kind and deploying an nginx application."
images: ["/images/capa-gerenciamento-de-infraestrutura-com-gitops-e-argocd.png"]
author: Jason
date: 2024-06-09
tags: [ "Containers", "Kubernetes", "GitOps", "IaC" ]
type: post
featured: true
---

The goal of this article is to create a tutorial on how to install ArgoCD on
Kubernetes using kind, and we'll also deploy an application (our
famous and beloved nginx) using ArgoCD.

## Prerequisites

To install ArgoCD and follow this article, you'll need
the following:

-   Helm installed.
-   kubectl installed.
-   A GitHub account.
-   A local cluster using kind, as I showed in [Creating a
    local Kubernetes cluster with
    kind]({{< relref "/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind" >}}).

## Concepts

Before getting to the hands-on part, let's take a few steps back to
understand GitOps.

GitOps is a powerful way to manage and automate your
infrastructure declaratively using a git repository.

![GitOps diagram: the developer commits to Git, ArgoCD pulls the changes and syncs the state in Kubernetes](/images/argo-2.webp)

Basically, the developer and/or DevOps engineer keeps all the
infrastructure definitions in a repository, and ArgoCD checks it
periodically. If anything changes in the repository, a new
deploy of the application to the Kubernetes cluster starts.

With GitOps you can also reconcile the definitions: if
someone makes a manual change, for example changing the number of
replicas, ArgoCD compares it with what's in the git repository and reverts the
change.

## Installing ArgoCD

To proceed with the installation, you need to create a local cluster using
kind. Then create a namespace called argocd
with the command below:

```bash
kubectl create namespace argocd
```

We'll install ArgoCD using Helm. More information
on the [ArgoCD chart page on Artifact
Hub](https://artifacthub.io/packages/helm/argo-cd-oci/argo-cd).

Below is the command used to install ArgoCD:

```bash
helm install argocd oci://ghcr.io/argoproj/argo-helm/argo-cd --version 7.1.3 -n argocd
```

The installation will start and, if it succeeds, you'll see a
screen like the one below:

![ArgoCD Helm installation output with instructions to access the UI and get the initial admin password](/images/argo-3.webp)

To check whether the pods came up correctly we can run the
following command:

```bash
kubectl get pods -n argocd
```

## Accessing ArgoCD

In this step we'll retrieve the ArgoCD admin password, which is in a
secret, by running the command below:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Save the value shown in your terminal.

Now let's access the ArgoCD console with a port-forward
pointing to the ArgoCD service, using the command below:

```bash
kubectl port-forward service/argocd-server -n argocd 8080:443
```

In any web browser, open
[https://127.0.0.1:8080](https://127.0.0.1:8080) (with **https**, because the
port-forward points to ArgoCD's port 443). The browser should
show a certificate error, which can be ignored for this test. You should see
the screen below:

![ArgoCD login screen](/images/argo-4.webp)

Now use the admin user and the password retrieved from the
secret in the previous step to access the ArgoCD console. You should
see a screen like the one below:

![ArgoCD home screen with no applications yet](/images/argo-5.webp)

## Creating a git repository

Now let's create a git repository with the definitions of an nginx,
as below:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
  namespace: argocd
spec:
  replicas: 2
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

If you want, you can use the
[nginx-example](https://github.com/jjasonhenrique/nginx-example) repository, which
I created for this article.

## Connecting a git repository to ArgoCD

Now let's connect the ArgoCD server to the
repository created in the previous step. To do that, go to the
**Settings** option on the left side, then **Repositories**, and click
**Connect Repo**. You'll see the screen below:

![ArgoCD repository connection screen with the SSH method](/images/argo-6.webp)

In this option you can add either a git repository or a
Helm chart repository.

In our case, it should be configured as below:

![Connecting the nginx-example repository in ArgoCD over HTTPS](/images/argo-7.webp)

Then click the **Connect** button and the connection to the
GitHub repository will be shown, as below:

![nginx-example repository connected in ArgoCD with Successful status](/images/argo-8.webp)

## Creating the App with the ArgoCD console

To create our application with ArgoCD, go to the
**Applications** menu and click the **New App** option,
as in the image below:

![ArgoCD new application form with the General and Sync Policy sections](/images/argo-9.webp)

Now let's configure our application. Below is the image
of the first settings:

![nginx application in the default project with automatic sync, prune, self heal and auto create namespace](/images/argo-10.webp)

Under **General** we have the following settings:

-   **Application Name** will be our application's name.
-   **Project** lets us select which ArgoCD project our
    application will be created in. Since we didn't create any project, we'll
    use the **Default** project, which already exists.
-   **Sync Policy** we'll change to **Automatic** and enable the two
    options below:
    -   **Prune Resources**: when enabled, ArgoCD deletes from the
        cluster the resources that were removed from the git repository. Without
        it, a resource deleted from git keeps running in the cluster.
    -   **Self Heal**: when enabled, ArgoCD undoes changes
        made directly in the cluster, such as a `kubectl scale` that changes the
        number of replicas, and goes back to the state defined in git.
-   **Auto-Create Namespace**: if the namespace doesn't exist, it will be
    created in the Kubernetes cluster.

Now let's move on to the second part of the settings, as in the image
below:

![Application source pointing to the nginx-example repository, path deploy, and destination in the nginx namespace of the same cluster](/images/argo-11.webp)

Under **Source** we have the following settings:

-   **Repository URL** is the URL of the git repository that holds
    our app's definition.
-   **Revision** we'll leave at the default value, HEAD, but we can
    configure other branches or tags.
-   **Path** is the directory in the repository where our
    YAML file's definition is.

Under **Destination** we have the following settings:

-   **Cluster URL** is our Kubernetes cluster's URL; since we
    only have one cluster, only one option will show up, but it's
    possible to configure more than one cluster.
-   **Namespace** is the name of the namespace that will be created in our cluster
    if it doesn't exist, and where the application will be created.

Now just click the **Create** button, and the application will be shown
as below:

![nginx application in ArgoCD with Healthy and Synced status](/images/argo-12.webp)

Clicking on our application's tile you get a view of all
the objects created in Kubernetes for this deploy, as below:

![nginx application tree in ArgoCD with the deployment, replicaset and pods, all healthy](/images/argo-13.webp)

## Creating the application from the command line

In the previous step we created our application with the ArgoCD console,
but we can also create it from the command line using kubectl. To
do that, create a file called app.yaml with the content below:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/jjasonhenrique/nginx-example.git'
    path: deploy
    targetRevision: HEAD
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: nginx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Then run the command below:

```bash
kubectl apply -f app.yaml
```

To check your application's state in ArgoCD you can use the
command below:

```bash
kubectl get apps -n argocd 
```

You should see output similar to the one below:

![kubectl output showing the nginx application with Synced and Healthy status](/images/argo-14.png)

## Validation

In this step we'll make a manual change to our application
directly in the cluster, changing the number of replicas to 10 with the
command below:

```bash
kubectl scale deploy/nginx --replicas 10 -n nginx
```

You'll notice that several pods are terminated right away and
only 2 pods remain Running, as configured in the definition
in git. This is ArgoCD's reconcile concept. Below you
can see an image with the pods being terminated:

![nginx pods being terminated after the number of replicas was reduced back to the repository's value](/images/argo-15.webp)

You can also check the Kubernetes events with the command
below:

```bash
kubectl get events -n nginx
```

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

The goal of this article was to explain a bit about GitOps and to show
how to install ArgoCD and create an application from a git
repository.

Keep in mind that the way ArgoCD was installed here is only for testing.
For production environments, it's recommended to run ArgoCD in HA mode.
