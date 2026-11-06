---
title: "Getting to know the kics static analysis tool"
slug: "2022-09-10-getting-to-know-the-kics-static-analysis-tool"
description: "How to use KICS to run static analysis on Infrastructure as Code, such as Terraform and CloudFormation."
images: ["/images/capa-conhecendo-a-ferramenta-de-analise-estatica-kics.png"]
author: Jason
date: 2022-09-10
tags: [ "CI/CD", "Security", "IaC" ]
type: post
---
In this article I'll show how to use a tool called
**kics** (**keeping infrastructure as code secure**), which runs
static analysis on Infrastructure
as Code, such as Terraform, CloudFormation and others.

kics is an open source tool written in Golang using OPA
(Open Policy Agent). For more information, see the [OPA
documentation](https://www.openpolicyagent.org/docs/latest/#overview). OPA
lets you apply policies to your
microservices, Kubernetes, CI/CD pipelines, API gateways and many
other things.

kics runs a static analysis looking for security
vulnerabilities, compliance issues and misconfigurations in your
Infrastructure as Code tools. At the time of writing this
article, the following tools are supported:

![Logos of the tools supported by KICS, such as Terraform, Kubernetes, Docker, Ansible, CloudFormation, Helm, OpenAPI, AWS CDK and gRPC](/images/kics-2.webp)

With kics you can create new queries according to your
environment's needs. See how in the [kics query creation
documentation](https://docs.kics.io/latest/creating-queries/).

## Installation

To use kics you have two options: install it from the
source code or use kics through a Docker image. In my
case I'll use a Docker image.

### Docker

``` bash
docker pull checkmarx/kics:latest
docker run -t -v "{path_to_host_folder_to_scan}":/path checkmarx/kics scan -p "/path" -o "/path/"
```

For more details, see the [kics installation
guide](https://docs.kics.io/latest/getting-started/).

## Demo

For this demo I'll use the
[addamstj/tfcourse](https://github.com/addamstj/tfcourse) repository, with
Terraform examples. By default
kics generates a JSON file; to make it easier to read, we'll generate
an HTML file.

To do that, just run the command below:

``` bash
nerdctl run -it -v "C:\Users\jhmrj\Documents\tfcourse":/tmp checkmarx/kics scan -p "/tmp/" -o "/tmp/results" --report-formats "html"
```

**Note:** I'm using the nerdctl utility because my
machine runs containerd. It works the same way with Docker.

![Terminal running KICS in a container with nerdctl and the scan summary: 21 files scanned and 1040 queries loaded](/images/kics-3.webp)

Below are some of the rules flagged by kics:

![KICS result flagging the Variable Without Description rule on Terraform variables](/images/kics-4.png)

![KICS result flagging the RDS With Backup Disabled rule, with medium severity](/images/kics-5.webp)

![KICS result flagging the EC2 Instance Has Public IP rule, with high severity, on aws_instance resources](/images/kics-6.png)

![KICS scan summary: 10 HIGH, 19 MEDIUM, 11 LOW and 64 INFO findings, with the report saved to /tmp/results/results.html](/images/kics-7.png)

An HTML file was generated at /tmp/results/results.html. Below
are some screenshots of this report.

![KICS HTML report with the total vulnerabilities by severity and the DB Instance Storage Not Encrypted finding](/images/kics-8.webp)

![Detail of the KICS HTML report for the EC2 Instance Has Public IP finding, with the affected files and lines](/images/kics-9.webp)

## Integrations

kics integrates with several CI/CD tools and
there's also a VS Code plugin.
All available integrations are in the [kics integrations
documentation](https://docs.kics.io/latest/integrations/).

For vulnerability scanning of container images, which complements
IaC analysis, see the post about [Trivy]({{< relref "/posts/2023/2023-11-05-conhecendo-a-ferramenta-trivy" >}}).

## References

- [kics on GitHub](https://github.com/Checkmarx/kics)
- [Official documentation](https://docs.kics.io/latest/getting-started/)
