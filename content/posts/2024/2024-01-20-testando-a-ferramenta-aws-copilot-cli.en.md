---
title: "Testing the AWS Copilot CLI tool"
slug: "2024-01-20-testing-the-aws-copilot-cli-tool"
description: "Testing AWS Copilot CLI to easily create a containerized application running on ECS."
images: ["/images/capa-testando-a-ferramenta-aws-copilot-cli.png"]
author: Jason
date: 2024-01-20
lastmod: 2026-11-04
tags: [ "AWS", "Containers", "ECS" ]
type: post
---

> **Update (November 2026):** AWS ended support for Copilot
> CLI on June 12, 2026, and the
> [repository](https://github.com/aws/copilot-cli) was archived. The
> tool no longer receives updates or security fixes, so
> don't use it in new projects. As an
> [alternative](https://aws.amazon.com/blogs/containers/announcing-the-end-of-support-for-the-aws-copilot-cli),
> AWS recommends Amazon ECS Express Mode or AWS CDK. This post remains as a
> record.

In this article I'll show how to use the AWS Copilot CLI tool to
easily create a containerized application running on
ECS. With AWS Copilot CLI you can also manage
the application's whole infrastructure, such as infrastructure as code,
storage, secrets and the pipeline, all from the command line.

For more information, see the [AWS Copilot
CLI documentation](https://aws.github.io/copilot-cli/docs/overview/).

## Prerequisites

Below are the prerequisites to follow this tutorial:

-   An AWS account
-   AWS Copilot CLI installed on your computer
-   An IAM user configured on your machine

## Installation

To install AWS Copilot, run the commands below:

``` bash
sudo curl -Lo /usr/local/bin/copilot https://github.com/aws/copilot-cli/releases/latest/download/copilot-linux \
   && sudo chmod +x /usr/local/bin/copilot \
   && copilot --help
```

You should see a screen similar to the one below:

![copilot -h output with the commands available in Copilot CLI](/images/copilot-2.webp)

In my case, I used a Linux computer. For other operating
systems, see the [Copilot installation
guide](https://aws.github.io/copilot-cli/docs/getting-started/install/).

## Configuring the AWS IAM user

As a prerequisite for using AWS Copilot CLI, you need to configure
an IAM user on your computer. To do that, follow the post [Creating and
configuring an IAM
user]({{< relref "/posts/2024/2024-01-07-criando-e-configurando-um-usuario-iam" >}}).

For this test I'll use the **AdministratorAccess** policy. If you use
a profile other than default in the .aws/credentials file, you'll
need to export the **AWS_PROFILE** environment variable or add
it to your .bashrc or .zshrc file. In my case I'll export the
variable by running the command below:

``` bash
export AWS_PROFILE="blog"
```

To check the variable's value, run the command below:

``` bash
echo $AWS_PROFILE
```

## Deploying the application

To test AWS Copilot CLI we'll deploy the sample
application
[amazon-ecs-cli-sample-app](https://github.com/aws-samples/amazon-ecs-cli-sample-app.git).
The first step is to git clone this repository
using the command below:

``` bash
git clone https://github.com/aws-samples/amazon-ecs-cli-sample-app.git example 
```

Then go into the example folder and run the **copilot init** command.
You'll see the screen below:

![copilot init asking for the application name, filled in with teste-app-blog](/images/copilot-3.webp)

Enter the application name. In my case it will be teste-app-blog.

On the next screen, you'll be asked to choose your application's
workload type. For this demo we'll use the **Load
Balanced Web Service** option, but there are other options, as below:

![List of Copilot workload types, such as Request-Driven Web Service, Load Balanced Web Service and Scheduled Job](/images/copilot-4.webp)

Now choose a name for your service. In my case it will be
teste-app-blog-svc, as below:

![Copilot asking for the service name, filled in with teste-app-blog-svc](/images/copilot-5.webp)

Now you'll be asked for the location of the Dockerfile or
the location of an application image. We'll use the
Dockerfile at the root of the repository.

![Copilot asking which Dockerfile to use, with ./Dockerfile selected](/images/copilot-6.webp)

In this next step, all the infrastructure roles needed for
our application to work will be created. After that,
you'll be asked whether you'd like to deploy your application,
as in the image below:

![Copilot creating the application's infrastructure roles and the service manifest, and asking whether to deploy an environment](/images/copilot-7.webp)

On the next screen, you'll be asked to create an environment. This
environment will be called dev, as in the image below:

![Copilot asking for the environment name, filled in with dev](/images/copilot-8.webp)

On the next screen, the whole infrastructure will be created, such as the ECR
repository, an S3 bucket, the networking and the ECS cluster.

![Copilot creating the dev environment in sa-east-1 with VPC, subnets, Internet Gateway and ECS cluster](/images/copilot-9.webp)

On the next screen, AWS Copilot CLI builds our application's image
and pushes it to the ECR repository created earlier,
as below:

![Building the service's Docker image and pushing it to the ECR repository](/images/copilot-10.webp)

On the next screen the application is configured in the ECS cluster,
deployed, and the application URL is made available, as in
the image below:

![Copilot creating the load balancer, the task definition and the ECS service and showing the service URL](/images/copilot-11.webp)

Opening the application URL should return a page similar to the one below:

![Browser opening the load balancer URL and showing the page with the AWS Copilot logo](/images/copilot-12.webp)

## Interacting with AWS Copilot CLI

With the application already deployed to your ECS cluster, we can use AWS
Copilot CLI to look up information about our application. Below are
some examples:

``` bash
copilot svc status
```

![copilot svc status output with the task running and healthy health checks](/images/copilot-13.webp)

``` bash
copilot svc ls
```

![copilot svc ls output listing the teste-app-blog-svc service of type Load Balanced Web Service](/images/copilot-14.webp)

``` bash
copilot app show
```

![copilot app show output with the teste-app-blog application, the dev environment in sa-east-1 and the service](/images/copilot-15.webp)

There are many other options you can use with AWS Copilot CLI.

## Deleting your application

Now let's delete the application we created with the command below:

``` bash
copilot app delete
```

You'll be asked for confirmation and then all the resources
created by AWS Copilot CLI will be deleted, as in the image
below:

![copilot app delete output removing the service, the environment and the application's resources](/images/copilot-16.webp)

## Conclusion

The goal of this article was to show how to use the AWS
Copilot CLI tool, deploying the whole infrastructure and a test
application.
