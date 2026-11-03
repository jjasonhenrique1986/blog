---
title: "Testando o falco em uma máquina virtual"
description: "Segundo artigo sobre o Falco: instalação em uma máquina virtual na AWS e alteração do output de uma regra."
image: "/images/falco.png"
author: Jason
date: 2023-11-22
tags: [ "Kubernetes", "Segurança", "Containers", "AWS" ]
type: post
showTableOfContents: true
---

![Testando o falco em uma máquina virtual](/images/falco.png)

O objetivo deste segundo artigo falando sobre a ferramenta falco é
demonstrar a utilização do falco instalando em uma máquina virtual na
AWS e fazer uma alteração no output de uma regra do falco.

Para ler meu primeiro artigo sobre o falco basta acessar esse
[link](/posts/2023/2023-11-16-instalando-e-testando-a-ferramenta-falco/).
Nesse artigo eu instalei o falco no Kubernetes.

## Pré-requisitos 

Abaixo os pré-requisitos necessários para seguir esse artigo:

-   Conta AWS.
-   Uma máquina virtual na AWS (EC2).
-   EC2 deve ter acesso a Internet.
-   EC2 deve estar acessível via SSH.

## Instalação 

Para a instalação irei executar uma EC2 rodando Ubuntu 22.04. Abaixo, os
comandos utilizados para a instalação do falco:

Adicionar a chave do repositório do Falco

``` bash
    curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | \
    sudo gpg --dearmor -o /usr/share/keyrings/falco-archive-keyring.gpg
```

Adicionar o repositório do Falco

``` bash
echo "deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] https://download.falco.org/packages/deb stable main" > /etc/apt/sources.list.d/falcosecurity.list
```

Fazer o update do SO executando o comando abaixo:

``` bash
apt-get update -y
```

Após finalizar o update, instale alguns pacotes obrigatórios para a
instalação:

``` bash
apt-get install -y dkms make linux-headers-$(uname -r) dialog
```

Agora sim vamos fazer a instalação do falco usando o comando abaixo:

``` bash
apt-get install -y falco
```

Deve ser escolhida a opção eBPF conforme abaixo:

![Tela de instalação do Falco para escolher o driver, com a opção eBPF selecionada](/images/falco-4.webp)

Para verificar se o Falco foi instalado com sucesso, execute o comando abaixo:

``` bash
systemctl status falco-bpf
```

Deve apresentar uma saída conforme tela abaixo:

![Saída do systemctl status com o serviço falco-bpf ativo na máquina virtual](/images/falco-5.webp)

## Demonstração 

Agora vamos validar se o falco realmente está funcionando executando o
comando abaixo:

``` bash
cat /etc/shadow
```

E depois vamos verificar os logs do falco executando o comando abaixo:

``` bash
grep Sensitive /var/log/syslog
```

Deve apresentar a seguinte saída:

![Alertas do Falco de arquivo sensível /etc/shadow aberto por um programa não confiável](/images/falco-6.webp)

## Alteração regra do Falco 

Para verificar todas as regras padrão do Falco, acesse o arquivo
/etc/falco/falco_rules.yaml. Vamos
alterar o output da regra Read sensitive file untrusted de Sensitive
file opened for reading by non-trusted program para ACESSO INDEVIDO.

Primeiramente, vamos copiar o conteúdo abaixo, da linha 396 até a 431, para o
arquivo /etc/falco/falco_rules.local.yaml, que o Falco carrega depois das regras padrão. É uma boa prática colocar as regras
customizadas nesse arquivo.

![Regra Read sensitive file untrusted no arquivo de regras do Falco, com condição, output e prioridade](/images/falco-7.webp)

A seção output deve ser alterada conforme abaixo:

![Linha output da regra alterada para exibir a mensagem ACESSO INDEVIDO](/images/falco-8.webp)

Depois devemos salvar o arquivo **/etc/falco/falco_rules.local.yaml**, parar e
iniciar o serviço do falco usando os comandos abaixo:

``` bash
systemctl stop falco-bpf
systemctl start falco-bpf
```

Para verificar se o falco está funcionando executar o comando abaixo:

``` bash
systemctl status falco-bpf
```

Agora vamos executar o comando cat /etc/shadow novamente, mas o grep
deve ser modificado:

``` bash
grep ACESSO /var/log/syslog
```

Por fim deve apresentar uma saída conforme abaixo:

![Alertas do Falco com a nova mensagem ACESSO INDEVIDO ao ler o /etc/shadow](/images/falco-9.webp)

## Conclusão 

O objetivo desse artigo era fazer a instalação do falco em uma máquina
virtual e também a alteração de uma regra do falco.

Para mais informações sobre as regras padrão do Falco, acesse esse
[link](https://falco.org/docs/rules/).