---
title: "Using AWS App Runner to deploy an application"
slug: "2023-12-02-using-aws-app-runner-to-deploy-an-application"
description: "How to use AWS App Runner to deploy an application from a public ECR image."
images: ["/images/capa-usando-o-aws-app-runner-para-subir-uma-aplicacao.png"]
author: Jason
date: 2023-12-02
lastmod: 2026-11-04
tags: [ "AWS", "Containers" ]
type: post
---

> **Update (November 2026):** AWS App Runner
> [no longer accepts new customers](https://docs.aws.amazon.com/apprunner/latest/dg/apprunner-availability-change.html).
> Existing users can keep using it, including creating new services, but AWS
> doesn't plan new features and recommends Amazon ECS Express Mode for
> migrating. This post remains as a record.

The goal of this article is to show how to use the AWS App
Runner service. We'll deploy an application using a public ECR image to
test the service.

## What AWS App Runner is

AWS App Runner is an AWS service that lets you deploy and manage
web applications and APIs without having to manage the infrastructure, such as the load
balancer and autoscaling. With it, you can get an application up
quickly.

For more information, see the [AWS App
Runner page](https://aws.amazon.com/apprunner/).

## Application from a public ECR

We'll deploy this application from the public ECR: the
[hello-app-runner](https://gallery.ecr.aws/aws-containers/hello-app-runner) image.

Sign in to your AWS account, look for the **AWS App Runner** service and
click **Create an App Runner Service**. You'll see the
screen below:

![App Runner Source and deployment screen with the Container registry and Amazon ECR options](/images/apprunner-2.webp)

We can use AWS App Runner from a Container Registry, with
ECR being the only one supported. You can use a public
ECR or a private ECR. Below are the settings used
to deploy an application from a public ECR:

![Source configuration with Amazon ECR Public, the hello-app-runner image and manual deployment](/images/apprunner-3.webp)

On the next screen we'll configure the service name, the amount of CPU
and memory to use, the port and, if needed, environment variables.

![Configuration of the AppECR service with 0.5 vCPU, 1 GB of memory and port 8000](/images/apprunner-4.webp)

On the next screen we'll configure the application's auto scaling. Since this is
just a test, we'll use the default settings, but you can create
your own by clicking the Create button.

![Default App Runner auto scaling configuration: concurrency of 100 requests, from 1 to 25 instances](/images/apprunner-5.webp)

On the next screen, configure the Health Check as below:

![App Runner HTTP health check configuration on path /](/images/apprunner-6.webp)

We can also configure security, networking,
observability and tags. The next screen shows a review of
your application's settings, and then you click the
**Create & Deploy** button.

On the next screen we can follow the deploy's progress, as
below:

![App Runner event logs showing the image deploy and the successful health check](/images/apprunner-7.webp)

You can also check the application logs, as in the image below:

![Application logs in App Runner with GET requests answered with status 200](/images/apprunner-8.webp)

To test the application, just click the address generated under
Default Domain, as in the image below:

![AppECR service overview with Running status and the App Runner default domain](/images/apprunner-9.webp)

You should see a page like the one below:

![App Runner sample application page with the message And we're live!](/images/apprunner-10.webp)

## Conclusion

The goal of this article was to show how to use AWS App Runner
with the Container Registry method, deploying an application
from a public ECR. In the following article I showed how to use
AWS App
Runner with an API hosted on GitHub:
[Deploying an API with AWS App Runner]({{< relref "/posts/2023/2023-12-23-deployando-uma-api-usando-aws-app-runner" >}}).
