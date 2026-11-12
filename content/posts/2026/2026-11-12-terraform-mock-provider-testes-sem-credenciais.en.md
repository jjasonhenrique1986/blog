---
title: "Terraform mock_provider: testing modules without credentials"
slug: "2026-11-12-terraform-mock-provider-testing-without-credentials"
description: "How to use the Terraform 1.7 mock_provider to run terraform test without credentials and without creating resources: generated values, mock_resource, mock_data, override_during, override blocks and .tfmock.hcl files."
images: ["/images/capa-terraform-mock-provider.png"]
author: Jason
date: 2026-11-12
tags: [ "Terraform", "IaC", "AWS", "CI/CD" ]
type: post
---

In the post
[A Terraform module for the GitHub Actions OIDC role]({{< relref "/posts/2026/2026-11-11-modulo-terraform-para-role-oidc-do-github-actions" >}})
I used `mock_provider` to test the module without access to AWS, but only
with an empty block. In this post I'll show the feature in more detail: what
the mocked provider returns, where it breaks and how to control the values
with `mock_resource`, `mock_data`, `override_during` and the `override_*`
blocks.

All the examples were run with Terraform 1.17.1 and the AWS provider
6.73.0, with no credentials configured.

The test names and error messages are in Portuguese, since I wrote the
examples for the Portuguese version of this post. I kept them as they are
and explain each part in the text.

## The problem

`terraform test`, available since Terraform 1.6, really runs the module.
With the real provider, a `run` with `command = apply` creates the
resources in the account and destroys them at the end, and even a
`command = plan` needs credentials so the provider can query AWS. That makes
the tests slow, costs money and forces the CI of the modules repository to
have access to an account.

Terraform 1.7 brought `mock_provider`. It replaces the provider with a
mocked version, which knows the same schema as the real provider but calls
no API. The resources exist only in the test's in-memory state.

## The example module

I'll use a small module that creates a logs bucket with the account ID in
its name and a policy that allows writing to it:

```hcl
data "aws_caller_identity" "current" {}

resource "aws_s3_bucket" "logs" {
  bucket = "${var.name}-${data.aws_caller_identity.current.account_id}"
}

data "aws_iam_policy_document" "write_logs" {
  statement {
    actions   = ["s3:PutObject"]
    resources = ["${aws_s3_bucket.logs.arn}/*"]
  }
}

resource "aws_iam_policy" "write_logs" {
  name   = "${var.name}-write-logs"
  policy = data.aws_iam_policy_document.write_logs.json
}
```

The outputs are the bucket name and ARN and the policy ARN:

```hcl
output "bucket_name" {
  value = aws_s3_bucket.logs.bucket
}

output "bucket_arn" {
  value = aws_s3_bucket.logs.arn
}

output "policy_arn" {
  value = aws_iam_policy.write_logs.arn
}
```

The module has two data sources and two attributes computed by AWS (the
ARNs), and those are the points the mocked provider has to fill in.

## The first test

Tests live in `.tftest.hcl` files in the `tests/` directory. The simplest
one has an empty `mock_provider`:

```hcl
mock_provider "aws" {}

variables {
  name = "app"
}

run "plan_basico" {
  command = plan

  assert {
    condition     = aws_iam_policy.write_logs.name == "app-write-logs"
    error_message = "Nome da policy incorreto."
  }
}
```

To run it:

```bash
terraform init
terraform test
```

```text
tests/basico.tftest.hcl... in progress
  run "plan_basico"... pass
tests/basico.tftest.hcl... tearing down
tests/basico.tftest.hcl... pass

Success! 1 passed, 0 failed.
```

`terraform init` still downloads the AWS provider, because Terraform needs
its schema. What the test no longer needs are the credentials.

## What the mocked provider returns

The attributes you set in the configuration are kept. Computed attributes,
which the real provider would fill in after creating the resource, get
values generated according to their type:

| Type | Generated value |
|------|-----------------|
| String | 8 random alphanumeric characters |
| Number | `0` |
| Boolean | `false` |
| List, set and map | empty |

Running an `apply` with `terraform test -verbose` shows the test state.
This is an excerpt of the output:

```text
# data.aws_caller_identity.current:
data "aws_caller_identity" "current" {
    account_id = "o0a5b2kv"
    arn        = "3idg4vkf"
    id         = "79iqltu5"
    user_id    = "wz7ryi77"
}

# aws_s3_bucket.logs:
resource "aws_s3_bucket" "logs" {
    arn    = "htm7dneh"
    bucket = "app-o0a5b2kv"
    id     = "4agb9ujq"
    ...
}

Outputs:

bucket_arn = "htm7dneh"
bucket_name = "app-o0a5b2kv"
policy_arn = "5kkbdrot"
```

The account ID became `o0a5b2kv` and the bucket ARN became `htm7dneh`. The
mocked provider doesn't know the format of an ARN or of an account ID. It
only knows that the attribute is a string.

## Where the empty mock breaks

This behavior causes two problems in the example module.

The first one shows up on `apply`. The `aws_iam_policy_document` data source
isn't executed, so the `json` attribute also gets a random string.
`aws_iam_policy` validates the content of `policy` and rejects it:

```text
  run "apply_sem_defaults"... fail

Error: "policy" contains an invalid JSON policy: not a JSON object

  with aws_iam_policy.write_logs,
  on main.tf line 16, in resource "aws_iam_policy" "write_logs":
  16:   policy = data.aws_iam_policy_document.write_logs.json
```

In other words, the provider's validations still run, even when it is
mocked.

The second one shows up on `plan`. By default, values are generated only on
`apply`, so on `plan` a computed attribute is still unknown. An `assert` on
the bucket ARN fails with:

```text
Error: Unknown condition value

  on tests/plan_unknown.tftest.hcl line 11, in run "arn_no_plan":
  11:     condition     = startswith(aws_s3_bucket.logs.arn, "arn:aws:s3:::")
    ├────────────────
    │ aws_s3_bucket.logs.arn is a string

Condition expression could not be evaluated at this time.
```

The next sections fix both.

## Default values with mock_resource and mock_data

Inside `mock_provider`, the `mock_resource` and `mock_data` blocks set
default values for a resource type or a data source type. The attributes
you don't set are still generated:

```hcl
mock_provider "aws" {
  mock_data "aws_caller_identity" {
    defaults = {
      account_id = "111122223333"
    }
  }

  mock_data "aws_iam_policy_document" {
    defaults = {
      json = "{\"Version\":\"2012-10-17\",\"Statement\":[]}"
    }
  }

  mock_resource "aws_s3_bucket" {
    defaults = {
      arn = "arn:aws:s3:::app-111122223333"
    }
  }
}

variables {
  name = "app"
}

run "nome_e_arn_do_bucket" {
  command = apply

  assert {
    condition     = output.bucket_name == "app-111122223333"
    error_message = "O nome do bucket deveria terminar com o ID da conta."
  }

  assert {
    condition     = output.bucket_arn == "arn:aws:s3:::app-111122223333"
    error_message = "ARN do bucket incorreto."
  }

  assert {
    condition     = data.aws_iam_policy_document.write_logs.statement[0].resources == toset(["arn:aws:s3:::app-111122223333/*"])
    error_message = "A policy deveria liberar só os objetos do bucket de logs."
  }
}
```

Now the `apply` passes, because `json` is a valid policy, and the first two
`assert` blocks check the name and the ARN.

The third `assert` deserves attention. Since `aws_iam_policy_document` isn't
executed, the `json` in the test is the fixed value from `defaults`, not the
policy the module would build. Testing `json` says nothing about the module.
What you can test are the data source's arguments, such as the `resources`
list of the `statement`, which come from the configuration.

## Values on plan with override_during

To use the mocked values already on `plan`, there is the `override_during`
attribute, available from Terraform 1.11. The default is `apply`:

```hcl
mock_provider "aws" {
  override_during = plan

  mock_resource "aws_s3_bucket" {
    defaults = {
      arn = "arn:aws:s3:::app-111122223333"
    }
  }
}

variables {
  name = "app"
}

run "arn_no_plan" {
  command = plan

  assert {
    condition     = startswith(aws_s3_bucket.logs.arn, "arn:aws:s3:::")
    error_message = "ARN do bucket fora do formato."
  }
}
```

With that, the `assert` that failed with `Unknown condition value` passes.
`override_during` can also be set on a specific `mock_resource` or
`mock_data`, and in that case the innermost block wins.

If you use this attribute, the module's `required_version` has to be
`>= 1.11`, not `>= 1.7`.

## Overriding a specific resource

`mock_resource` and `mock_data` apply to **all** resources of a type. To
set the value of one specific resource, there are the `override_resource`,
`override_data` and `override_module` blocks. They take a `target` with the
resource address and can sit at the root of the file, applying to every
`run`, or inside a `run`:

```hcl
mock_provider "aws" {}

override_data {
  target = data.aws_iam_policy_document.write_logs
  values = {
    json = "{\"Version\":\"2012-10-17\",\"Statement\":[]}"
  }
}

variables {
  name = "app"
}

run "conta_de_producao" {
  command = apply

  override_data {
    target = data.aws_caller_identity.current
    values = {
      account_id = "999988887777"
    }
  }

  assert {
    condition     = output.bucket_name == "app-999988887777"
    error_message = "O nome do bucket deveria usar a conta de produção."
  }
}

run "conta_de_teste" {
  command = apply

  override_data {
    target = data.aws_caller_identity.current
    values = {
      account_id = "111122223333"
    }
  }

  assert {
    condition     = output.bucket_name == "app-111122223333"
    error_message = "O nome do bucket deveria usar a conta de teste."
  }
}
```

This way each `run` simulates a different account. When the same `target`
appears at the root of the file and inside a `run`, the one in the `run`
wins.

`override_module` does the same for a whole child module: instead of
`values`, it takes `outputs` with the outputs the module would return. The
`override_*` blocks also work with the real provider, which lets you mock
only part of the configuration.

## Reusing mocks in files

When several test files need the same values, the `mock_resource` and
`mock_data` blocks can live in a `.tfmock.hcl` file:

```hcl
# tests/mocks/aws.tfmock.hcl
mock_data "aws_caller_identity" {
  defaults = {
    account_id = "111122223333"
  }
}

mock_data "aws_iam_policy_document" {
  defaults = {
    json = "{\"Version\":\"2012-10-17\",\"Statement\":[]}"
  }
}

mock_resource "aws_s3_bucket" {
  defaults = {
    arn = "arn:aws:s3:::app-111122223333"
  }
}
```

`mock_provider` loads the directory with the `source` attribute:

```hcl
mock_provider "aws" {
  source = "./tests/mocks"
}

variables {
  name = "app"
}

run "usa_os_mocks_do_arquivo" {
  command = apply

  assert {
    condition     = output.bucket_name == "app-111122223333"
    error_message = "O nome do bucket deveria terminar com o ID da conta."
  }
}
```

The path is relative to the directory where `terraform test` runs, which
here is the module root. If the same type appears in the file and inside
`mock_provider`, the block in `mock_provider` wins.

Running all the test files from this post:

```text
tests/arquivo.tftest.hcl... in progress
  run "usa_os_mocks_do_arquivo"... pass
tests/arquivo.tftest.hcl... tearing down
tests/arquivo.tftest.hcl... pass
tests/defaults.tftest.hcl... in progress
  run "nome_e_arn_do_bucket"... pass
tests/defaults.tftest.hcl... tearing down
tests/defaults.tftest.hcl... pass
tests/override.tftest.hcl... in progress
  run "conta_de_producao"... pass
  run "conta_de_teste"... pass
tests/override.tftest.hcl... tearing down
tests/override.tftest.hcl... pass
tests/override_during.tftest.hcl... in progress
  run "arn_no_plan"... pass
tests/override_during.tftest.hcl... tearing down
tests/override_during.tftest.hcl... pass

Success! 5 passed, 0 failed.
```

## Mixing real and mocked providers

A mocked provider can live alongside the real one in the same test file,
as long as it has an `alias`, such as
`mock_provider "aws" { alias = "fake" }`. Each `run` picks which one to use
with the `providers` attribute.

One detail I noticed while testing: a real `provider "aws"` block in the
file is enough for Terraform to configure it, even if every `run` uses the
mocked one. Without credentials, the test fails with `No valid credential
sources found`. If the goal is to run without access to AWS, the test file
can only have the `mock_provider`.

## Limitations

- The mocked provider doesn't know the format of the values. ARNs, IDs and
  endpoints become random strings, unless you set a default.
- Data sources aren't executed. `aws_iam_policy_document` doesn't build the
  JSON and `aws_caller_identity` doesn't query the account.
- The test checks the **module's logic**: names, conditions, variable
  validations, what gets created or not. It doesn't check whether AWS
  accepts the configuration. An invalid bucket name or a missing permission
  only show up with the real provider.
- Mocks exist only inside `terraform test`. You can't use them in a regular
  `terraform plan`.

That's why the two kinds of test complement each other: the ones with
`mock_provider` can run on every pull request, and a test with the real
provider is left for when the change justifies creating real resources.

## Conclusion

`mock_provider` lets `terraform test` run in seconds, without credentials
and at no cost, which makes it possible to put the tests in the CI of any
modules repository. The empty block covers the simple cases. When the module
depends on computed values, `mock_resource`, `mock_data` and the
`override_*` blocks set those values, and `override_during = plan` makes
them available already on `plan`.

The CI workflow that runs these tests without credentials is in the post
[A Terraform module for the GitHub Actions OIDC role]({{< relref "/posts/2026/2026-11-11-modulo-terraform-para-role-oidc-do-github-actions" >}}).

## References

- [Mocks in terraform test](https://developer.hashicorp.com/terraform/language/tests/mocking)
- [Tests in Terraform](https://developer.hashicorp.com/terraform/language/tests)
- [Terraform 1.7 changelog](https://github.com/hashicorp/terraform/blob/v1.7/CHANGELOG.md)
- [Terraform 1.11 changelog](https://github.com/hashicorp/terraform/blob/v1.11/CHANGELOG.md)
