---
date: 2024-01-03
slug: tracing-com-spring-boot-2-2-spring-cloud-sleuth-e-aws-sqs
readtime: 20
authors:
  - vmutter
categories:
  - Java
  - Spring Boot
  - Spring Cloud
  - AWS
tags:
  - Java
  - Spring
  - Spring Boot
  - Spring Cloud
  - AWS
  - AWS SQS
  - Trace
  - Tracing
comments: true
---

# Tracing com Spring Boot 2.2, Spring Cloud Sleuth e AWS SQS

## Configurando uma fila SQS local com Localstack

Criar um arquivo `docker-compose.yml` com o seguinte conteúdo:

``` yaml title="docker-compose.yml"
version: '3.7'
services:
  aws:
    container_name: localstack
    image: localstack/localstack:latest
    environment:
      - SERVICES=sqs
      - DEFAULT_REGION=us-east-1
      - DEBUG=0
      - DATA_DIR=/tmp/localstack/data

    ports:
      - '4566:4566'
    volumes:
      - ./localstack_bootstrap:/etc/localstack/init/ready.d/
    networks:
      - localstack

networks:
  localstack:
    driver: bridge
```

<!-- more -->

Nas configurações de `volume`, mapeado a pasta `./localstack_bootstrap` para a `/etc/localstack/init/ready.d/`, com isso, todos os scripts que estiverem nessa pasta, serão executados após a inicialização do Localstack.

Agora, vamos criar a pasta, `localstack_bootstrap` e adicionar o script `sqs_bootstrap.sh` abaixo:

``` bash title="sqs_bootstrap.sh"
#!/usr/bin/env bash

set -euo pipefail

# enable debug
# set -x

echo "configuring sqs"
echo "==================="
LOCALSTACK_HOST=localhost
AWS_REGION=us-east-1

create_queue() {
    local QUEUE_NAME_TO_CREATE=$1
    aws --endpoint-url=http://${LOCALSTACK_HOST}:4566 sqs create-queue --queue-name ${QUEUE_NAME_TO_CREATE} --region ${AWS_REGION} --attributes VisibilityTimeout=30,FifoQueue=true
}

create_queue "sample-queue.fifo"
```

Com isso será criado uma fila FIFO no SQS com o nome `sample-queue.fifo`.

Para iniciar o *LocalStack* execute:
- `docker compose up` para iniciar no modo interativo e anexar o docker ao console, para visualizar os logs ou;
- `docker compose up -d` o para iniciar no modo background, sem anexar ao console. 

## Producer

### pom.xml

Configurando as dependências do Spring Boot 2.2 e Spring Cloud.
Atenção para as versões compatíveis de ambos, para mais informações: [Documentação Spring Cloud](https://spring.io/projects/spring-cloud/)

``` xml title="pom.xml parcial"

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.13.RELEASE</version>
        <relativePath/>
    </parent>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Hoxton.SR12</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-sleuth</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-aws-messaging</artifactId>
        </dependency>

        <dependency>
            <groupId>io.zipkin.aws</groupId>
            <artifactId>brave-instrumentation-aws-java-sdk-sqs</artifactId>
            <version>0.21.2</version>
        </dependency>
        
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
    </dependencies>

```

### application.yml

``` yaml title="application.yml"
spring:
  application:
    name: producer-app

cloud:
  aws:
    stack: false
    stack.auto: false
    region:
      static: us-east-1
      auto: false
    credentials:
      access-key: ANUJDEKAVADIYAEXAMPLE
      secret-key: 2QvM4/Tdmf38SkcD/qalvXO4EXAMPLEKEY
    queue:
      uri: http://localhost:4566
      name: sample-queue.fifo
```

`spring.application.name` para adicionar o nome da  aplicação, além do trace e span id.

`cloud.aws.*` são as configurações para se conectar a fila FIFO do SQS criado no Localstack.

### Configurando o cliente da fila SQS

``` java title="SQSConfig.java"
@Configuration
public class SQSConfig {

    @Value("${cloud.aws.region.static}")
    private String region;

    @Value("${cloud.aws.credentials.access-key}")
    private String accessKeyId;

    @Value("${cloud.aws.credentials.secret-key}")
    private String secretAccessKey;

    @Value("${cloud.aws.queue.uri}")
    private String sqsUrl;

    @Bean
    @Primary
    public AmazonSQSAsync amazonSQSAsync() {
        Tracing current = Tracing.current();
        SqsMessageTracing sqsMessageTracing = SqsMessageTracing.create(current);

        return AmazonSQSAsyncClientBuilder.standard()
                .withEndpointConfiguration(new AwsClientBuilder.EndpointConfiguration(sqsUrl, region))
                .withCredentials(new AWSStaticCredentialsProvider(new BasicAWSCredentials(accessKeyId, secretAccessKey)))
                .withRequestHandlers(sqsMessageTracing.requestHandler())
                .build();
    }

    @Bean
    public QueueMessagingTemplate queueMessagingTemplate() {
        return new QueueMessagingTemplate(amazonSQSAsync());
    }

}
```

Atenção para adicionar o `SqsMessageTracing` no handler do `AmazonSQSAsync`.

### Enviando uma mensagem com o AWS SDK

``` java title="Producer.java"
@Slf4j
@Service
@RequiredArgsConstructor
public class Producer {

    @Value("${cloud.aws.queue.uri}")
    private String sqsUrl;

    @Value("${cloud.aws.queue.name}")
    private String sqsName;

    private final AmazonSQSAsync async;
    private final ObjectMapper objectMapper;

    public void sendMessage() throws JsonProcessingException {
        String message = objectMapper.writeValueAsString(Collections.singletonMap("key", "value"));

        SendMessageRequest batchRequest = new SendMessageRequest()
                .withQueueUrl(sqsUrl + "/000000000000/" + sqsName)
                .withMessageBody(message)
                .withMessageGroupId(UUID.randomUUID().toString())
                .withMessageDeduplicationId(UUID.randomUUID().toString());

        log.info("sending message [{}]", message);
        async.sendMessage(batchRequest);
    }
}
```

### Enviando uma mensagem com o Spring Cloud AWS Messaging

``` java title="Producer.java"
@Slf4j
@Service
@RequiredArgsConstructor
public class Producer {

    @Value("${cloud.aws.queue.name}")
    private String sqsName;

    private final AmazonSQSAsync async;

    private final QueueMessagingTemplate messagingTemplate;

    public void sendMessage() {
        Map<String, String> payload = Collections.singletonMap("key", "value");

        Map<String, Object> headers = new HashMap<>();
        headers.put("message-group-id", UUID.randomUUID().toString());
        headers.put("message-deduplication-id", UUID.randomUUID().toString());

        log.info("sending message [{}]", payload);
        messagingTemplate.convertAndSend(sqsName, payload, headers);
    }
}
```

## Consumer

### pom.xml

``` xml title="pom.xml parcial"
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.13.RELEASE</version>
        <relativePath/>
    </parent>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Hoxton.SR12</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-sleuth</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-aws-autoconfigure</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-aws-messaging</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
    </dependencies>
```

### application.yml

``` yaml title="application.yml"
spring:
  application:
    name: consumer-app

server:
  port: 8081

cloud:
  aws:
    stack: false
    stack.auto: false
    region:
      static: us-east-1
      auto: false
    credentials:
      access-key: ANUJDEKAVADIYAEXAMPLE
      secret-key: 2QvM4/Tdmf38SkcD/qalvXO4EXAMPLEKEY
    queue:
      uri: http://localhost:4566
      name: sample-queue.fifo
```

`spring.application.name` para adicionar o nome da  aplicação, além do trace e span id.

`cloud.aws.*` são as configurações para se conectar a fila FIFO do SQS criado no Localstack.

### Configurando o cliente da fila SQS

``` java SQSConfig.java
@Configuration
public class SQSConfig {

    @Value("${cloud.aws.region.static}")
    private String region;

    @Value("${cloud.aws.credentials.access-key}")
    private String accessKeyId;

    @Value("${cloud.aws.credentials.secret-key}")
    private String secretAccessKey;

    @Value("${cloud.aws.queue.uri}")
    private String sqsUrl;

    @Bean
    @Primary
    public AmazonSQSAsync amazonSQSAsync() {
        return AmazonSQSAsyncClientBuilder.standard()
                .withEndpointConfiguration(new AwsClientBuilder.EndpointConfiguration(sqsUrl, region))
                .withCredentials(new AWSStaticCredentialsProvider(new BasicAWSCredentials(accessKeyId, secretAccessKey)))
                .build();
    }

}
```

### Consumindo uma mensagem com o Spring Cloud AWS Messaging

```java title="Consumer.java"
@Slf4j
@Service
public class Consumer {

    @SqsListener(value = "${cloud.aws.queue.name}",
            deletionPolicy = SqsMessageDeletionPolicy.ON_SUCCESS)
    public void receive(final Message message) {
        log.info(message.getPayload().toString());
    }

}
```
