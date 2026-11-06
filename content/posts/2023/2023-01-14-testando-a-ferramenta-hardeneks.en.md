---
title: "Testing the HardenEKS tool"
slug: "2023-01-14-testing-the-hardeneks-tool"
description: "Testing HardenEKS, a command-line tool that checks a Kubernetes cluster against AWS's EKS best practices guide."
images: ["/images/capa-testando-a-ferramenta-hardeneks.png"]
author: Jason
date: 2023-01-14
tags: [ "AWS", "Kubernetes", "EKS", "Containers" ]
type: post
---

HardenEKS is a command-line tool developed by [Doruk
Ozturk](https://github.com/dorukozturk), who currently works at AWS. The tool's
goal is to check a Kubernetes cluster against a best practices
guide created by AWS. The guide is at [EKS Best Practices
Guides](https://aws.github.io/aws-eks-best-practices/).

For more information, see the [HardenEKS repository on
GitHub](https://github.com/aws-samples/hardeneks).

## Prerequisites

As a prerequisite to run the tool, you need to already have
access to a Kubernetes cluster configured through the
kubeconfig file. For information about this file, see the [kubeconfig
documentation](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/).

In this demo I'll use a cluster created with Rancher
Desktop. For more information, see the [official Rancher
Desktop website](https://rancherdesktop.io/).

To run the tool against an EKS cluster, some minimum
permissions are required, both on AWS (IAM Policy) and
inside the EKS cluster (RBAC).

Below are the minimum permissions required for the IAM Policy:

``` json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "eks:ListClusters",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "eks:DescribeCluster",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "ecr:DescribeRepositories",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "inspector2:BatchGetAccountStatus",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "ec2:DescribeFlowLogs",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "ec2:DescribeInstances",
            "Resource": "*"
        }
    ]
}
```

Below is the ClusterRole to create in the cluster:

``` yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: hardeneks-runner
rules:
- apiGroups: [""]
  resources: ["namespaces", "resourcequotas", "persistentvolumes", "pods", "services"]
  verbs: ["list"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles", "clusterrolebindings", "roles", "rolebindings"]
  verbs: ["list"]
- apiGroups: ["networking.k8s.io"]
  resources: ["networkpolicies"]
  verbs: ["list"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["list"]
- apiGroups: ["apps"]
  resources: ["deployments", "daemonsets", "statefulsets"]
  verbs: ["list", "get"]
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["list"]
```

## Installation

To install the tool I'll use a Windows machine.
On **Windows** you'll need to install Python from the link
below:

[Python download
link](https://www.python.org/downloads/)

On **Linux** most distributions already ship Python,
but if needed, follow your distribution's instructions or the
[Python downloads page](https://www.python.org/downloads/).

To install the tool, run the following command:

``` bash
pip3 install hardeneks
```

To check whether the package was installed successfully, run `pip3 show hardeneks`:

![pip3 show hardeneks output showing version 0.7.2 installed](/images/hardeneks-2.webp)

## Overview

To use the tool, just run the command below:

``` bash
hardeneks
```

Below are some screenshots of the tool's output:

![hardeneks run on the kubecost namespace with the failed cluster rules, such as ClusterRoles with wildcards](/images/hardeneks-3.webp)

![hardeneks result on the opencost namespace with recommendations such as a read-only root file system, SSL certificate and HPA](/images/hardeneks-4.webp)

![hardeneks result on the opencost namespace with recommendations such as setting requests and limits for each container](/images/hardeneks-5.webp)

At the end of each check, the tool shows a link to the
best practices guide.

You can also pass a hardeneks configuration file to
disable some checks. Below is a configuration file in which
some checks can be disabled:

``` yaml
---
ignore-namespaces:
  - kube-node-lease
  - kube-public
  - kube-system
  - kube-apiserver
  - karpenter
  - kubecost
  - external-dns
  - argocd
  - aws-for-fluent-bit
  - amazon-cloudwatch
  - vpa
rules: 
  cluster_wide:
    security:
      iam:
        - disable_anonymous_access_for_cluster_roles
        - check_endpoint_public_access
        - check_aws_node_daemonset_service_account
        - check_access_to_instance_profile
        - restrict_wildcard_for_cluster_roles
      multi_tenancy:
        - ensure_namespace_quotas_exist
      detective_controls:
        - check_logs_are_enabled
      network_security:
        - check_vpc_flow_logs
        - check_awspca_exists
        - check_default_deny_policy_exists
      encryption_secrets:
        - use_encryption_with_ebs
        - use_encryption_with_efs
        - use_efs_access_points
      infrastructure_security:
        - deploy_workers_onto_private_subnets
        - make_sure_inspector_is_enabled
      pod_security:
        - ensure_namespace_psa_exist
      image_security:
        - use_immutable_tags_with_ecr
    reliability:
      applications:
        - check_metrics_server_is_running
        - check_vertical_pod_autoscaler_exists
  namespace_based:
    security: 
      iam:
        - disable_anonymous_access_for_roles
        - restrict_wildcard_for_roles
        - disable_service_account_token_mounts
        - disable_run_as_root_user
        - use_dedicated_service_accounts_for_each_deployment
        - use_dedicated_service_accounts_for_each_stateful_set
        - use_dedicated_service_accounts_for_each_daemon_set
      pod_security:
        - disallow_container_socket_mount
        - disallow_host_path_or_make_it_read_only
        - set_requests_limits_for_containers
        - disallow_privilege_escalation
        - check_read_only_root_file_system
      network_security:
        - use_encryption_with_aws_load_balancers
      encryption_secrets:
        - disallow_secrets_from_env_vars    
      runtime_security:
        - disallow_linux_capabilities
    reliability:
      applications:
        - check_horizontal_pod_autoscaling_exists
        - schedule_replicas_across_nodes
        - run_multiple_replicas
        - avoid_running_singleton_pods
```

Other options the tool has:

- Run the check on a specific namespace;
- Run the check in a specific region;
- Run the check on a specific context or by cluster
    name;
- Export the report to HTML or TXT;

One idea would be to run this tool in
a CI/CD pipeline.

## Conclusion

The idea of this post was to show how to use the hardeneks tool
to check best practices when using Kubernetes.
