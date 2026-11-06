---
title: "Testing the AWS CodeWhisperer tool"
slug: "2023-01-24-testing-the-aws-codewhisperer-tool"
description: "Testing AWS CodeWhisperer, an AWS tool that suggests code using Machine Learning, similar to GitHub Copilot."
images: ["/images/capa-testando-a-ferramenta-aws-codewhisperer.png"]
author: Jason
date: 2023-01-24
lastmod: 2026-11-03
tags: [ "AWS", "Security", "AI" ]
type: post
---

> **Update (November 2026):** in April 2024 CodeWhisperer became
> part of **Amazon Q Developer**, and the CodeWhisperer name is no longer
> used. The lists of languages and IDEs below are the ones from when this post was written.

The goal of this post is to show how to use AWS CodeWhisperer,
a tool that suggests code to a
developer using Machine Learning and Artificial Intelligence. This
tool is similar to GitHub Copilot.

The tool supports the following languages:

-   C#
-   Java
-   JavaScript
-   Python
-   TypeScript

And it integrates with the following IDEs:

-   IntelliJ IDEA
-   PyCharm
-   VS Code
-   AWS Cloud9
-   AWS Lambda

For this demo I'll use VS Code. For more information,
see the [CodeWhisperer page on
AWS](https://aws.amazon.com/pt/codewhisperer/).

The service can also run a security scan of your code.

## Prerequisites

To test this AWS service you need the following
prerequisites:

-   VS Code installed. Download it from the [VS
    Code download page](https://code.visualstudio.com/download).
-   The AWS Toolkit extension installed in VS Code, as in the image below:

![AWS Toolkit extension in the VS Code marketplace](/images/codewhisperer-2.webp)

## Installation and configuration

After installing the AWS Toolkit extension, an AWS icon will show up on the
left side, similar to the image below:

![AWS Toolkit Developer Tools panel connected with AWS Builder ID and the CodeWhisperer options](/images/codewhisperer-3.png)

To start using AWS CodeWhisperer you need to
configure the connection to AWS.

I used the **"Use a personal email to sign up and sign in with
AWS Builder ID"** option. When you click it, a web page opens to
create your AWS Builder ID.

![AWS Toolkit connection options: AWS Builder ID, IAM Identity Center or IAM credentials](/images/codewhisperer-4.png)

## Demo

For the demo I created a file with the .py extension and added
a comment describing what I wanted to create, for example "Create simple
API with fastAPI". It then keeps suggesting code, as in
the image below:

![VS Code with CodeWhisperer suggesting the code for a FastAPI API from a comment](/images/codewhisperer-5.webp)

This was the final result produced by AWS CodeWhisperer:

![FastAPI API code with the / and /about routes generated with CodeWhisperer's suggestions](/images/codewhisperer-6.png)

## Conclusion

I really liked this AWS service. It can be a very
interesting alternative to GitHub Copilot.
