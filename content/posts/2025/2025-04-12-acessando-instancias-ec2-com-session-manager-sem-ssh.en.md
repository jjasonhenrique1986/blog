---
title: "Accessing EC2 instances with Session Manager, without SSH"
slug: "2025-04-12-accessing-ec2-instances-with-session-manager-without-ssh"
description: "How to access EC2 instances with AWS Systems Manager Session Manager, without opening port 22 and without an SSH key, including port forwarding."
images: ["/images/capa-session-manager.png"]
author: Jason
date: 2025-04-12
tags: [ "AWS", "Security" ]
type: post
---

The goal of this article is to show how to access an EC2 instance
using **Session Manager**, a feature of AWS Systems Manager. With it
you don't need to open port 22 in the security group, you don't have to
manage SSH keys and access is controlled by IAM permissions.

Besides that, each session is recorded in CloudTrail, and you can
log the content of the sessions to an S3 bucket or to CloudWatch Logs.

## Prerequisites

-   An AWS account and AWS CLI v2 configured. If you need to, see how to
    [create and configure an IAM user]({{< relref "/posts/2024/2024-01-07-criando-e-configurando-um-usuario-iam" >}})
    or, more securely,
    [an IAM role]({{< relref "/posts/2024/2024-03-30-criando-e-configurando-uma-role-iam" >}}).
-   The **Session Manager plugin** installed on your machine (step below).

## How it works

For an instance to show up in Session Manager, three things are
needed:

1.  The **SSM Agent** installed and running. It comes preinstalled on the
    Amazon Linux 2023 and Ubuntu Server AMIs, for example.
2.  An **instance profile** with a role that has the
    `AmazonSSMManagedInstanceCore` policy.
3.  **Outbound network access** from the instance to the Systems
    Manager endpoints, through the internet or a NAT gateway, or through VPC endpoints (`ssm`,
    `ssmmessages` and `ec2messages`) in private subnets.

Notice that the connection goes from the agent to AWS. That's why no
inbound rule is needed in the security group.

## Installing the Session Manager plugin

On Ubuntu/Debian:

```bash
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o "session-manager-plugin.deb"
sudo dpkg -i session-manager-plugin.deb
session-manager-plugin --version
```

For other operating systems, follow the
[official documentation](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html).

## Creating the role and the instance profile

Create a file called trust-ec2.json with the trust policy that allows the
EC2 service to assume the role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Now create the role, attach the managed policy and create the instance profile:

```bash
aws iam create-role \
  --role-name SSMInstanceRole \
  --assume-role-policy-document file://trust-ec2.json

aws iam attach-role-policy \
  --role-name SSMInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam create-instance-profile --instance-profile-name SSMInstanceProfile

aws iam add-role-to-instance-profile \
  --instance-profile-name SSMInstanceProfile \
  --role-name SSMInstanceRole
```

## Creating the instance

Let's launch an Amazon Linux 2023 instance in the default VPC, **with no key
pair**. The `resolve:ssm:` parameter fetches the latest AMI ID straight
from Parameter Store, so we don't have to look up the ID manually:

```bash
aws ec2 run-instances \
  --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --instance-type t3.micro \
  --iam-instance-profile Name=SSMInstanceProfile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ssm-demo}]' \
  --query 'Instances[0].InstanceId' --output text
```

Save the returned instance ID (something like `i-0abc...`). If the command
fails saying the instance profile doesn't exist, wait a few
seconds: IAM can take a little while to propagate.

After a minute or two, check whether the instance has been registered
in Systems Manager:

```bash
aws ssm describe-instance-information \
  --query "InstanceInformationList[].{Id:InstanceId,Status:PingStatus,OS:PlatformName}" \
  --output table
```

The status should show as **Online**.

## Starting a session

```bash
aws ssm start-session --target i-0abc123def4567890
```

That's it, you're inside the instance with a shell, without SSH. By default
the session opens as the `ssm-user` user. To leave, type `exit`.

The session can also be opened from the console: **EC2 → select the
instance → Connect → Session Manager**.

## Port forwarding

A very useful feature is port forwarding. For example, to
reach port 80 of an instance from your machine (assuming there's
a web server on it) through local port 8080:

```bash
aws ssm start-session \
  --target i-0abc123def4567890 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["80"],"localPortNumber":["8080"]}'
```

You can also use the instance as a bridge to a resource in a private
subnet, such as an RDS database, with the
`AWS-StartPortForwardingSessionToRemoteHost` document:

```bash
aws ssm start-session \
  --target i-0abc123def4567890 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["my-database.xxxx.us-east-1.rds.amazonaws.com"],"portNumber":["5432"],"localPortNumber":["5432"]}'
```

That way you connect to `localhost:5432` and reach the database, with no bastion
exposed to the internet.

## Controlling access with IAM

Who can open sessions is defined by IAM policies, using the
`ssm:StartSession` action. An example policy that allows opening sessions
only on instances with the tag `Environment=dev`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ssm:StartSession",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": { "ssm:resourceTag/Environment": "dev" }
      }
    },
    {
      "Effect": "Allow",
      "Action": [ "ssm:TerminateSession", "ssm:ResumeSession" ],
      "Resource": "arn:aws:ssm:*:*:session/${aws:userid}-*"
    }
  ]
}
```

## Cleanup

```bash
aws ec2 terminate-instances --instance-ids i-0abc123def4567890

aws iam remove-role-from-instance-profile \
  --instance-profile-name SSMInstanceProfile --role-name SSMInstanceRole
aws iam delete-instance-profile --instance-profile-name SSMInstanceProfile
aws iam detach-role-policy \
  --role-name SSMInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role --role-name SSMInstanceRole
```

## Conclusion

Session Manager is a more secure way to access instances: no
open port 22, no keys to manage and access controlled and
audited through IAM and CloudTrail. Port forwarding even replaces the
bastion host in many scenarios.

## References

-   [AWS Systems Manager Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
-   [Install the Session Manager plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)
