---
title: "How to use the for_each expression to create subnets on AWS with Terraform"
slug: "2024-08-18-how-to-use-for-each-to-create-subnets-on-aws-with-terraform"
description: "How to use the for_each expression in Terraform to write less code when creating subnets on AWS."
images: ["/images/capa-como-usar-a-expressao-for_each-para-criar-subnets-na-aws-com.png"]
author: Jason
date: 2024-08-18
tags: [ "IaC", "AWS", "Terraform" ]
type: post
---

The goal of this article is to show how to use the for_each expression
to reduce the amount of code needed to create your
infrastructure with Terraform. In our example we'll streamline the creation
of subnets on AWS.

## Prerequisites

Below are the prerequisites to follow this article:

-   An AWS account.
-   The Terraform CLI installed.
-   A GitHub account.
-   A GitHub repository already set up to use Terraform.
    You can use the
    [terraform-blog-jason](https://github.com/jjasonhenrique/terraform-blog-jason) repository as an example.

## What is for_each?

**`for_each`** is an expression that lets you iterate over a collection of
items to create resources in Terraform. It's useful when you need to
create multiple resources that share a common structure but
have some variations, for example each subnet's CIDR block and
availability zone (AZ).

## Creating a VPC

To create the subnets, you need a VPC on AWS.
Create a file called **vpc.tf** with the code below:

``` hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "main-vpc"
  }
}
```

## Creating the subnets

In this next step, let's imagine we need to create multiple
subnets in different availability zones. To do that we'll
use the for_each expression so there's no repeated code.

Now let's declare the subnets variable in the **variables.tf** file
as below:

``` hcl
variable "subnets" {
  type         = map(any)
  default      = {}
  description  = "Subnets to create on AWS"
}
```

Now let's set the variable's value in the
**terraform.tfvars** file as below:

``` hcl
subnets = {
    subnet1 = {
        az  = "us-east-1a"
        cidr = "10.0.1.0/24"
    },
    subnet2 = {
        az  = "us-east-1b"
        cidr = "10.0.2.0/24"
    },
    subnet3 = {
        az  = "us-east-1c"
        cidr = "10.0.3.0/24"
    }
}
```

Now let's create the aws_subnet resource using the for_each expression so
we can iterate over the map declared above. Create a
file called **subnets.tf** with the code below:

``` hcl
resource "aws_subnet" "subnets" {
  for_each = var.subnets

  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value["cidr"]
  availability_zone = each.value["az"]

  tags = {
    Name = each.key
  }
}
```

We use `for_each` to iterate over the `subnets` map. For each item
in the map, a subnet is created in the VPC. Each subnet gets a CIDR and is
assigned to an availability zone.

We use each.key to set the name of each subnet.

With this, we don't need to repeat the code for each subnet, which makes
managing the resources more efficient.

## Validation

To validate, just run the commands below:

``` bash
terraform init
terraform plan -out plan
terraform apply plan
```

You should see output like the image below:

![terraform apply output creating the VPC and the three subnets with for_each](/images/terraform-11.webp)

You can also check in the AWS console, as in the image below:

![VPC console with the subnets subnet1, subnet2 and subnet3 created in different availability zones](/images/terraform-12.webp)

## Advantages of using for_each

Below are some advantages that, in my opinion, make for_each worth
using:

-   **Configuration flexibility**: by defining subnets in a map,
    you can easily adjust the CIDR or the AZ without duplicating code.
-   **Scalability**: adding new subnets is simple and only requires
    adding a new item to the `subnets` map, without changing the
    resource's main code.
-   **Maintenance**: since each subnet is identified by a unique key
    (`subnet1`, `subnet2`, etc.), you can manage or update
    specific subnets without affecting the others.

## Conclusion

The goal of this article was to show how to use for_each to
create subnets, but it can be used to create other
resources on AWS too.

To store this code's state in an S3 bucket, with state locking,
see [Remote Terraform State configuration with an S3 bucket on AWS]({{< relref "/posts/2024/2024-08-11-configuracao-de-remote-terraform-state-com-bucket-s3-na-aws" >}}).
