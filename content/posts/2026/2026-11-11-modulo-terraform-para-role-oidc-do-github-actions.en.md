---
title: "A Terraform module for the GitHub Actions OIDC role"
slug: "2026-11-11-terraform-module-for-the-github-actions-oidc-role"
description: "How to turn the GitHub Actions IAM role with OIDC into a reusable Terraform module, with validations that block wildcards and AdministratorAccess, tests with terraform test and versioning with tags."
images: ["/images/capa-modulo-terraform-oidc.png"]
author: Jason
date: 2026-11-11
tags: [ "Terraform", "IaC", "AWS", "CI/CD", "Security" ]
type: post
---

In the post
[GitHub Actions with OpenID Connect on AWS, without access keys]({{< relref "/posts/2026/2026-08-22-github-actions-com-oidc-na-aws-sem-access-keys" >}})
we created an IAM role that GitHub Actions assumes with OIDC, without storing
any long-lived credentials. It works well for one repository. The problem
shows up when each project gets its own role, which is what I
recommend: a compromised repository can only do what **its** role
allows.

With several roles, the trust policy starts getting copied and pasted, and that's where the
mistakes show up: a `sub` with a wildcard that accepts any branch, a CI role
with `AdministratorAccess` "just for testing". In this post I'll show the
Terraform module I use to create these roles, with validations that
reject those mistakes before `apply`, tests with `terraform test` and
versioning with tags.

The module's code, test names and error messages are in Portuguese, since I
wrote it for my own projects. I kept them as they are and explain each part in the text.

## What the module does

For each project, the module creates:

- A role with a trust policy tied to GitHub's OIDC provider, with `aud`
  equal to `sts.amazonaws.com` and an **explicit** list of `sub` values.
- Optionally, an inline policy with the project's permissions.
- Optionally, attached managed policies, as long as it isn't
  `AdministratorAccess`.

The OIDC provider `token.actions.githubusercontent.com` stays **outside** the
module, because it's created once per account, not once per project.

The layout is that of a modules repository, with one directory per module:

```text
terraform-modules/
└── modules/
    └── github-oidc-role/
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        ├── versions.tf
        ├── README.md
        └── tests/
            └── github_oidc_role.tftest.hcl
```

## The sub format

Before the code, a detail that changed in 2026. GitHub started using an
**immutable** format in the `sub` claim of OIDC tokens: repositories created
from July 15, 2026 on include the numeric IDs of the owner and the
repository. Instead of:

```text
repo:my-user/my-repo:ref:refs/heads/main
```

the `sub` becomes:

```text
repo:my-user@123456/my-repo@456789:ref:refs/heads/main
```

This prevents someone who creates an account or repository with a name you
deleted from inheriting access to your roles. The practical consequence is that you
need to know which format your repository uses before writing the trust
policy. GitHub shows the prefix through the API:

```bash
gh api repos/OWNER/REPO/actions/oidc/customization/sub --jq .sub_claim_prefix
```

That's why the module takes the `sub` in two parts: the repository
**prefix** (`github_sub_prefix`), which you copy from that command's output, and
the **suffixes** of the events that can assume the role (`subjects`), such as
`pull_request` or `ref:refs/heads/main`.

## Variables and validations

The module's protection lives in the variables' `validation` blocks. The prefix
must start with `repo:` and can't end with `:`:

```hcl
variable "github_sub_prefix" {
  type        = string
  description = "Prefixo do claim sub do repositório, por exemplo repo:meu-usuario@123456/meu-repo@456789"

  validation {
    condition     = startswith(var.github_sub_prefix, "repo:") && !endswith(var.github_sub_prefix, ":")
    error_message = "Use o prefixo no formato repo:OWNER/REPO (ou repo:OWNER@ID/REPO@ID), sem ':' no final."
  }
}
```

The suffixes must exist and can't contain wildcards:

```hcl
variable "subjects" {
  type        = list(string)
  description = "Sufixos do sub aceitos, por exemplo [\"pull_request\"] ou [\"ref:refs/heads/main\"]"

  validation {
    condition     = length(var.subjects) > 0 && alltrue([for s in var.subjects : !strcontains(s, "*")])
    error_message = "Informe ao menos um subject e não use curingas (*)."
  }
}
```

And the list of managed policies rejects `AdministratorAccess`:

```hcl
variable "managed_policy_arns" {
  type        = list(string)
  default     = []
  description = "ARNs de managed policies para anexar. Evite AdministratorAccess em roles de CI."

  validation {
    condition     = !anytrue([for arn in var.managed_policy_arns : endswith(arn, ":policy/AdministratorAccess")])
    error_message = "AdministratorAccess não é permitido em role de CI."
  }
}
```

The other variables are the role name, the OIDC provider ARN, the inline
policy (`inline_policy_json`), the maximum session duration, the description and the
tags.

## The trust policy

In `main.tf`, the module builds the full list of `sub` values by joining the prefix
with each suffix, and uses `StringEquals` instead of `StringLike`. With
`StringEquals`, a wildcard that slipped past the validation would be treated as the
literal `*` character, not as "anything":

```hcl
locals {
  oidc_host = "token.actions.githubusercontent.com"

  # Lista explícita de subs aceitos, sem curingas
  allowed_subs = [for s in var.subjects : "${var.github_sub_prefix}:${s}"]
}

resource "aws_iam_role" "this" {
  name                 = var.name
  description          = var.description
  max_session_duration = var.max_session_duration
  tags                 = var.tags

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Federated = var.oidc_provider_arn }
      Action    = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${local.oidc_host}:aud" = "sts.amazonaws.com"
          "${local.oidc_host}:sub" = local.allowed_subs
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "this" {
  count  = var.inline_policy_json == null ? 0 : 1
  name   = "${var.name}-inline"
  role   = aws_iam_role.this.id
  policy = var.inline_policy_json
}

resource "aws_iam_role_policy_attachment" "this" {
  for_each   = toset(var.managed_policy_arns)
  role       = aws_iam_role.this.name
  policy_arn = each.value
}
```

The outputs are `role_arn`, `role_name` and `allowed_subs`, which helps
check in the `plan` exactly which `sub` values the role will accept.

## Tests without AWS access

`terraform test`, available since Terraform 1.6, runs the module and
checks assertions. With `mock_provider`, which arrived in 1.7, the AWS
provider is simulated: the tests run without credentials and without creating anything. That's why
the module's `versions.tf` requires `required_version = ">= 1.7"`.

The test file defines default values for the variables and four
scenarios: the trust policy restricted to the repository, the inline policy, blocking a
wildcard in the subject and blocking admin:

```hcl
mock_provider "aws" {}

variables {
  name              = "gh-teste"
  oidc_provider_arn = "arn:aws:iam::111122223333:oidc-provider/token.actions.githubusercontent.com"
  github_sub_prefix = "repo:meu-usuario@123456/meu-repo@7891011"
  subjects          = ["pull_request", "ref:refs/heads/main"]
}

run "trust_policy_restrita_ao_repo" {
  command = plan

  assert {
    condition = jsondecode(aws_iam_role.this.assume_role_policy).Statement[0].Condition.StringEquals["token.actions.githubusercontent.com:sub"] == [
      "repo:meu-usuario@123456/meu-repo@7891011:pull_request",
      "repo:meu-usuario@123456/meu-repo@7891011:ref:refs/heads/main",
    ]
    error_message = "O sub da trust policy não corresponde ao repositório e eventos informados."
  }

  assert {
    condition     = jsondecode(aws_iam_role.this.assume_role_policy).Statement[0].Condition.StringEquals["token.actions.githubusercontent.com:aud"] == "sts.amazonaws.com"
    error_message = "O aud precisa ser sts.amazonaws.com."
  }

  assert {
    condition     = length(aws_iam_role_policy.this) == 0
    error_message = "Sem inline_policy_json não deve haver política inline."
  }
}

run "politica_inline" {
  command = plan

  variables {
    inline_policy_json = "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Action\":\"s3:ListBucket\",\"Resource\":\"*\"}]}"
  }

  assert {
    condition     = length(aws_iam_role_policy.this) == 1
    error_message = "A política inline deveria ser criada."
  }
}

run "bloqueia_curinga_no_subject" {
  command = plan

  variables {
    subjects = ["*"]
  }

  expect_failures = [var.subjects]
}

run "bloqueia_admin" {
  command = plan

  variables {
    managed_policy_arns = ["arn:aws:iam::aws:policy/AdministratorAccess"]
  }

  expect_failures = [var.managed_policy_arns]
}
```

The last two scenarios use `expect_failures`: the test **passes** when the
variable validation **fails**. That's how I make sure the protections
keep working if someone changes the module.

To run them:

```bash
cd modules/github-oidc-role
terraform init
terraform test
```

The output:

```text
tests/github_oidc_role.tftest.hcl... in progress
  run "trust_policy_restrita_ao_repo"... pass
  run "politica_inline"... pass
  run "bloqueia_curinga_no_subject"... pass
  run "bloqueia_admin"... pass
tests/github_oidc_role.tftest.hcl... tearing down
tests/github_oidc_role.tftest.hcl... pass

Success! 4 passed, 0 failed.
```

## CI for the modules repository

On GitHub Actions, a workflow runs `fmt`, `validate` and `test` on every pull
request, with a matrix that has one item per module. To add a new
module, just include it in the list:

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        module: [github-oidc-role]
    defaults:
      run:
        working-directory: modules/${{ matrix.module }}
    steps:
      - uses: actions/checkout@v7
      - uses: hashicorp/setup-terraform@v4
      - run: terraform fmt -check -recursive
      - run: terraform init -backend=false
      - run: terraform validate
      - run: terraform test
```

Since the tests use `mock_provider`, the workflow doesn't need any
AWS credentials.

## Versioning with tags

Each version of the module is a `vX.Y.Z` tag in the repository, and whoever uses the
module pins the version in `source` with `?ref=`. That way, a change in the module
only reaches a project when someone updates the tag on purpose, in a pull
request that shows the `plan` of the change.

## Using the module

The OIDC provider is created once in the account. You no longer need to provide the
certificate thumbprint: AWS validates GitHub's certificate on its
own.

```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url            = "https://token.actions.githubusercontent.com"
  client_id_list = ["sts.amazonaws.com"]
}
```

And each project becomes a `module` block. In the example, the repository
`my-repo` can assume the role only on pull requests, and only to read a bucket:

```hcl
data "aws_iam_policy_document" "my_repo" {
  statement {
    actions   = ["s3:GetObject"]
    resources = ["arn:aws:s3:::my-bucket/*"]
  }
}

module "gh_my_repo" {
  source = "git::https://github.com/my-user/terraform-modules.git//modules/github-oidc-role?ref=v0.1.0"

  name               = "gh-my-repo"
  description        = "GitHub Actions for my-user/my-repo (OIDC)"
  oidc_provider_arn  = aws_iam_openid_connect_provider.github.arn
  github_sub_prefix  = "repo:my-user@123456/my-repo@456789"
  subjects           = ["pull_request"]
  inline_policy_json = data.aws_iam_policy_document.my_repo.json
}

output "gh_my_repo_role_arn" {
  value = module.gh_my_repo.role_arn
}
```

After `apply`, I store the role ARN as the repository's `AWS_ROLE_ARN`
secret. In the workflow, the AWS action assumes the role and, with
`mask-aws-account-id`, hides the account ID in the logs:

```yaml
- uses: aws-actions/configure-aws-credentials@v6
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    aws-region: us-east-1
    mask-aws-account-id: true
```

If you try to pass `subjects = ["*"]` or attach `AdministratorAccess`, the
`plan` stops at the validation, before any change to the account.

## Conclusion

Turning the role into a module took the copy-and-paste work out of the trust policy,
which is where security mistakes usually creep in. The validations
reject wildcards and `AdministratorAccess`, the tests make sure those
protections stay in place, and the tag in `source` lets each project update the
module at its own pace.

The state for these resources lives in an S3 bucket with native locking, as I showed in
[Terraform: native state locking on S3, without DynamoDB]({{< relref "/posts/2026/2026-11-07-terraform-state-locking-nativo-no-s3-sem-dynamodb" >}}).

## References

- [OpenID Connect reference: immutable subject claims](https://docs.github.com/en/actions/reference/security/oidc#immutable-subject-claims)
- [Tests in Terraform](https://developer.hashicorp.com/terraform/language/tests)
- [Mocks in terraform test](https://developer.hashicorp.com/terraform/language/tests/mocking)
- [Input variable validation in Terraform](https://developer.hashicorp.com/terraform/language/expressions/custom-conditions#input-variable-validation)
- [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)
