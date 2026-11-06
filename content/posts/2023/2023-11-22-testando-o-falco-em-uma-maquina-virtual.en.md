---
title: "Testing Falco on a virtual machine"
slug: "2023-11-22-testing-falco-on-a-virtual-machine"
description: "Second article about Falco: installing it on a virtual machine on AWS and changing a rule's output."
images: ["/images/capa-testando-o-falco-em-uma-maquina-virtual.png"]
author: Jason
date: 2023-11-22
tags: [ "Kubernetes", "Security", "Containers", "AWS" ]
type: post
---

The goal of this second article about the Falco tool is to
show how to use Falco by installing it on a virtual machine on
AWS and changing the output of a Falco rule.

The first article is [Installing and testing the Falco
tool]({{< relref "/posts/2023/2023-11-16-instalando-e-testando-a-ferramenta-falco" >}}).
In that article I installed Falco on Kubernetes.

## Prerequisites

Below are the prerequisites to follow this article:

-   An AWS account.
-   A virtual machine on AWS (EC2).
-   The EC2 instance must have internet access.
-   The EC2 instance must be reachable over SSH.

## Installation

For the installation I'll run an EC2 instance with Ubuntu 22.04. Below are the
commands used to install Falco:

Add the Falco repository key

``` bash
    curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | \
    sudo gpg --dearmor -o /usr/share/keyrings/falco-archive-keyring.gpg
```

Add the Falco repository

``` bash
echo "deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] https://download.falco.org/packages/deb stable main" > /etc/apt/sources.list.d/falcosecurity.list
```

Update the OS by running the command below:

``` bash
apt-get update -y
```

Once the update is done, install a few packages required for the
installation:

``` bash
apt-get install -y dkms make linux-headers-$(uname -r) dialog
```

Now let's install Falco with the command below:

``` bash
apt-get install -y falco
```

Choose the eBPF option, as below:

![Falco installation screen to choose the driver, with the eBPF option selected](/images/falco-4.webp)

To check whether Falco was installed successfully, run the command below:

``` bash
systemctl status falco-bpf
```

It should show output like the screen below:

![systemctl status output with the falco-bpf service active on the virtual machine](/images/falco-5.webp)

## Demo

Now let's check whether Falco is really working by running the
command below:

``` bash
cat /etc/shadow
```

And then let's check Falco's logs by running the command below:

``` bash
grep Sensitive /var/log/syslog
```

It should show the following output:

![Falco alerts for the sensitive file /etc/shadow opened by an untrusted program](/images/falco-6.webp)

## Changing a Falco rule

To see all of Falco's default rules, open the
/etc/falco/falco_rules.yaml file. We'll
change the output of the Read sensitive file untrusted rule from Sensitive
file opened for reading by non-trusted program to ACESSO INDEVIDO
(Portuguese for "unauthorized access").

First, let's copy the content below, from line 396 to 431, to the
/etc/falco/falco_rules.local.yaml file, which Falco loads after the default rules. It's good practice to put custom
rules in this file.

![Read sensitive file untrusted rule in the Falco rules file, with condition, output and priority](/images/falco-7.webp)

The output section should be changed as below:

![Rule output line changed to show the message ACESSO INDEVIDO](/images/falco-8.webp)

Then save the **/etc/falco/falco_rules.local.yaml** file, and stop and
start the Falco service with the commands below:

``` bash
systemctl stop falco-bpf
systemctl start falco-bpf
```

To check that Falco is working, run the command below:

``` bash
systemctl status falco-bpf
```

Now let's run cat /etc/shadow again, but the grep
needs to change:

``` bash
grep ACESSO /var/log/syslog
```

Finally, it should show output like the one below:

![Falco alerts with the new ACESSO INDEVIDO message when reading /etc/shadow](/images/falco-9.webp)

## Conclusion

The goal of this article was to install Falco on a virtual
machine and also change a Falco rule.

For more information, see the [Falco rules
documentation](https://falco.org/docs/rules/).
