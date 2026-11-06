---
title: "Terraform: native state locking on S3, without DynamoDB"
slug: "2026-11-07-terraform-native-state-locking-on-s3-without-dynamodb"
description: "How native state locking works in Terraform's S3 backend with use_lockfile, the IAM permissions it needs and how to migrate from DynamoDB locking, which is deprecated."
images: ["/images/capa-terraform-lock-s3.png"]
author: Jason
date: 2026-11-07
tags: [ "Terraform", "IaC", "AWS" ]
type: post
---

In the post about
[Remote Terraform State with an S3 bucket]({{< relref "/posts/2024/2024-08-11-configuracao-de-remote-terraform-state-com-bucket-s3-na-aws" >}})
the state lived in a bucket, but without any concurrency control. If two
people, or two pipelines, run `terraform apply` at the same time on the same
state, both runs read the same version, each one writes its own, and the
last one overwrites the first. That's how resources get lost from the state.

**State locking** fixes this: before any operation that may
change the state, Terraform takes a lock, and a second run fails (or
waits) until the lock is released.

For a long time, the S3 backend could only lock with a **DynamoDB** table.
Since Terraform 1.10 it can lock directly in the bucket, and in
1.11 this option became stable and DynamoDB locking was
deprecated. In this post we'll see how native locking works, how to
enable it and how to migrate a configuration that still uses DynamoDB.

## How native locking works

With `use_lockfile = true`, Terraform creates a file with the same name as the
state plus the `.tflock` suffix. For a state at `network/terraform.tfstate`, the
lock lives at `network/terraform.tfstate.tflock`.

The important detail is **how** this file is created. Terraform does a
`PutObject` with the `If-None-Match: *` header, which is an S3 conditional
write: the object is only written if it doesn't exist yet. If another
run has already created the `.tflock`, S3 rejects the write and Terraform
knows the state is locked. Since the check and the write happen in a
single S3 operation, there's no race window like there would be if
Terraform first checked and then wrote.

The content of `.tflock` is a JSON with the lock information: an ID, who
took the lock, the operation and the date. To release it, Terraform reads the file,
checks that the ID is the one it created and deletes the object.

You can see this in Terraform's
[S3 backend code](https://github.com/hashicorp/terraform/blob/main/internal/backend/remote-state/s3/client.go),
in the `lockWithFile` and `unlockWithFile` functions.

## Enabling the lock

Just add `use_lockfile = true` to the backend block. Since the option is only
stable from 1.11 on, it's worth requiring that version:

```hcl
terraform {
  required_version = ">= 1.11"

  backend "s3" {
    bucket       = "terraform-blog-jason"
    key          = "network/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true
  }
}
```

After changing the backend, run `init` again so Terraform applies
the new configuration:

```bash
terraform init -reconfigure
```

From then on, every `plan`, `apply` and `destroy` takes the lock before starting
and releases it at the end.

## IAM permissions

On top of the permissions already needed to read and write the state, the
identity that runs Terraform needs `s3:GetObject`, `s3:PutObject`
and `s3:DeleteObject` on the `.tflock` file. Note that `DeleteObject` is
only needed for the lock: Terraform never deletes the state.

A least-privilege policy for the example state looks like this:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::terraform-blog-jason"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::terraform-blog-jason/network/terraform.tfstate"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::terraform-blog-jason/network/terraform.tfstate.tflock"
    }
  ]
}
```

If the pipeline runs on GitHub Actions, this policy goes on the role that the
workflow assumes via OIDC, as I showed in
[GitHub Actions with OpenID Connect on AWS, without access keys]({{< relref "/posts/2026/2026-08-22-github-actions-com-oidc-na-aws-sem-access-keys" >}}).

## Migrating from DynamoDB

If your backend still has `dynamodb_table`, the migration can be done in
two steps, with no window where the state is unlocked.

**Step 1: turn on both locks at the same time.** The backend accepts
`use_lockfile` and `dynamodb_table` together. In this mode Terraform takes
both locks, and the operation only proceeds if it gets both:

```hcl
terraform {
  required_version = ">= 1.11"

  backend "s3" {
    bucket         = "terraform-blog-jason"
    key            = "network/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-lock"
    use_lockfile   = true
  }
}
```

On `terraform init`, Terraform already warns that the parameter's days are
numbered:

```text
Warning: Deprecated Parameter

  on main.tf line 8, in terraform:
   8:     dynamodb_table = "terraform-lock"

The parameter "dynamodb_table" is deprecated. Use parameter "use_lockfile"
instead.
```

This is useful when several people or pipelines use the same state and not
all of them have been updated yet: whoever has the old configuration keeps
respecting the DynamoDB lock, and whoever has the new one respects both.

**Step 2: remove DynamoDB.** Once everyone is on the
new configuration, remove `dynamodb_table`, run `terraform init
-reconfigure` and, after confirming nothing else uses the table, delete it.

## When the state is locked

If another run holds the lock, Terraform stops with the error
`Error acquiring the state lock`, followed by the lock information (ID,
path, operation, who and when) and this explanation:

```text
Terraform acquires a state lock to protect the state from being written
by multiple users at the same time. Please resolve the issue above and try
again. For most commands, you can disable locking with the "-lock=false"
flag, but this is not recommended.
```

A few options in this situation:

- **Wait for the lock to be released.** With `-lock-timeout`, Terraform retries
  for the given time instead of failing right away, which helps
  with pipelines that run in sequence:

  ```bash
  terraform apply -lock-timeout=5m
  ```

- **Release a forgotten lock.** If a run was interrupted halfway
  (the runner died, someone closed the terminal), the `.tflock` may stay in the
  bucket. Before releasing it, make sure nobody is really running
  Terraform, and use the ID shown in the error:

  ```bash
  terraform force-unlock LOCK-ID
  ```

- **Don't use `-lock=false`** day to day. It turns off the protection and takes you
  right back to the problem at the start of this post.

## Bucket versioning

The lock prevents concurrent writes, but it doesn't protect against a state that is deleted
or corrupted by mistake. For that, the backend documentation recommends
enabling **versioning** on the bucket: with it, you can roll back to a
previous version of the state.

## Conclusion

With `use_lockfile = true`, the state lock depends only on the S3
bucket that already held the state: one less table to create, pay for and grant
permissions to. If you still use DynamoDB, migrating with both locks turned on
at the same time lets you switch without rushing.

## References

- [S3 backend in the Terraform documentation](https://developer.hashicorp.com/terraform/language/backend/s3)
- [terraform force-unlock command](https://developer.hashicorp.com/terraform/cli/commands/force-unlock)
- [Conditional writes in Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html)
