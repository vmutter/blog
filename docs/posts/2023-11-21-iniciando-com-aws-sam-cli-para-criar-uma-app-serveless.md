---
date: 2023-11-21
slug: iniciando-app-serveless-aws-sam
readtime: 30
authors:
  - vmutter
categories:
  - AWS
  - AWS SAM
  - Serveless
tags:
  - AWS
  - AWS SAM
  - Serveless
comments: true
---

# Iniciando uma aplicação serveless utilizando o AWS SAM

## Pré Requisitos

É fundamental garantir que tudo esteja configurado corretamente. Certifique-se de que todas as dependencias estejam instaladas e configuradas. 

- Docker
- SAM CLI
- AWS CLI
- Git
- Uma das linguagens abaixo que se sentir mais familiarizado para criar a aplicação
  - Node.js
  - Python
  - Java (iremos utilizar Java 11 com Maven para os exemplos)

<!-- more -->

## Criando uma aplicação sem servidor com AWS  SAM

``` bash
sam init
```

Nas opções selecionar:

```
Which template source would you like to use?
        1 - AWS Quick Start Templates
        ...
Choice: 1

Choose an AWS Quick Start application template
        1 - Hello World Example
        ...
Template: 1

Use the most popular runtime and package type? (Python and zip) [y/N]: n
```

Após selecionado, iremos selecionar o _runtime_ e a versão, so caso, Java 11.

``` bash
        ...
        10 - java11
        ...
```

E em seguida, a forma de empacotamento e gerenciador de dependencias, selecionar `Zip` e `maven` , para as demais perguntas, responder `n` e para o nome da aplicação manter o padrão `sam-app`.

``` bash
What package type would you like to use?
        1 - Zip
        2 - Image
Package type: 1

Which dependency manager would you like to use?
        1 - gradle
        2 - maven
Dependency manager: 2

Would you like to enable X-Ray tracing on the function(s) in your application?  [y/N]: n

Would you like to enable monitoring using CloudWatch Application Insights?
For more info, please view https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-application-insights.html [y/N]: n

Project name [sam-app]:
```

Com isso teremos o projeto `sam-app` inicializado.

## Rodando localmente

### Instalando as dependencias do projeto

Para instalar as dependencias, acessar a pasta do projeto e rodar:

``` bash
mvn install
```

Para fazer um _build_ da aplicação e gerar os artefatos para _deploy_, executar:

``` bash
sam build
```

### Rodando uma função Lambda localmente

Para rodar uma função Lambda individualmente:

``` bash
sam local invoce --events events/event.json
```

Também é possível rodar criando um servidor HTTP local com:

```bash
sam local start-api --port 8080
```

### Rodando os testes unitários

De dentro da pasta da função Lambda:

``` bash
mvn test
```

## Fazendo o deploy manual

Para fazer o deploy manualmente pela primeira vez, com a ajuda do CLI:

``` bash
sam deploy --guided
```

No console:

``` bash
Configuring SAM deploy
======================

Looking for config file [samconfig.toml] :  Not found

Setting default arguments for 'sam deploy'
=========================================
Stack Name [sam-app]:
AWS Region [us-west-2]:
#Shows you resources changes to be deployed and require a 'Y' to initiate deploy
Confirm changes before deploy [y/N]:
#SAM needs permission to be able to create roles to connect to the resources in your template
Allow SAM CLI IAM role creation [Y/n]:
#Preserves the state of previously provisioned resources when an operation fails
Disable rollback [y/N]:
HelloWorldFunction may not have authorization defined, Is this okay? [y/N]: y
Save arguments to configuration file [Y/n]:
SAM configuration file [samconfig.toml]:
SAM configuration environment [default]:
```
 
## CI/CD com GitHub
## SAM Accelarate
## Gerenciando permissões