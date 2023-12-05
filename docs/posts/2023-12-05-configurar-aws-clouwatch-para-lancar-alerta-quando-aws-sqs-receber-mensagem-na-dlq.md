---
date: 2023-12-05
slug: configurar-cloudwatch-lancar-alerta-sqs-dlq
readtime: 15
authors:
  - vmutter
categories:
  - AWS
  - AWS CloudWatch
  - AWS SQS
tags:
  - AWS
  - AWS CloudWatch
  - AWS SQS
comments: true
---

# Configurar AWS CloudWatch para lançar alerta quando AWS SQS receber mensagem na DLQ

## AWS CloudWatch Metrics

Acessar `Metrics` e em `All metrics` selecionar `ApproximateNumberOfMessagesVisible` e `ApproximateNumberOfMessagesNotVisible` da fila DLQ da fila que se deseja monitorar.

<!-- more -->

- Na aba `Graphed metrics`, selecionar as métricas de `ApproximateNumberOfMessagesVisible` e `ApproximateNumberOfMessagesNotVisible` e clciar em `Add math` > `All functions` >  `RATE`.

- Na coluna `Details`, editar a fórmula para `RATE(m1+m2)`

- Na coluna `Actions`, clicar no ícone `Create alarm`

## AWS CloudWatch Alarms

### Step 1: Specify metric and conditions


- Em `Metrics`, alterar valores conforme necessidade, opcional.

- Em `Conditions`, deixar `Greater` e no valor do threshold, colocar 0 (zero), com isso qualquer mensagem que cair da DLQ, irá disparar um alerta. 

### Step 2: Configure actions

- Selecionar as opções confome necessidade, no caso, enviando para um tópico do SNS para fazer a notificação.

```
Alarm state trigger: In alarm
Send notification to the following SNS topic: Select an existing SNS topic ou Create new topic
Send a notification to...: Selecionar o tópico desejado
```

- Pode-se alterar as demais opções conforme necessário, senão clicar em `Next`.


### Step 3: Add name and description

- Adicionar um nome para o alarme criado.

- Caso deseje, pode-se adicionar uma descrição em formato Markdown.

- Ao finalizar, clicar em `Next`.


### Step 4: Preview and create

- Revise e valide as informações referentes ao alarme criado, e se estiver tudo certo, clique em `Create alarm`.