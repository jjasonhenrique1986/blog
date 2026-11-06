---
title: "Getting to know the Trivy tool"
slug: "2023-11-05-getting-to-know-the-trivy-tool"
description: "Get to know Trivy, the Aqua Security security scanner and a topic covered in the CKS (Certified Kubernetes Security Specialist) exam."
images: ["/images/capa-conhecendo-a-ferramenta-trivy.png"]
author: Jason
date: 2023-11-05
tags: [ "Kubernetes", "Security", "Containers", "CI/CD" ]
type: post
---

Trivy is a tool developed by Aqua Security and
it's also one of the topics covered in the CKS (Certified Kubernetes
Security Specialist) exam. Below are some links with
more information about Trivy:

-   <https://github.com/aquasecurity/trivy>
-   <https://aquasecurity.github.io/trivy/>

With Trivy you can run the following checks:

-   Vulnerability scanning
-   IaC misconfigurations
-   Detecting sensitive data

You can scan the following targets:

-   Container images
-   Repositories in a version control system, such as
    GitHub
-   Filesystems
-   Kubernetes clusters
-   AWS accounts

The goal of this article is to show vulnerability scanning on a
container image and on a GitHub repository with Trivy.

## Installation

To use Trivy you can install it directly on your machine or run
Trivy as a Docker container.

### Installing on Linux

In my case I'll show the installation on a Linux machine. The latest
version is on the [Trivy releases on
GitHub](https://github.com/aquasecurity/trivy/releases). Below are the
commands you need to run to download and install it:

``` bash
wget https://github.com/aquasecurity/trivy/releases/download/v0.46.0/trivy_0.46.0_Linux-64bit.deb
sudo dpkg -i trivy_0.46.0_Linux-64bit.deb
```

To validate the installation, run the command below:

``` bash
trivy version
```

### Docker

To use Trivy as a container, just use the
**aquasec/trivy:latest** image. For example, to check Trivy's version,
run the command below:

``` bash
docker run aquasec/trivy:latest version
```

## Scanning a container image

To run a vulnerability scan on a container image,
just run the command below:

``` bash
trivy image nginx

docker run aquasec/trivy:latest image nginx
```

Where **nginx** is the name of the container image.

You should see output similar to the one below:

![trivy image nginx result with the table of vulnerabilities by library and severity](/images/trivy-2.webp)

You can filter by HIGH and CRITICAL severity with the command below:

``` bash
trivy image nginx --severity=HIGH --severity=CRITICAL
```

Now only vulnerabilities with HIGH
and CRITICAL severity should show up, as in the image below:

![trivy image nginx result filtered by HIGH and CRITICAL severity](/images/trivy-3.webp)

In the **status** column, we can see the status is
**affected**: that means there's no fixed version
for these vulnerabilities yet.

## Scanning a repository

Now let's run a vulnerability scan on the
[Prometheus](https://github.com/prometheus/prometheus) repository, running the
command below and filtering only vulnerabilities with HIGH severity.

``` bash
trivy repository https://github.com/prometheus/prometheus --severity=HIGH
```

Below is the command's output:

![trivy repository result on the Prometheus repository with HIGH severity vulnerabilities](/images/trivy-4.webp)

In this case, the **status** column is **fixed**: that means
a fixed version already exists. The **Fixed Version** column shows the
version in which the vulnerability was fixed.

## Integrations

Trivy integrates with several tools that let you add
security to your application's workflow, such as CI/CD tools and
IDEs. See all the integrations in the [Trivy ecosystem
documentation](https://aquasecurity.github.io/trivy/v0.46/ecosystem/).

## Conclusion

The goal of this article was to show a bit of how to use Trivy
to run vulnerability scans.

For static analysis of infrastructure code, see the post about
[kics]({{< relref "/posts/2022/2022-09-10-conhecendo-a-ferramenta-de-analise-estatica-kics" >}}). And to see Trivy running alongside other
scanners and an AI review on pull requests, see
[Infrastructure PR review with AI, part 3]({{< relref "/posts/2026/2026-11-03-revisao-de-pr-de-infraestrutura-com-ia-parte-3" >}}).
