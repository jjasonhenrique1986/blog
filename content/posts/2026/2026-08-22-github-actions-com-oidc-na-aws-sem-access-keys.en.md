---
title: "GitHub Actions with OpenID Connect on AWS, without access keys"
slug: "2026-08-22-github-actions-with-openid-connect-on-aws-without-access-keys"
description: "How to set up OpenID Connect between GitHub Actions and AWS so workflows assume an IAM role with temporary credentials, without storing access keys."
images: ["/images/capa-github-actions-oidc.png"]
author: Jason
date: 2026-08-22
lastmod: 2026-11-03
tags: [ "AWS", "CI/CD", "Security" ]
type: post
---

The goal of this article is to show how to set up **OpenID Connect
(OIDC)** between **GitHub Actions** and your AWS account. With it, the
workflows assume an IAM role and receive **temporary credentials**, without
having to store `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` in the
repository secrets.

I already showed the same idea with GitLab in the article
[Configuring OpenID Connect between AWS and GitLab]({{< relref "/posts/2023/2023-07-22-configurando-openid-connect-entre-aws-e-gitlab" >}}).
Now let's do the same on GitHub.

## Prerequisites

-   An AWS account and AWS CLI v2 configured with permission to create
    IAM resources.
-   A GitHub repository.

In the examples the AWS account ID is `111122223333` and the repository is
`my-user/my-repo`. Replace them with your own values.

## How it works

1.  When a workflow job starts, GitHub can generate a signed **JWT
    token** with information about the run: repository,
    branch, environment, etc.
2.  The job sends this token to **AWS STS** asking to assume a
    role (`AssumeRoleWithWebIdentity`).
3.  AWS validates the token's signature with the **identity provider** we're
    going to register in IAM and checks the conditions of the role's **trust policy**.
4.  If everything is right, STS returns temporary credentials that
    are valid only for the duration of the job.

The big advantage is that **there are no long-lived credentials** to
leak or rotate.

## Registering GitHub as an identity provider

This step is done **once per AWS account**:

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

In older versions of the AWS CLI the `--thumbprint-list` parameter was
required. Today it's optional for GitHub, because AWS validates the
provider's certificate using its own library of trusted CAs.
The provider can also be created in the console under **IAM → Identity
providers → Add provider**, choosing **OpenID Connect**.

## Creating the role

Create a file called trust-github.json. The most important part is the
condition on the `sub` field, which defines **which repositories and branches**
can assume the role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::111122223333:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:my-user/my-repo:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

With this condition, only workflows running on the `main` branch of the
`my-user/my-repo` repository can assume the role. **Never
set the `sub` to `*`**: that would allow any GitHub repository,
from anyone, to assume your role.

Some useful `sub` formats:

| Scenario | `sub` value |
|---|---|
| Specific branch | `repo:my-user/my-repo:ref:refs/heads/main` |
| Any branch in the repository | `repo:my-user/my-repo:*` |
| Job with a GitHub environment | `repo:my-user/my-repo:environment:production` |
| Pull requests | `repo:my-user/my-repo:pull_request` |

> **Update (November 2026):** repositories created from July 15,
> 2026 on use an **immutable** `sub` format, which includes the numeric
> IDs of the owner and the repository, for example
> `repo:my-user@123456/my-repo@456789:ref:refs/heads/main`. Older
> repositories keep the format above, unless you turn on the new
> format. To see the prefix your repository uses:
>
> ```bash
> gh api repos/OWNER/REPO/actions/oidc/customization/sub --jq .sub_claim_prefix
> ```
>
> If the trust policy uses the wrong format, `AssumeRoleWithWebIdentity`
> fails. More details in the
> [GitHub documentation](https://docs.github.com/en/actions/reference/security/oidc#immutable-subject-claims).

Now create the role. For the example we'll grant read permission on S3:

```bash
aws iam create-role \
  --role-name github-actions-deploy \
  --assume-role-policy-document file://trust-github.json

aws iam attach-role-policy \
  --role-name github-actions-deploy \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

In production, create your own policy with only the permissions the
pipeline actually uses, following the principle of least privilege. If
you want a refresher on how roles work, see the article on
[how to create and configure an IAM role]({{< relref "/posts/2024/2024-03-30-criando-e-configurando-uma-role-iam" >}}).

## Creating the workflow

In the repository, create the file `.github/workflows/aws.yml`:

```yaml
name: AWS OIDC test

on:
  push:
    branches: [ main ]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  aws:
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111122223333:role/github-actions-deploy
          aws-region: us-east-1

      - name: Who am I?
        run: aws sts get-caller-identity

      - name: List buckets
        run: aws s3 ls
```

The `id-token: write` permission is what authorizes the job to request the
OIDC token from GitHub. **Without it, the credentials step fails.** The official
`aws-actions/configure-aws-credentials` action exchanges the token
for the credentials and exports the environment variables for the following
steps.

Push to `main` (or run it manually through `workflow_dispatch`) and
look at the job log. The **Who am I?** step should show an ARN like
`assumed-role/github-actions-deploy/...`.

## Common problems

-   **`Not authorized to perform sts:AssumeRoleWithWebIdentity`**:
    usually the trust policy's `sub` doesn't match the repository or
    branch that ran the workflow. Check upper and lower case in the
    user and repository names.
-   **`Credentials could not be loaded`** or an error asking for the token: the
    `id-token: write` permission is missing from the workflow or the job.
-   **Workflows from forks**: on pull requests from forks GitHub
    doesn't release the OIDC token, for security, so the credentials step
    fails. Don't try to work around it.

## Conclusion

With OIDC the pipeline accesses AWS with temporary credentials
restricted to a specific repository and branch. There are no access keys
stored in GitHub, nothing to rotate and, if the workflow is
compromised, the credentials expire on their own at the end of the job.

## References

-   [Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
-   [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)
-   [Create an OpenID Connect (OIDC) identity provider in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
