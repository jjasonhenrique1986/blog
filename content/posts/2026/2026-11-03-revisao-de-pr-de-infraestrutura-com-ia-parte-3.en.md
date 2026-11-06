---
title: "Infrastructure PR review with AI, part 3: the action and security"
slug: "2026-11-03-infrastructure-pr-review-with-ai-part-3"
description: "Last part of the series: the reusable GitHub action that runs checkov, Trivy, kube-linter and kubeconform, calls the AI on Amazon Bedrock and comments on the pull request, the result on a real PR and the precautions around forks, prompt injection and cost."
images: ["/images/capa-revisao-pr-ia.png"]
author: Jason
date: 2026-11-03
tags: [ "AWS", "CI/CD", "AI", "Security" ]
type: post
draft: false
---

In parts [1]({{< relref "/posts/2026/2026-11-01-revisao-de-pr-de-infraestrutura-com-ia-parte-1" >}})
and [2]({{< relref "/posts/2026/2026-11-02-revisao-de-pr-de-infraestrutura-com-ia-parte-2" >}})
we created the IAM role and the script. Now let's package everything into a GitHub action
and run it on every pull request.

## The action

Instead of copying the script and the workflow steps into every repository, the
reviewer became a **composite action** in the
[github-actions](https://github.com/jjasonhenrique1986/github-actions) repository. The
Python code, the tests and the examples live in the same folder as the action, so the action's
version (`@v1`) defines exactly which code runs.

In the repository being reviewed, the workflow looks like this
(`.github/workflows/ai-review.yml`):

```yaml
name: IaC review with AI

on:
  pull_request:
    paths: ["**.tf", "**.tfvars", "**.yaml", "**.yml"]

permissions:
  id-token: write       # OIDC token to assume the role on AWS
  contents: read
  pull-requests: write  # comment on the PR

concurrency:
  group: ai-review-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  review:
    # PRs from forks don't get the OIDC token
    if: github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - uses: jjasonhenrique1986/github-actions/iac-ai-review@v1
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          model-id: ${{ vars.MODEL_ID }}
```

Important points:

- **`fetch-depth: 0`**: without the full history, `git diff` against the base
  branch doesn't work. The action checks this and fails with a clear message.
- **`concurrency`**: if you push several times in a row, only the last run
  continues. That saves calls to the model.
- **`MODEL_ID`**: a repository variable that picks the model. Without it, the
  default is `anthropic.claude-opus-5-5`. In my tests it's set to
  `openai.gpt-oss-120b-1:0`, because the Claude quota in my account is still
  zero (see [part 1]({{< relref "/posts/2026/2026-11-01-revisao-de-pr-de-infraestrutura-com-ia-parte-1" >}})).
- **No access keys**: the only secret is `AWS_ROLE_ARN`, and it isn't a
  credential. It's a secret only so the account ID doesn't show up in the repository
  or in the logs. The action also masks the account ID.
- **One role per repository**: each repository that uses the action has its
  own role, tied to its `sub`, as in part 1.
- **Pinned by SHA**: `@v1` is a tag that moves with every new version of the
  action, so the code that runs can change without you touching the workflow. To
  lock exactly what was reviewed, use the commit SHA and keep the version
  in a comment:
  `uses: jjasonhenrique1986/github-actions/iac-ai-review@a49155e8d21ddaf1b68f7768733a3799c78c9369 # v1.1.0`.
  Dependabot understands this format and opens a PR when a new version comes out.

### Inside the action

`action.yml` chains the same steps that used to live in the workflow. Two
details are worth highlighting. The first: before touching AWS, the action looks for
infrastructure changes with the same patterns as the script (the step names in
the action are in Portuguese):

```yaml
- name: Detectar mudanças de infraestrutura
  id: changes
  shell: bash
  env:
    BASE: ${{ steps.context.outputs.base }}
  run: |
    files="$(git diff --name-only "$BASE...HEAD" -- '*.tf' '*.tfvars' '*.yaml' '*.yml')"
    if [ -n "$files" ]; then
      echo "has-changes=true" >> "$GITHUB_OUTPUT"
    else
      echo "has-changes=false" >> "$GITHUB_OUTPUT"
    fi
```

The scanners and the credentials only run when `has-changes == 'true'`. On a PR that only
touches documentation, the action doesn't assume the role, doesn't call the model and only
comments that there's nothing to review.

The second: the script runs from the action's own folder (`github.action_path`), not
from the reviewed repository. That way the reviewer's code doesn't end up in the diff or in the
scanners of whoever uses the action:

```yaml
- name: Revisar com IA
  id: review
  shell: bash
  env:
    DIR: ${{ github.action_path }}
    BASE: ${{ steps.context.outputs.base }}
    MODEL_ID: ${{ inputs.model-id }}
    AWS_REGION: ${{ inputs.aws-region }}
    STATUS_checkov: ${{ steps.checkov.outputs.status }}
    STATUS_trivy: ${{ steps.trivy.outputs.status }}
    STATUS_kube_linter: ${{ steps.kube-linter.outputs.status }}
    STATUS_kubeconform: ${{ steps.kubeconform.outputs.status }}
  run: |
    out="$RUNNER_TEMP/review.md"
    args=()
    for name in checkov trivy kube-linter kubeconform; do
      var="STATUS_${name//-/_}"
      case "${!var:-}" in
        ok) args+=(--scan "$name=$RUNNER_TEMP/scan-$name.json") ;;
        failed) args+=(--scan-failed "$name") ;;
      esac
    done
    "$RUNNER_TEMP/venv-reviewer/bin/python" "$DIR/reviewer/review.py" \
      --base "$BASE" "${args[@]}" --output "$out"
    echo "review-file=$out" >> "$GITHUB_OUTPUT"
```

Each scanner runs in its own step and writes a JSON report. The script
gets one `--scan` per scanner that worked and one `--scan-failed` per scanner
that failed, and the comment says which ones didn't run. A scanner that is down doesn't
take the review down.

The other steps are `setup-python`, a venv for the reviewer, another one for
checkov (installed together, checkov downgraded `boto3` in one of my tests), the
download of Trivy, kube-linter and kubeconform, which only run if the binary's SHA-256
matches the one pinned in the action, `configure-aws-credentials` and
`sticky-pull-request-comment`, which updates the same comment on every push instead
of flooding the PR with comments. No scanner breaks the job for finding
problems: the idea is to pass the findings on to the model.

The action is also tested in its own repository: `actionlint`, `pytest` (with
real output from each scanner), a job that checks that a PR without infrastructure
doesn't get credentials, one that checks that an unknown scanner is rejected and one
that reviews a Terraform file with open SSH and a manifest with `cluster-admin`,
calling the model for real with all four scanners.

## Security

### PRs from forks

For security, GitHub **doesn't hand out the OIDC token** to workflows triggered
by PRs from forks. The `if` condition skips the job in those cases instead of letting it
fail.

Don't try to work around this by swapping `pull_request` for `pull_request_target`.
That event runs with the permissions of the base repository, and checking out the
PR's code in it is a well-known way to leak credentials.

### Prompt injection

Whoever opens the PR controls the content of the diff, and the diff goes to the model. The
project's defenses come in layers:

1. The diff goes inside `<diff>` tags, and the prompt says to ignore instructions that
   show up there.
2. The role can **only** call the model. It has no access to the infrastructure.
3. The GitHub token in the job only **reads** the code and **comments** on the PR.
4. The AI **doesn't approve** the PR. In the worst case, a successful attack results in a
   wrong comment, which the human review should catch.

What really protects you are layers 2 to 4: limiting what the job **can
do**, instead of trusting that the model will always follow the prompt.

### Data

The diff and the scanner reports are sent to Bedrock, inside your AWS account.
Even so, check your company's policy before using it on repositories
with sensitive data.

## Cost

The cost depends on the size of the diff and on the model. A few ways to keep it under control:

- The workflow's `paths` filter avoids running on PRs that don't touch
  infrastructure.
- `concurrency` cancels old runs.
- `MAX_DIFF_CHARS` avoids sending huge PRs.
- The `MODEL_ID` variable lets you switch models, for example from
  `openai.gpt-oss-120b-1:0` to `openai.gpt-oss-20b-1:0`, which is smaller, and
  compare the quality of the reviews.

Track the spend in **AWS Cost Explorer** by filtering on the Bedrock service.

## Testing

The action's folder has **intentionally insecure** examples in `examples/`. Don't
apply any of them to a real account or cluster:

| File | Problems |
|---|---|
| `terraform/main.tf` | Security Group with SSH open to `0.0.0.0/0` |
| `terraform/rds/main.tf` | public RDS, no encryption, password in the code and no backups |
| `terraform/iam/main.tf` | IAM user with an access key and an `Action: "*"` policy on `Resource: "*"` |
| `terraform/s3-publico/main.tf` | public access block turned off and read access for `Principal: "*"` |
| `terraform/eks/main.tf` | API endpoint open to `0.0.0.0/0`, no logs and no KMS for Secrets |
| `k8s/deployment.yaml` | `latest` image, password in an environment variable, no resource limits |
| `k8s/pod-privilegiado.yaml` | privileged DaemonSet running as root, with `hostNetwork`, `hostPID` and the node's `/` mounted |
| `k8s/rbac-cluster-admin.yaml` | application ServiceAccount with `cluster-admin` |
| `k8s/banco-exposto.yaml` | PostgreSQL on a public `LoadBalancer` and a password in a ConfigMap |
| `k8s/ingress-sem-tls.yaml` | admin panel published over HTTP, with no TLS and no authentication |

### The result on a real PR

The new examples went into
[PR #4](https://github.com/jjasonhenrique1986/iac-ai-reviewer/pull/4) of the
project's original repository, before it became the action. The workflow ran with `openai.gpt-oss-120b-1:0` and posted a
comment with an overall risk of **high** and 12 findings. An excerpt, translated
from the Portuguese comment:

> **Overall risk:** 🔴 high
>
> ### 🔴 ClusterRoleBinding granting cluster-admin to a regular ServiceAccount
> `examples/k8s/rbac-cluster-admin.yaml` · severity **high**
>
> The `app` ServiceAccount gets the `cluster-admin` `ClusterRole`, granting
> administrative permissions across the whole cluster, violating the principle of least
> privilege.
>
> **Suggestion:** Create specific roles with the required permissions and bind
> them only to the ServiceAccount. Remove the `cluster-admin` binding.
>
> ### 🔴 HTTP Ingress with no TLS or authentication
> `examples/k8s/ingress-sem-tls.yaml` · severity **high**
>
> The Ingress exposes the admin over plain HTTP (`host: admin.exemplo.com.br`) with no TLS
> and no authentication layer, making interception attacks and unauthorized
> access easier.

### What the AI adds to checkov

On the same examples, checkov failed 49 Terraform checks and 59
Kubernetes checks. Even so, **none** of them points out the two findings above: the
ServiceAccount with `cluster-admin` and the Ingress without TLS. The AI caught both on the
PR and in the three local runs I did with the same diff.

This is where the division of labor from part 1 shows up: checkov is exhaustive
within the rules it knows, and the AI reads the intent of the resource (an `admin`
panel on the internet, a regular application with cluster administrator power).

### More scanners: Trivy, kube-linter and kubeconform

checkov doesn't have to be the only one. I ran three more tools on the same
examples to see what each one adds:

| Risk | checkov | Trivy | kube-linter | AI |
|---|---|---|---|---|
| Open SSH, public S3, public RDS, open EKS | ✅ | ✅ | — | ✅ |
| IAM policy `*` on `*` | ✅ | ❌ | — | ✅ |
| Privileged container, `hostNetwork`, `hostPID` | ✅ | ✅ | ✅ | ✅ |
| ServiceAccount with `cluster-admin` | ❌ | ✅ | ✅ | ✅ |
| Password in a ConfigMap | ❌ | ✅ | ❌ | ✅ |
| PostgreSQL on a public `LoadBalancer` | ❌ | ❌ | ✅ | ✅ |
| Ingress without TLS | ❌ | ❌ | ❌ | ✅ |

- **Trivy** (`trivy config`) complements checkov: it catches `cluster-admin` and the
  password in the ConfigMap, but misses the IAM policy with `*`, which checkov catches well.
- **kube-linter** only adds value with two rules beyond the defaults,
  `cluster-admin-role-binding` and `exposed-services`. With all rules turned on,
  it gets far too noisy (key order, `owner` label, a minimum of three replicas).
- **kubeconform** doesn't look for risks: it validates the manifests against the Kubernetes
  API schema and catches nonexistent fields, wrong types or removed `apiVersion`s,
  which the others don't catch.
- No scanner flagged the **Ingress without TLS**. Only the AI did.

That's why the action runs all four by default. You can choose with the
`scanners` input, for example `scanners: checkov,trivy` in a Terraform-only
repository. A practical detail: Trivy's text output on the examples went over
60 KB, and checkov's over 30 KB. Sending that raw to the model burns tokens and
gets close to the diff limit, so the script summarizes each report into one line
per finding (`file:line [SEVERITY] ID title`). On the examples, all four
reports together came to 16 thousand characters.

### The AI also gets things wrong

In the same comment, the EKS finding says the public endpoint "lets
anyone on the internet access the cluster API and read secrets". That's an
exaggeration: an endpoint open to `0.0.0.0/0` greatly increases exposure, but
whoever reaches it still has to authenticate with IAM and have permissions in the cluster's
RBAC. That's why the footer of every comment says the AI is support, not
approval.

To test it in your repository, set up the action, copy one of these files and
open a PR.

If your manifest has probe problems, the post
[Kubernetes: configuring liveness, readiness and startup probes]({{< relref "/posts/2025/2025-02-15-kubernetes-configurando-liveness-readiness-e-startup-probes" >}})
helps you fix them.

## Conclusion

With a Python script packaged into an action, every infrastructure PR
gets a second reading that explains the risks in plain language,
with no hard-coded credentials and without giving the AI the power to approve anything.

The full code, with the action's documentation, is at
[github-actions/iac-ai-review](https://github.com/jjasonhenrique1986/github-actions/tree/main/iac-ai-review).

## References

- [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- [Creating a composite action](https://docs.github.com/en/actions/sharing-automations/creating-actions/creating-a-composite-action)
- [checkov](https://www.checkov.io/)
- [Trivy](https://trivy.dev/)
- [kube-linter](https://github.com/stackrox/kube-linter)
- [kubeconform](https://github.com/yannh/kubeconform)
- [marocchino/sticky-pull-request-comment](https://github.com/marocchino/sticky-pull-request-comment)
