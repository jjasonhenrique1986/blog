---
title: "Infrastructure PR review with AI, part 1: architecture and access to Bedrock"
slug: "2026-11-01-infrastructure-pr-review-with-ai-part-1"
description: "First part of the series on reviewing Terraform and Kubernetes pull requests with AI on Amazon Bedrock (Claude or gpt-oss): the project architecture, model access and the IAM role with OIDC, without access keys."
images: ["/images/capa-revisao-pr-ia.png"]
author: Jason
date: 2026-11-01
tags: [ "AWS", "CI/CD", "AI", "Terraform", "Kubernetes" ]
type: post
draft: false
---

Reviewing infrastructure pull requests is tiring. A `cidr_blocks = ["0.0.0.0/0"]`
or a container without a memory limit slips through easily in the middle of a big diff.

In this series I'll build an **AI pull request reviewer**: on every pull request, a
GitHub Actions workflow takes the Terraform and Kubernetes diff, runs it through
four scanners (**checkov**, **Trivy**, **kube-linter** and **kubeconform**) and asks a model on **Amazon Bedrock** to explain the risks in a
comment. The default is **Claude**, but the project also accepts OpenAI's **gpt-oss**,
and that's the one I used for the tests in this series (I explain why below).

The series has three parts:

1. **Architecture and access to Bedrock** (this post)
2. [The Python script that talks to the model]({{< relref "/posts/2026/2026-11-02-revisao-de-pr-de-infraestrutura-com-ia-parte-2" >}})
3. [The GitHub action and the security precautions]({{< relref "/posts/2026/2026-11-03-revisao-de-pr-de-infraestrutura-com-ia-parte-3" >}})

Everything became a **reusable action**: to use the reviewer in a repository, just
create a workflow that calls the action. The full code is at
[github-actions/iac-ai-review](https://github.com/jjasonhenrique1986/github-actions/tree/main/iac-ai-review).

## How it works

1. A PR changes `.tf` or `.yaml` files.
2. The workflow runs the scanners, which do the **deterministic** checks.
3. The workflow assumes an IAM role via **OIDC** and calls the model on Bedrock with the
   diff and the scanner output.
4. The model returns a review in JSON (summary, overall risk and a list of findings).
5. The workflow posts the review as a comment on the PR.

The division of labor matters: **the scanners find, the AI explains**. The
scanners don't hallucinate, but their output is long and generic. The model prioritizes
what matters in the changed lines and explains it in plain language. The AI **doesn't approve or
block** the PR: it's a second reading, not the human review.

## Why Bedrock

You could call the Anthropic API directly, with an API key stored in secrets.
With Bedrock:

- The diff **never leaves your AWS account**, which makes the conversation with security and
  compliance at your company easier.
- Authentication uses **IAM**. With GitHub's OIDC, there's no key
  stored in the repository.
- Billing comes on the AWS invoice.
- With the same role and the same account, you can switch models (and vendors)
  by changing a single variable, `MODEL_ID`.

## Model access

The reviewer accepts a closed list of models:

| `MODEL_ID` | Model | API used |
|---|---|---|
| `anthropic.claude-opus-5-5` (default) | Claude Opus 5.5 | Anthropic SDK (`bedrock-mantle`) |
| `anthropic.claude-opus-4-8` | Claude Opus 4.8 | Anthropic SDK (`bedrock-mantle`) |
| `openai.gpt-oss-120b-1:0` | gpt-oss-120b | Bedrock Converse API |
| `openai.gpt-oss-20b-1:0` | gpt-oss-20b | Bedrock Converse API |

Having the model in the region's catalog isn't enough. In my account, the
Anthropic models went through three steps:

1. **Model agreement**: fill in the use case form and accept the model
   offer in the Bedrock console.
2. **IAM permission** on the role that calls the model (next section).
3. **Quota**: even with the agreement accepted, the account's tokens-per-minute quotas
   were **zero**, and every call came back with access denied. A zero quota
   is solved with an increase request in **Service Quotas**.

So you don't have to guess which step is failing, the script has a command that
lists the supported models, the agreement status and, with `--test`, makes a
minimal call to each one. This is the real output in my account (the script
prints in Portuguese) while the Claude quota increase is pending:

```text
$ python reviewer/review.py --list-models --test
| ID | Provedor | Nome | Acordo | Chamada |
|---|---|---|---|---|
| `anthropic.claude-opus-5-5` | anthropic | Claude Opus 5.5 | AVAILABLE | acesso negado |
| `anthropic.claude-opus-4-8` | anthropic | Claude Opus 4.8 | AVAILABLE | acesso negado |
| `openai.gpt-oss-120b-1:0` | openai | gpt-oss-120b | AVAILABLE | ok |
| `openai.gpt-oss-20b-1:0` | openai | gpt-oss-20b | AVAILABLE | ok |

Padrão: `anthropic.claude-opus-5-5`. Escolha outro com a variável `MODEL_ID`.
```

The agreement is `AVAILABLE` for all of them, but only gpt-oss answers ("acesso negado"
means access denied). That's why the tests in this series use `openai.gpt-oss-120b-1:0`.

## Creating the role

If you haven't registered GitHub as an identity provider in IAM yet, follow the
post [GitHub Actions with OpenID Connect on AWS, without access keys]({{< relref "/posts/2026/2026-08-22-github-actions-com-oidc-na-aws-sem-access-keys" >}}).
It's the foundation for this part.

### Trust policy

Since the workflow runs on pull requests, the `sub` is
`repo:my-user/my-repo:pull_request`:

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
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:my-user/my-repo:pull_request"
        }
      }
    }
  ]
}
```

Before creating the role, check the `sub` format in your repository. If it
uses the *immutable subject*, the `sub` includes the numeric IDs of the owner and the
repository, like `repo:my-user@123456/my-repo@7891011:pull_request`. With the
trust policy above, the workflow fails with `Not authorized to perform
sts:AssumeRoleWithWebIdentity`. To see the prefix GitHub uses:

```bash
gh api repos/my-user/my-repo/actions/oidc/customization/sub --jq .sub_claim_prefix
```

In the trust policy, use that prefix followed by `:pull_request`. The advantage of the
immutable format is that it doesn't accept a repository that was deleted and recreated with the
same name.

### Permission to call the models

The current Claude integration on Bedrock uses the `bedrock-mantle` endpoint, and the
IAM inference action is `bedrock-mantle:CreateInference`. It's different from the
`bedrock:InvokeModel` of the old integration. The resource is the Bedrock **project**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ClaudeNoBedrock",
      "Effect": "Allow",
      "Action": "bedrock-mantle:CreateInference",
      "Resource": "arn:aws:bedrock-mantle:*:111122223333:project/*"
    }
  ]
}
```

To restrict the role to specific models, there's the condition key
`bedrock-mantle:Model`.

gpt-oss uses the Bedrock Converse API, which is authorized by the
`bedrock:InvokeModel` action on the model ARN. If you'll only use one of the two
vendors, keep only its statement:

```json
{
  "Sid": "GptOss",
  "Effect": "Allow",
  "Action": "bedrock:InvokeModel",
  "Resource": [
    "arn:aws:bedrock:*::foundation-model/openai.gpt-oss-120b-1:0",
    "arn:aws:bedrock:*::foundation-model/openai.gpt-oss-20b-1:0"
  ]
}
```

Creating the role:

```bash
aws iam create-role \
  --role-name github-iac-ai-reviewer \
  --assume-role-policy-document file://iam/trust-policy.json

aws iam put-role-policy \
  --role-name github-iac-ai-reviewer \
  --policy-name bedrock-claude \
  --policy-document file://iam/bedrock-policy.json
```

Notice that this role **only** calls the model. It doesn't need any
permission on the infrastructure being reviewed.

To run the reviewer on your machine, don't create an access key: use an
IAM Identity Center profile with the same permissions, as I showed in the post
[AWS access without access keys with IAM Identity Center]({{< relref "/posts/2026/2026-10-31-acesso-a-aws-sem-access-keys-com-iam-identity-center" >}}).

## Next step

With the role ready, in **[part 2]({{< relref "/posts/2026/2026-11-02-revisao-de-pr-de-infraestrutura-com-ia-parte-2" >}})** we'll look at the Python script: how to build the
prompt, get the response in a guaranteed format and what to do when the
model refuses.

## References

- [Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock)
- [OpenAI models in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters-openai.html)
- [Actions, resources, and condition keys for Amazon Bedrock Powered by AWS Mantle](https://docs.aws.amazon.com/service-authorization/latest/reference/list_bedrock-mantle.html)
- [Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
