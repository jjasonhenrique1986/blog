---
title: "Configuring OpenID Connect between AWS and GitLab"
slug: "2023-07-22-configuring-openid-connect-between-aws-and-gitlab"
description: "How to configure OpenID Connect between AWS and GitLab to run pipelines with an IAM role, without long-lived access keys."
images: ["/images/capa-configurando-openid-connect-entre-aws-e-gitlab.png"]
author: Jason
date: 2023-07-22
lastmod: 2026-11-06
tags: [ "AWS", "CI/CD" ]
type: post
---

> **Update (November 2026):** GitLab removed the `CI_JOB_JWT_V2` variable
> used in the pipeline below in version 17.0. Today the token is requested
> with the `id_tokens` keyword; see the
> [Current version of the pipeline](#current-version-of-the-pipeline) section
> at the end of the post. The AWS side of the configuration stays the same.

The goal of this post is to show how to configure OpenID Connect
between your AWS account and your GitLab account to run a pipeline
using a role from your AWS account.

First of all, OpenID Connect (OIDC) lets you get short-lived tokens
to talk to your cloud provider. In this case
we'll use AWS, but the same concept can also be used
with Azure and GCP.

The big advantage of this integration between AWS and GitLab using OpenID
Connect is security: a short-lived token is generated, and you don't
need to create an **Access Key** and a **Secret Access
Key**. We'll use the AWS role concept, and this role will only
be usable from a specific GitLab repository.

## Prerequisites

To follow this tutorial you need the following
prerequisites:

-   An AWS account;
-   A [GitLab.com](https://about.gitlab.com/) (SaaS) account;

## OIDC configuration

In the first step we'll configure OIDC on AWS, more specifically
in IAM. Open the **IAM** service and the **Identity providers** option. You should
see the screen below:

![IAM Identity providers screen with no providers registered yet](/images/gitlab-2.webp)

Clicking **Add provider** shows the screen below, where
we'll start configuring OIDC:

![IAM Add an Identity provider screen with the SAML and OpenID Connect options](/images/gitlab-3.webp)

Select the OpenID Connect option. In the **Provider URL** and
**Audience** fields, enter the URL <https://gitlab.com>, as in the
image below:

![Registering the OpenID Connect provider with the URL and audience https://gitlab.com](/images/gitlab-4.webp)

Click **Get thumbprint**, which generates a
certificate thumbprint that validates the OIDC Provider's CA (in our case,
GitLab). Then click **Add provider** to finish the
OIDC configuration. A provider will be created, as in the
image below:

![gitlab.com provider of type OpenID Connect created in the IAM Identity providers list](/images/gitlab-5.webp)

## Creating the AWS role

In the second step we'll create a role on AWS by going to **IAM**,
then **Roles**, and then **Create Role**;
you should see a screen similar to the one below:

![IAM role creation Select trusted entity screen](/images/gitlab-6.webp)

Select the **Web Identity** option and select the **Identity
provider** and the **Audience** created in the previous step, as below:

![Creating the role with the Web identity type, the gitlab.com provider and the https://gitlab.com audience](/images/gitlab-7.webp)

Then we go to step 2, where we select this
role's permissions. In this case we'll use the
[AmazonS3ReadOnlyAccess](https://us-east-1.console.aws.amazon.com/iam/home#/policies/arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess) policy
just to list the S3 buckets of an AWS account.

Then we go to step 3, where we just enter the role name and
click **Create role**.

After the role is created, we'll edit its trust relationship
to make it less permissive. Below is this role's trust policy:

``` json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::aws_account_id:oidc-provider/gitlab.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "gitlab.com:sub": "project_path:user_gitlab/name_repository:ref_type:branch:ref:main"
                }
            }
        }
    ]
}
```

Where:

-   **aws_account_id** is your AWS account ID;
-   **user_gitlab** is the name of the user used on GitLab. A GitLab
    group can also be used here;
-   **name_repository** is the name of the repository created on GitLab;

## GitLab configuration

In the third step we'll create a GitLab repository and a
simple pipeline that assumes the role created in the previous step and lists the
AWS account's S3 buckets.

Create a GitLab repository and add the following environment
variable under Settings \>\> CI/CD \>\> Variables. It should
show a screen similar to the one below:

![ROLE_ARN variable registered as masked in the GitLab CI/CD variables](/images/gitlab-8.webp)

where:

-   **aws_account_id** is your AWS account ID;
-   **name_role** is the name of the role created on AWS;

This value can also be copied directly from the role created on AWS, in the
ARN field.

After that, create the **.gitlab-ci.yml** file with the content below:

``` yaml
image:
  name: "amazon/aws-cli:latest"
  entrypoint:
    - ""

stages:
  - "assume-role"

job_assume_role:
    stage: "assume-role"
    script: 
      - >
        STS=($(aws sts assume-role-with-web-identity
        --role-arn ${ROLE_ARN}
        --role-session-name "GitLabRunner-${CI_PROJECT_ID}-${CI_PIPELINE_ID}"
        --web-identity-token ${CI_JOB_JWT_V2}
        --duration-seconds 3600
        --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]'
        --output text))
      - export AWS_ACCESS_KEY_ID="${STS[0]}"
      - export AWS_SECRET_ACCESS_KEY="${STS[1]}"
      - export AWS_SESSION_TOKEN="${STS[2]}"
      - aws sts get-caller-identity
      - aws s3 ls
```

With this, a simple pipeline runs and lists the AWS account's S3
buckets.

## Current version of the pipeline

Since GitLab 17.0, the `CI_JOB_JWT_V2` variable no longer exists. The job
declares the token it needs with `id_tokens`, using as `aud` the same
audience registered in the AWS identity provider (`https://gitlab.com`):

``` yaml
job_assume_role:
    stage: "assume-role"
    id_tokens:
      GITLAB_OIDC_TOKEN:
        aud: https://gitlab.com
    script:
      - >
        STS=($(aws sts assume-role-with-web-identity
        --role-arn ${ROLE_ARN}
        --role-session-name "GitLabRunner-${CI_PROJECT_ID}-${CI_PIPELINE_ID}"
        --web-identity-token ${GITLAB_OIDC_TOKEN}
        --duration-seconds 3600
        --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]'
        --output text))
      - export AWS_ACCESS_KEY_ID="${STS[0]}"
      - export AWS_SECRET_ACCESS_KEY="${STS[1]}"
      - export AWS_SESSION_TOKEN="${STS[2]}"
      - aws sts get-caller-identity
      - aws s3 ls
```

More details in the
[GitLab documentation on OIDC with AWS](https://docs.gitlab.com/ci/cloud_services/aws/).

## Conclusion

The goal of this post was to show how to configure OpenID Connect
between AWS and GitLab. The same procedure can be done with GitHub,
following the [GitHub documentation on OIDC with
AWS](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services),
and I showed the step by step in
[GitHub Actions with OpenID Connect on AWS, without access keys]({{< relref "/posts/2026/2026-08-22-github-actions-com-oidc-na-aws-sem-access-keys" >}}).
