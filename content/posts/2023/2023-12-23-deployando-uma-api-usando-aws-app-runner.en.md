---
title: "Deploying an API with AWS App Runner"
slug: "2023-12-23-deploying-an-api-with-aws-app-runner"
description: "Second article about AWS App Runner: deploying a Hello World API built with FastAPI."
images: ["/images/capa-deployando-uma-api-usando-aws-app-runner.png"]
author: Jason
date: 2023-12-23
lastmod: 2026-11-04
tags: [ "AWS", "Containers" ]
type: post
---

> **Update (November 2026):** AWS App Runner
> [no longer accepts new customers](https://docs.aws.amazon.com/apprunner/latest/dg/apprunner-availability-change.html).
> Existing users can keep using it, including creating new services, but AWS
> doesn't plan new features and recommends Amazon ECS Express Mode for
> migrating. This post remains as a record.

In this second article about App Runner I'll show how to create
an API built with FastAPI that returns a Hello World. The source code
for this API is in the
[apprunner-example](https://github.com/jjasonhenrique1986/apprunner-example.git) repository
on GitHub.

The first article about App Runner is [Using AWS App Runner to
deploy an
application]({{< relref "/posts/2023/2023-12-02-usando-o-aws-app-runner-para-subir-uma-aplicacao" >}}).

## Prerequisites

Below are the prerequisites to follow this article:

-   An AWS account
-   A GitHub repository with the source code mentioned above.

## Configuring your GitHub account

As a first step, we need to connect your GitHub account to your
AWS account. To do that, go to **App Runner** \>\> **Connected
Accounts** and you should see the screen below:

![App Runner Connected accounts screen with no connections yet](/images/apprunner-11.webp)

Now click **Add New** and select the provider. In
my case it's GitHub, but you can also use Bitbucket.
You'll be redirected to a screen to sign in to your
GitHub account to set up the **AWS Connector for GitHub**, as below:

![GitHub sign-in screen to authorize the AWS Connector for GitHub](/images/apprunner-12.webp)

Enter the name of your connection (in my case, **GitHub**) and click the
**Install another** button, as in the image below:

![Creating a new connection called Github in App Runner](/images/apprunner-13.webp)

On the next screen, select only the repository that holds our
API's source code, as in the image below:

![Installing the AWS Connector for GitHub with access only to the apprunner-example repository](/images/apprunner-14.webp)

Click **Install**; your connection to
GitHub will be created successfully and you'll go back to the page below:

![Github connection with the account's GitHub app selected](/images/apprunner-15.webp)

Now click **Next** to finish the configuration
on AWS, and you should see a screen similar to the one below confirming that
the connection between your AWS account and your GitHub account succeeded.

![GitHub connection with Available status in Connected accounts](/images/apprunner-16.webp)

## Deploying the API

Now let's deploy our API with AWS App Runner. Find
the AWS App Runner service, go to **Create an App Runner Service** and
select the **Source Code Repository** method. You should see
a screen like the one below:

![Service source configured as a GitHub code repository, apprunner-example repository and main branch](/images/apprunner-17.webp)

The information about the repository with our API's source code
was loaded automatically. Now just click Next and we'll
configure
the build options, as in the image below:

![Build configuration with the Python 3 runtime, pip install -r requirements.txt command, uvicorn start command and port 8080](/images/apprunner-18.webp)

On the next screen we'll configure the service name, CPU and
memory, Autoscaling, Health Check and
other settings. For more information about these settings, see the
[first article about AWS App
Runner]({{< relref "/posts/2023/2023-12-02-usando-o-aws-app-runner-para-subir-uma-aplicacao" >}}).

![Configuration of the API-Python service with 0.25 vCPU and 0.5 GB of memory](/images/apprunner-19.webp)

In the Health Check option, change the protocol to HTTP, as below:

![HTTP health check configuration for the API-Python service on path /](/images/apprunner-20.webp)

Then move on to the next screen and click **Create &
Deploy.** On the screen below we can see that the API was deployed
successfully.

![App Runner event logs with the build of the GitHub code and the deploy completed successfully](/images/apprunner-21.webp)

To access the API, just open the link shown in the **Default
Domain** field. You should see a screen like the image below:

![Browser opening the App Runner domain and showing Hello World](/images/apprunner-22.webp)

## Conclusion

The purpose of this article was to show the deployment of a simple API
hosted on GitHub running on the AWS App Runner service.
