---
title: "Remote Terraform State configuration with an S3 bucket on AWS"
slug: "2024-08-11-remote-terraform-state-with-an-s3-bucket-on-aws"
description: "How to configure remote Terraform state in an S3 bucket on AWS instead of keeping it locally."
images: ["/images/capa-configuracao-de-remote-terraform-state-com-bucket-s3-na-aws.png"]
author: Jason
date: 2024-08-11
lastmod: 2026-11-03
tags: [ "IaC", "AWS", "Terraform" ]
type: post
---

The goal of this post is to configure a remote Terraform
state using an S3 bucket instead of keeping your Terraform state
locally, and also to create a resource on AWS to test that
Terraform works.

## Prerequisites

To set up your remote Terraform state you'll need
the following prerequisites:

-   An AWS account
-   An IAM user
-   AWS CLI installed.
-   A GitHub account
-   An S3 bucket
-   Terraform CLI installed

## Creating the IAM user

In this first step we'll create an IAM user to use
in the Terraform configuration files.

To create the IAM user, follow the post [Creating and configuring an IAM
user]({{< relref "/posts/2024/2024-01-07-criando-e-configurando-um-usuario-iam" >}}).

For our setup we'll attach the **AdministratorAccess** policy to
our **blog** user, as in the image below:

![IAM console with the AdministratorAccess policy attached to the blog user](/images/terraform-2.webp)

I attached this policy because it's just a lab. In production environments,
only the required permissions should be granted.

## Creating the S3 bucket

In this next step we'll create our S3 bucket, which will
store our Terraform state. To do that, go to the AWS
management console, look for the S3 service and click **Create
Bucket**. You should see the screen below:

![Create bucket screen in the S3 console](/images/terraform-3.webp)

In my case I'll name the bucket terraform-blog-jason, and another
detail is that this bucket will be created in the us-east-1 region.

We should leave the **Block all public access** option enabled, as in the
image below:

![Block all public access option enabled when creating the S3 bucket](/images/terraform-4.webp)

After that, click the **Create bucket** button at the bottom of the screen.
Our bucket will then be created, as in the image below:

![terraform-blog-jason bucket created in the S3 console](/images/terraform-5.webp)

## Creating the GitHub repository

Now let's create a GitHub repository to store our
Terraform configuration files, as below:

![Creating the GitHub repository with the Terraform .gitignore template](/images/terraform-6.webp)

In the Add .gitignore option, add the Terraform template. The repository
created is
[terraform-blog-jason](https://github.com/jjasonhenrique/terraform-blog-jason).

Now let's clone this repository with the command below:

```bash
git clone https://github.com/jjasonhenrique/terraform-blog-jason.git
```

## Creating the configuration files

Now let's create the Terraform configuration files.
As a prerequisite, the IAM user must already be configured in the AWS CLI,
as in the article I wrote earlier and linked at the
beginning of this article.

The first file we'll create is **versions.tf**, as below:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

Next we'll create the **provider.tf** file, as below:

```hcl
provider "aws" {
  shared_config_files      = ["/home/jason/.aws/config"]
  shared_credentials_files = ["/home/jason/.aws/credentials"]
  profile                  = "blog"
}
```

In this file we pass the user that will be used to connect to
AWS.

Now let's create the **main.tf** file, as below:

```hcl
terraform {
  backend "s3" {
    bucket  = "terraform-blog-jason"
    key     = "terraform.tfstate"
    region  = "us-east-1"
    profile = "blog"
  }
}
```

In the file above we're configuring an S3 bucket on AWS
to be used as the remote Terraform state.

> **Update (November 2026):** the configuration above has no **state
> locking**, so two people (or two pipelines) running `terraform
> apply` at the same time can corrupt the state. See how to enable it in the
> [State locking on S3](#state-locking-on-s3) section.

Now let's create a resource on AWS by creating the **ecs.tf** file, as
below:

```hcl
resource "aws_ecs_cluster" "teste-blog" {
  name = "teste-blog"

  setting {
    name  = "containerInsights"
    value = "disabled"
  }
}
```

Keep in mind that creating an ECS cluster on AWS involves more
configuration. This is just a test to validate that the
remote Terraform state works.

After creating the files, commit them by running the
commands below:

```bash
git add .
git commit -m "Create ECS cluster"
git push
```

## Validation

Now let's validate the remote Terraform state and the creation of a resource
on AWS. First we'll initialize our Terraform state
by running the command below:

```bash
terraform init
```

The command output should be similar to the one below:

![Output of terraform init configuring the S3 backend](/images/terraform-7.webp)

Now let's run terraform plan to check what will be
created on AWS. The output should be similar to the one below:

![Output of terraform plan with the ECS cluster that will be created](/images/terraform-8.webp)

Now let's run terraform apply -auto-approve to
create the resource on AWS.

To validate, go to the ECS service and check whether the ECS cluster was
created. It should have been created as in the image below:

![teste-blog ECS cluster created in the AWS console](/images/terraform-9.webp)

Let's also check whether a terraform.tfstate file was created in the
bucket, as in the image below:

![terraform.tfstate file stored in the S3 bucket](/images/terraform-10.webp)

To destroy the resource created on AWS for this test, run the command
below:

```bash
terraform destroy --auto-approve
```

## State locking on S3

State locking prevents two runs from changing the same state at the same
time: whoever runs first takes a lock, and the second run waits or
fails. This used to require a DynamoDB table
(`dynamodb_table`). Since Terraform 1.11 the S3 backend locks
directly in the bucket (the option arrived in 1.10 and became stable in 1.11), creating a
`.tflock` file next to the state, and the DynamoDB option was
deprecated.

To enable it, just add `use_lockfile = true` to the backend and require
a Terraform version that supports the option:

```hcl
terraform {
  required_version = ">= 1.11"

  backend "s3" {
    bucket       = "terraform-blog-jason"
    key          = "terraform.tfstate"
    region       = "us-east-1"
    profile      = "blog"
    use_lockfile = true
  }
}
```

After changing the backend, run `terraform init -reconfigure` so
Terraform applies the new configuration. The identity that runs Terraform
needs permission to read, create and delete the
`terraform.tfstate.tflock` object in the bucket (`s3:GetObject`, `s3:PutObject` and
`s3:DeleteObject`),
on top of the permissions it already used for the state.

It's also worth enabling **versioning** on the bucket: if a state is
corrupted or deleted by mistake, you can recover a previous version.
More details in the
[S3 backend documentation](https://developer.hashicorp.com/terraform/language/backend/s3).

## Conclusion

The goal of this article was to show how to configure
a remote Terraform state using an S3 bucket on AWS.

For the next step with Terraform, see
[how to use the for_each expression to create subnets on AWS]({{< relref "/posts/2024/2024-08-18-como-usar-a-expressao-for_each-para-criar-subnets-na-aws-com-terraform" >}}).
