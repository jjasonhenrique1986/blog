---
title: "Creating and configuring an IAM role"
slug: "2024-03-30-creating-and-configuring-an-iam-role"
description: "A tutorial to create and configure an IAM role on AWS, a more secure way to access AWS than using an IAM user."
images: ["/images/capa-criando-e-configurando-uma-role-iam.png"]
author: Jason
date: 2024-03-30
tags: [ "AWS", "Security" ]
type: post
---

The goal of this article is to create a tutorial showing how to create and
configure an IAM role on AWS so you can interact with AWS
programmatically. Using an IAM role is more secure than using
an IAM user directly, as I showed in [Creating and configuring an
IAM
user]({{< relref "/posts/2024/2024-01-07-criando-e-configurando-um-usuario-iam" >}}).

Some of the screenshots below show the AWS console in Portuguese; in the
text I use the English names of the console options.

## Prerequisites

Below are the prerequisites to follow this tutorial:

-   An AWS account
-   A Linux machine
-   An IAM user without any permissions
-   AWS CLI installed

## Creating the IAM user

As a first step, sign in to your AWS account, open the **IAM**
service, go to **Users** and then click **Create
User**. You'll see the screen below:

![Specify user details screen when creating the blog IAM user](/images/iam-14.webp)

Don't enable the **Provide user access to the AWS Management
Console** option, since it grants access to the AWS management console.

Enter a name for the IAM user and click **Next**.

On the next screen, don't add any permissions to this IAM user: the
permission will be granted by the IAM role, through an IAM policy.

After that, click **Create User** and your user will be created.

To learn how to generate the **Access Key** and the **Secret Access Key** and how to
configure them, see the [post about the IAM
user]({{< relref "/posts/2024/2024-01-07-criando-e-configurando-um-usuario-iam" >}}).

## Creating an IAM role

To demonstrate how a role works we'll create two IAM roles,
as below:

-   An IAM role with permission to list S3 buckets
-   An IAM role with permission to stop, start and reboot an EC2 instance

Before creating the roles we'll create two IAM policies, one for each role.

To do that, open the AWS management console, go to the
**IAM** service, then **Policies**, and click the
**Create policy** button. After clicking **Create policy**, you'll
see the screen below:

![IAM policy editor to create a policy](/images/iam-15.webp)

On the right side of the screen above, click the JSON option and replace the
content with the one below:

``` json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "s3:ListAllMyBuckets",
            "Resource": "*"
        }
    ]
}
```

Click Next and give the policy a name. In my case, the
name will be **s3-list**, and then click **Create
policy**.

Create another policy with the content below, named
**ec2-reboot**.

``` json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "ec2:RebootInstances",
                "ec2:StartInstances",
                "ec2:StopInstances"
            ],
            "Resource": "*"
        }
    ]
}
```

Now let's create two roles, one for each policy we created. To do that,
go to the **IAM** service, then **Roles**, and
click **Create role**. You should see the screen below:

![Creating the role with the AWS account trusted entity type, using this account](/images/iam-16.webp)

Select the **AWS account** option and click Next. After
that, select the **s3-list** policy created earlier and click
Next. You'll see the screen below:

![Name, review and create screen for the s3 role with the trust policy for sts:AssumeRole](/images/iam-17.webp)

Enter **s3** as the role name and click **Create
role**. Now we need to edit the trust policy of the
role we created, on the screen below:

![Summary of the s3 role with the trust policy that allows the blog user to assume the role](/images/iam-18.webp)

Now let's replace the trust policy content with the one
below:

``` json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::account_id:user/blog"
            },
            "Action": "sts:AssumeRole",
            "Condition": {}
        }
    ]
}
```

Where account_id is your AWS account number. That completes the
configuration of the **s3** role. Do the same for the
**ec2** role, attaching the **ec2-reboot** policy to it.

## Configuring the roles in the CLI

As a prerequisite, the user you created must be configured on your
machine with the AWS CLI. Remember that this user can't have any kind of access on AWS.

Now let's configure this user's access to the **s3** and
**ec2** roles. Add the following profiles to the .aws/config file:

``` ini
[profile s3]
role_arn = arn:aws:iam::account_id:role/s3
source_profile = blog

[profile ec2]
role_arn = arn:aws:iam::account_id:role/ec2
source_profile = blog
```

Where account_id is your AWS account number.

## Testing the roles

Now let's test the roles we created. For these tests, I created an S3
bucket and an EC2 instance. First, let's check whether the user
we created has any permissions. In my case, the IAM user is called blog.

![Access denied errors when trying to stop an EC2 instance and list S3 buckets with the blog user, with the account ID redacted](/images/iam-19.webp)

You should get the errors above for the blog user. Now let's
test the s3 role with the command below:

``` bash
aws s3 ls --profile s3
```

You should see the screen below, listing an S3 bucket:

![Output of aws s3 ls --profile s3 listing the teste-blog bucket with the assumed role](/images/iam-20.webp)

Now for the second test, where we'll stop an EC2 instance
using the ec2 role. To do that, run the command below:

``` bash
aws ec2 stop-instances --region sa-east-1 --instance-ids  i-0085922c8e050b223 --profile ec2
```

Running the command above shows the EC2 instance's state
changing from running to stopping, as in the image below:

![Output of aws ec2 stop-instances with the instance going from running to stopping](/images/iam-21.webp)

Once the instance has stopped, you can run the command below to
start the same EC2 instance.

``` bash
aws ec2 start-instances --region sa-east-1 --instance-ids  i-0085922c8e050b223 --profile ec2
```

You can also test terminating the EC2 instance by running the
command below. You'll get an error, since this role doesn't have that permission.

``` bash
aws ec2 terminate-instances --region sa-east-1 --instance-ids  i-0085922c8e050b223 --profile ec2
```

Finally, to make sure the s3 role can't start and stop an
EC2 instance, run one of the commands below:

``` bash
aws ec2 stop-instances --region sa-east-1 --instance-ids  i-0085922c8e050b223 --profile s3

aws ec2 start-instances --region sa-east-1 --instance-ids  i-0085922c8e050b223 --profile s3
```

## Conclusion

The goal of this article was to show how to create a role and run
tests using roles instead of the IAM user. In this article
I showed how to use a role within a single AWS account, but you can
also use multiple accounts, as in AWS's [cross-account access
with roles
tutorial](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html).

To go beyond roles assumed from an IAM user, see how to
access AWS with temporary credentials and MFA in
[AWS access without access keys with IAM Identity Center]({{< relref "/posts/2026/2026-10-31-acesso-a-aws-sem-access-keys-com-iam-identity-center" >}}),
or how to access EC2 instances without an SSH key in
[Accessing EC2 instances with Session Manager, without SSH]({{< relref "/posts/2025/2025-04-12-acessando-instancias-ec2-com-session-manager-sem-ssh" >}}).
