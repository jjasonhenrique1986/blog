---
title: "Minha Jornada para certificação AWS Developer"
description: "Como me preparei em 4 meses para a certificação AWS Certified Developer Associate e os materiais de estudo que usei até a aprovação."
image: "/images/dca.jpg"
author: Jason
date: 2022-09-03
tags: [ "AWS", "Certificações" ]
type: post
showTableOfContents: true
---
![image-1](/images/dca.jpg)

Recentemente tirei a certificação **AWS Certified Developer Associate**
(DVA-C01) e resolvi compartilhar sobre minha jornada de estudos e os
materiais que usei durante esses 4 meses de preparação até conseguir a
aprovação.

## Sobre a prova

A prova é composta de 65 perguntas de múltipla escolha ou com várias
respostas certas em um tempo de 130 minutos para a prova em português
(não recomendo). No meu caso eu fiz a prova em inglês e tive um
acréscimo de mais 30 minutos por não ter o inglês como língua nativa.

A prova custa USD 150 ou caso já seja certificado AWS você ganha 50% de
desconto na próxima certificação. Atualmente a prova pode ser realizada
pela Pearson VUE e PSI presencialmente ou remotamente.

Abaixo seguem os domínios que são exigidos na prova:

![Tabela com os domínios do exame AWS Developer Associate e o peso de cada um: Implantação 22%, Segurança 26%, Desenvolvimento 30%, Refatoração 10% e Monitoramento 12%](/images/conteudo-dca.webp)

## Minha Preparação

Minha principal fonte de estudo foi o curso da Udemy do Stephane Maarek
onde procurei colocar em prática todos os exemplos de labs que o
Stephane executa durante o curso.

Alguns pontos que vale destacar:

-   Entender sobre os principais recursos do Lambda;
-   Saber calcular RCU e WCU do DynamoDB;
-   Entender o funcionamento do API Gateway;
-   Entender sobre os componentes do ECS;
-   Ter domínio sobre o awscli;
-   Entender quando usar exponential backoff;
-   Entender as diferenças de filas SQS e suas opções;
-   Entender os diferentes tipos de Kinesis e quando utilizar cada um;
-   Formas de utilizar ElastiCache;

Apesar de ser uma prova focada nos serviços de desenvolvimento da AWS,
como Lambda, API Gateway e DynamoDB, ainda são cobrados alguns serviços
do exame SysOps, como S3, IAM, Parameter
Store, Secrets Manager e KMS.

No final dos meus estudos também fiz dois cursos AWS Skill Builder que
também me ajudaram bastante. Vou colocar os links na seção de
referências.

## Simulados

O principal simulado que utilizei foi o da Whizlabs, que na minha
opinião é o mais parecido com a prova, mas também utilizei dois outros
simulados que comprei na Udemy. Abaixo coloquei os simulados que utilizei:

- [Whizlabs](https://www.whizlabs.com/learn/course/aws-developer-associate/)
- [Simulado1](https://www.udemy.com/course/aws-developer-associate-practice-exams/)
- [Simulado2](https://www.udemy.com/course/aws-certified-developer-associate-practice-tests-dva-c01/)

## Conclusão

Essa certificação foi a quarta certificação AWS que tirei e até o
momento foi a prova mais divertida e a que mais me ensinou durante a
preparação. Por ter um background de infraestrutura, tive que aprender e
me aprofundar em alguns serviços da AWS como Lambda, API Gateway e
DynamoDB.

## Referências

Seguem abaixo algumas referências que utilizei para escrever esse post e
também na minha preparação:

- [Documentação
    Oficial](https://aws.amazon.com/pt/certification/certified-developer-associate/?ch=tile&tile=getstarted)
- [Exam
  Guide](https://d1.awsstatic.com/pt_BR/training-and-certification/docs-dev-associate/AWS-Certified-Developer-Associate_Exam-Guide.pdf)
- [Curso
    Udemy](https://www.udemy.com/course/aws-certified-developer-associate-dva-c01/)
- [Cursos AWS Skill
    Builder](https://explore.skillbuilder.aws/learn/course/internal/view/elearning/42/exam-readiness-aws-certified-developer-associate-digital?da=sec&sec=prep)
- [Plano de estudo na AWS Skill
    Builder](https://explore.skillbuilder.aws/learn/lp/1022/developer-learning-plan-portuguese)

