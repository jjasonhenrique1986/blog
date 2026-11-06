---
title: "My journey to the AWS SysOps Administrator certification"
slug: "2024-05-03-my-journey-to-the-aws-sysops-administrator-certification"
description: "How I prepared in 1 month for the AWS Certified SysOps Administrator Associate (SOA-C02) certification and the materials I used."
images: ["/images/capa-minha-jornada-para-a-certificacao-aws-sysops-administrator.png"]
author: Jason
date: 2024-05-03
lastmod: 2026-11-03
tags: [ "AWS", "Certifications" ]
type: post
---
![AWS Certified SysOps Administrator – Associate badge](/images/soa.png)

> **Update (November 2026):** AWS renamed this certification to
> **AWS Certified CloudOps Engineer – Associate**, with a new version of the
> exam (SOA-C03). Those who earned SysOps before the change keep the old
> name. The domains, materials and practice exams below are the ones for SOA-C02. More
> details on the
> [certification page](https://aws.amazon.com/certification/certified-cloudops-engineer-associate/).

On May 2 I passed the AWS Certified
SysOps Administrator Associate (SOA-C02) certification, so I decided to share
my study journey and the materials I used during that 1
month of preparation until I passed.

This was the second time I earned the AWS Certified SysOps
Administrator Associate certification, since I had already taken this exam in 2021.

## About the exam

The exam has 65 multiple-choice or multiple-response questions,
with 130 minutes when taken in Portuguese.
In my case I took the exam in English and got an extra 30 minutes
because English isn't my native language.

The exam costs USD 150, but if you already have another AWS certification,
you can get a 50% discount voucher. Currently the exam
can be taken through Pearson VUE, in person or online. In
my case, I decided to take it at a Pearson VUE
authorized test center in Campinas (SP), the city where I live.

Below are the domains covered by the exam:

-   Domain 1: Monitoring, Logging, and Remediation (20% of scored
    content)
-   Domain 2: Reliability and Business Continuity (16% of scored
    content)
-   Domain 3: Deployment, Provisioning, and Automation (18% of scored
    content)
-   Domain 4: Security and Compliance (16% of scored content)
-   Domain 5: Networking and Content Delivery (18% of scored content)
-   Domain 6: Cost and Performance Optimization (12% of scored
    content)

For more information, see the [SOA-C02 exam guide
(PDF)](https://d1.awsstatic.com/training-and-certification/docs-sysops-associate/AWS-Certified-SysOps-Administrator-Associate_Exam-Guide.pdf).

## My preparation

My main study source was Stephane Maarek's Udemy course,
where I tried to put into practice every lab example that
Stephane runs during the course.
See [Stephane Maarek's SysOps course on
Udemy](https://www.udemy.com/course/ultimate-aws-certified-sysops-administrator-associate).

Below are some topics and services worth highlighting that may
show up on the exam:

-   Understand when to use Multi-AZ or read replicas in RDS
-   Understand how to encrypt an RDS instance that isn't encrypted
-   Understand how to encrypt an EBS volume
-   Understand EKS
-   Advanced knowledge of EC2 and autoscaling configuration
-   Understand basic networking concepts on AWS
-   Understand what an Elastic IP is and when to use it
-   Understand when to use an IAM Role and how to configure it
-   Be able to analyze an IAM Policy and identify what it does
-   Understand the CloudTrail service
-   Understand AWS Config and how to use it
-   Understand AWS Systems Manager
-   Understand how to scale an ElastiCache cluster
-   Understand the difference between RDS (MySQL) and Aurora
-   Understand when to use CloudFront and its settings
-   Understand the difference between a security group and a NACL
-   Understand how to run a Windows workload on EC2
-   Understand how to create and configure alarms in CloudWatch
-   When you need to install the CloudWatch Agent to collect
    metrics or logs
-   Advanced knowledge of Load Balancer configuration
-   Understand when to use an ALB or an NLB
-   Understand the SNS service

## Practice exams

The main practice exam I used was Whizlabs', which in my opinion is
the one most similar to the real exam. See the [Whizlabs SysOps
practice exam](https://www.whizlabs.com/aws-sysops-administrator-associate/).

Another practice exam I recommend for your preparation is the [SysOps
practice exam on
Udemy](https://www.udemy.com/course/aws-certified-sysops-administrator-associate-practice-exams-soa-c02/).

## Conclusion

The purpose of this post was to share my experience while
preparing for the AWS Certified SysOps Administrator
Associate certification, as well as the study materials and practice exams. Below
are some other links that may help you too:

-   <https://aws.amazon.com/pt/certification/certified-sysops-admin-associate/>
-   <https://d1.awsstatic.com/training-and-certification/docs-sysops-associate/AWS-Certified-SysOps-Administrator-Associate_Sample-Questions.pdf>
