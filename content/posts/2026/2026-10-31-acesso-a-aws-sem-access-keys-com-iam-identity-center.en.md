---
title: "AWS access without access keys with IAM Identity Center"
slug: "2026-10-31-aws-access-without-access-keys-with-iam-identity-center"
description: "How to replace the IAM user with a long-lived access key with IAM Identity Center: Organizations, permission sets, MFA and AWS CLI login with aws sso login."
author: Jason
date: 2026-10-31
images: ["/images/capa-identity-center.png"]
tags: [ "AWS", "Security" ]
type: post
---

The goal of this article is to show how to replace AWS access through an **IAM
user with a long-lived access key** with **IAM Identity Center**. In the end, the AWS CLI
uses **temporary credentials**, login happens in the browser with
**MFA** and no key is left stored in `~/.aws/credentials`.

A few years ago I showed how to
[create and configure an IAM user]({{< relref "/posts/2024/2024-01-07-criando-e-configurando-um-usuario-iam" >}})
and then how to
[use an IAM role]({{< relref "/posts/2024/2024-03-30-criando-e-configurando-uma-role-iam" >}}),
a more secure way to access AWS. Identity Center is the next step: the IAM user
goes away and, with it, the access key.

## Prerequisites

-   An AWS account and access to the root user.
-   AWS CLI v2 installed.

In the examples the AWS account ID is `111122223333`. Replace it with your own values.

## Why move away from access keys

An IAM user's access key doesn't expire. If it leaks (a commit, a backup,
a lost laptop), whoever has the key has the user's permissions until
someone notices and deletes the key.

With Identity Center:

-   Login happens in the browser, with a password and MFA.
-   The AWS CLI receives temporary credentials, which expire on their own.
-   Permissions live in **permission sets**, which become IAM roles in the account.

## How it works

Identity Center has two kinds of instance:

-   **Organization instance**: gives access to the organization's AWS accounts.
    This is the one we need.
-   **Account instance**: only works for AWS applications, such as Amazon Q.
    It does **not** give access to the account's console or AWS CLI.

To have an organization instance, the account has to be part of **AWS
Organizations**. Even with a single account, it becomes the *management account* of
a one-account organization.

The AWS CLI login looks like this:

![Login flow with IAM Identity Center: the AWS CLI opens the portal in the browser, login happens with a password and MFA, the token is exchanged for temporary credentials and calls to AWS use the permission set's role](/images/identity-center.svg)

## Step 1: MFA on the root user

After the migration, root becomes the emergency access. Before starting, turn on
MFA for it: in the console, signed in as root, click the account name →
**Security credentials** → **Assign MFA device**.

To check with the AWS CLI:

```bash
aws iam get-account-summary --query 'SummaryMap.AccountMFAEnabled'
```

The result must be `1`.

## Step 2: create the organization

```bash
aws organizations create-organization --feature-set ALL

aws organizations enable-aws-service-access \
  --service-principal sso.amazonaws.com
```

The second command allows Identity Center to operate in the organization.

## Step 3: enable Identity Center

The organization instance is enabled in the console: **IAM Identity Center** →
**Enable**, choosing the option with AWS Organizations.

### If the account already has an account instance

That was my case. The account had an old, empty *account instance*, and with it
the permission set commands fail:

```text
An error occurred (ValidationException) when calling the ListPermissionSets
operation: This operation is not supported for account instances of IAM
Identity Center.
```

Before deleting it, check that it has no users or applications. In recent
versions of the AWS CLI:

```bash
aws sso-admin list-instances
aws sso-admin list-applications --instance-arn <instance-arn>
aws identitystore list-users --identity-store-id <identity-store-id>
```

If it's empty, delete the instance and enable it again in the console:

```bash
aws sso-admin delete-instance --instance-arn <instance-arn>
```

### MFA in Identity Center

Under **Settings** → **Authentication**, configure MFA to be required at every
sign-in.

## Step 4: user and permission sets

Get the instance ARN and the identity store ID:

```bash
aws sso-admin list-instances
```

Create your user:

```bash
aws identitystore create-user \
  --identity-store-id <identity-store-id> \
  --user-name my-user \
  --display-name "My Name" \
  --name GivenName=My,FamilyName=Name \
  --emails Value=my-email@example.com,Primary=true
```

Create an administration permission set with a 1-hour session. The duration uses the
ISO 8601 format (`PT1H`):

```bash
aws sso-admin create-permission-set \
  --instance-arn <instance-arn> \
  --name AdministratorAccess \
  --session-duration PT1H

aws sso-admin attach-managed-policy-to-permission-set \
  --instance-arn <instance-arn> \
  --permission-set-arn <permission-set-arn> \
  --managed-policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

And assign the permission set to the user in the account:

```bash
aws sso-admin create-account-assignment \
  --instance-arn <instance-arn> \
  --permission-set-arn <permission-set-arn> \
  --principal-type USER \
  --principal-id <user-id> \
  --target-type AWS_ACCOUNT \
  --target-id 111122223333
```

AWS creates a role in the account named
`AWSReservedSSO_AdministratorAccess_<suffix>`. Don't change this role directly in
IAM: it's managed by Identity Center.

It's also worth creating a read-only permission set (`ReadOnlyAccess`) for
day-to-day work and keeping the administration one for when you need to change something.

### User password

A user created through the API doesn't get the invitation on its own. In the console, under
**Users** → your user → **Reset password**, send the email to set the
password. On the first sign-in, register the MFA device.

## Step 5: AWS CLI with aws sso login

In `~/.aws/config`, create an `sso-session` and a profile for each permission
set:

```ini
[sso-session my-account]
sso_start_url = https://d-1234567890.awsapps.com/start
sso_region = us-east-1
sso_registration_scopes = sso:account:access

[profile admin]
sso_session = my-account
sso_account_id = 111122223333
sso_role_name = AdministratorAccess
region = us-east-1
```

`sso_start_url` is the *AWS access portal URL*, shown in the
Identity Center dashboard.

Sign in. The command opens the browser and shows a code: check that it's the
same one as in the terminal, sign in with password and MFA and authorize the access.

```bash
aws sso login --sso-session my-account
```

Test it:

```bash
aws sts get-caller-identity --profile admin
```

The `Arn` shows the permission set's role:

```text
arn:aws:sts::111122223333:assumed-role/AWSReservedSSO_AdministratorAccess_<suffix>/my-user
```

### What about Terraform?

Terraform uses the profiles in `~/.aws/config`. If you need the credentials in
environment variables, the AWS CLI exports the ones from the current session:

```bash
eval "$(aws configure export-credentials --profile admin --format env)"
```

## Step 6: retire the IAM user

After testing the login, delete the old IAM user. A user can only be
deleted with nothing attached to it, so the order is: access keys, MFA, console
password, policies and, finally, the user.

```bash
aws iam delete-access-key --user-name my-iam-user --access-key-id <access-key-id>
aws iam deactivate-mfa-device --user-name my-iam-user --serial-number <mfa-arn>
aws iam delete-virtual-mfa-device --serial-number <mfa-arn>
aws iam delete-login-profile --user-name my-iam-user
aws iam detach-user-policy --user-name my-iam-user --policy-arn <policy-arn>
aws iam delete-user-policy --user-name my-iam-user --policy-name <inline-policy-name>
aws iam delete-user --user-name my-iam-user
```

Finally, delete `~/.aws/credentials`.

## Conclusion

With Identity Center there's no longer a long-lived access key to leak. Access
goes through a password and MFA, credentials expire on their own and permissions are
organized in permission sets.

For pipelines, the same idea applies with OIDC: see
[GitHub Actions with OpenID Connect on AWS, without access keys]({{< relref "/posts/2026/2026-08-22-github-actions-com-oidc-na-aws-sem-access-keys" >}}).
