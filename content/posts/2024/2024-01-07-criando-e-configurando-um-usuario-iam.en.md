---
title: "Creating and configuring an IAM user"
slug: "2024-01-07-creating-and-configuring-an-iam-user"
description: "A tutorial to create and configure an IAM user with an Access Key and Secret Access Key and install the AWS CLI."
images: ["/images/capa-criando-e-configurando-um-usuario-iam.png"]
author: Jason
date: 2024-01-07
lastmod: 2026-11-03
tags: [ "AWS", "Security" ]
type: post
---

> **Update (November 2026):** an IAM user with a long-lived access key
> is no longer the recommended way to access AWS from the command line, because
> the key doesn't expire and can leak. For people, prefer IAM Identity
> Center, which hands out temporary credentials with MFA: see
> [AWS access without access keys with IAM Identity Center]({{< relref "/posts/2026/2026-10-31-acesso-a-aws-sem-access-keys-com-iam-identity-center" >}}).
> For pipelines, use OpenID Connect, as in
> [GitHub Actions with OpenID Connect on AWS, without access keys]({{< relref "/posts/2026/2026-08-22-github-actions-com-oidc-na-aws-sem-access-keys" >}}).
> The step by step below is still useful to understand IAM.

The goal of this article is to create a tutorial showing how to create and
configure an IAM user on AWS so you can interact with it
programmatically using an **Access Key** and a **Secret Access Key**.

I'll also show how to install the AWS CLI to manage your
AWS services.

## Prerequisites

Below are the prerequisites to follow this tutorial:

-   An AWS account
-   A Linux machine

## Creating the IAM user

As a first step, sign in to your AWS account, open the
**IAM** service, go to **Users** and then click **Create
User**. You'll see the screen below:

![Specify user details screen when creating the blog IAM user, without console access](/images/iam-2.webp)

Don't enable the **Provide user access to the AWS Management
Console** option, since it grants access to the AWS management console.

Enter a name for the IAM user and click **Next**.

On the next screen we'll add the permissions for this IAM user. In
my case I'll add a policy with full access to S3, as in the screen
below:

![Set permissions screen with the AmazonS3FullAccess policy attached directly to the user](/images/iam-3.webp)

Keep in mind that you can customize this policy.

Then click Next and you'll see a review of
your user's creation, as below:

![Review and create screen for the blog user with the AmazonS3FullAccess policy](/images/iam-4.webp)

After that, click **Create User** and your user will be
created. It should show up as in the screen below:

![User created successfully message and the blog user in the IAM list](/images/iam-5.webp)

## Creating your user's access key

Now let's create the access key: click your IAM user, go to the
Security credentials tab and click Create access key, as in the
image
below:

![The user's Security credentials tab with the Access keys section still empty](/images/iam-6.webp)

On the next screen, choose the first option, **Command Line
Interface (CLI)**, check **I understand the above
recommendation and want to proceed to create an access key** and click
Next, as in the image below:

![Access key best practices & alternatives screen with the Command Line Interface (CLI) use case selected](/images/iam-7.webp)

On the next screen you can set a description for your access key and
then click **Create access key**, as in the image below:

![Access key description tag filled in with "Para post no blog" (for a blog post)](/images/iam-8.webp)

After clicking **Create access key**, the **Access
key** and the **Secret access key** are generated, as in the image below:

![Retrieve access keys screen with the access key created, with the ID redacted](/images/iam-9.webp)

Remember that the **Secret access key** is only available at this
moment. It's recommended to download the CSV file with the values
of the **Access key** and the **Secret access key** and then click **Done**
to go back to the main screen, as below:

![Active access key in the user's Security credentials tab, with the ID redacted](/images/iam-10.webp)

AWS has a page with [best practices for managing access
keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html#securing_access-keys).

## Installing the AWS CLI

Now let's install the AWS CLI so we can configure the
**Access key** and the **Secret access key** generated in the previous step.
Below are the installation commands. For more information, see the
[AWS CLI installation
guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

To check whether the installation succeeded, run `aws --version`:

![Output of aws --version showing AWS CLI 2.13.34](/images/iam-11.webp)

## Configuring your access key

Now let's configure your Access key and Secret
access key on your computer. Run the aws
configure command as below. In my case I'll add the --profile flag to
create a new profile in the .aws/credentials file.

![aws configure --profile blog command with the access key and secret redacted and the sa-east-1 region](/images/iam-12.webp)

## Testing your access key

Now let's test CLI access by trying to list the S3 buckets in your
account, running the command below:

```bash
aws s3 ls --profile blog
```

It should return the following result:

![Output of aws s3 ls --profile blog listing the teste-blog bucket](/images/iam-13.png)

Remember that this Access key and Secret access key only have access
to S3. If you try to run the command **aws ec2
describe-availability-zones
--profile blog**, it should return the following error due to missing
permissions:

``` text
An error occurred (UnauthorizedOperation) when calling the DescribeAvailabilityZones operation: You are not authorized to perform this operation. User: arn:aws:iam::account_id:user/blog is not authorized to perform: ec2:DescribeAvailabilityZones because no identity-based policy allows the ec2:DescribeAvailabilityZones action
```

## Conclusion

The goal of this article was to create a tutorial on creating and
configuring an IAM user with an Access key and Secret access key
to interact with your AWS account through the CLI.

Keep in mind that the Access key and Secret access key from this post no longer
exist. Generate your own values in your account if you want to follow the
article.
