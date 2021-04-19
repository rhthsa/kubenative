# Enhance your microservice app on Kubernetes with Event Driven Architecture
<!-- TOC -->

- [Enhance your microservice app on Kubernetes with Event Driven Architecture](#enhance-your-microservice-app-on-kubernetes-with-event-driven-architecture)
  - [Presentation](#presentation)
  - [Micro service with Kafka & Quarkus](#micro-service-with-kafka--quarkus)
  - [Change Data Capture (CDC) with AMQ Streams & Debezium](#change-data-capture-cdc-with-amq-streams--debezium)

<!-- /TOC -->

## Presentation

Presentation ([eda.pptx](presentation/eda.pptx))


## Micro service with Kafka & Quarkus

- Use demo from https://quarkus.io/guides/kafka
- Step Demo
  - start local kafka with strimzi, show https://strimzi.io/
  - open source code from quarkus-quickstarts/kafka-quickstart
  - show pom.xml --> quarkus-smallrye-reactive-messaging-kafka
  - show src/main/java/org/acme/kafka/PriceGenerator.java put to kafka , producer
  - show src/main/java/org/acme/kafka/PriceConverter.java get from kafka , consumer
  - show src/main/java/org/acme/kafka/PriceResource.java sse/server send event to webui
  - show application.properties
  - show src/main/resources/META-INF/resources/prices.html display price
  - show docker-compose.yaml in kafka-quickstart folder
  - start docker desktop
  - run docker-compose up --> start strimzi/kafka in local laptop
  ```bash
  $ docker-compose up
  ```
  - run with live code
  ```bash
  $ mvn quarkus:dev
  ```
  - test wit http://localhost:8080/prices.html on browser

- Optional: use kafdrop to view kafka topic
  - download code from https://github.com/obsidiandynamics/kafdrop
  - build kafdrop jar
  ```bash
  $ mvn clean package
  ```
  - run kafdrop with jar file
  ```bash
  $ java --add-opens=java.base/sun.nio.ch=ALL-UNNAMED -jar target/kafdrop-3.28.0-SNAPSHOT.jar
  ```
  - open browser to url: http://localhost:9000/

## Change Data Capture (CDC) with AMQ Streams & Debezium

- Request demo from RHPDS, DIL Streaming Event-driven Workshop (T), Module 2: Change Data Capture & Apache Kafka
- Prepare Step for Demo
  - open code ready workspace
  - open terminal
  - run 
  ```bash
  oc login
  oc project user1
  ```
  - open openshift dev console
  - create kafka connect
  - ceate kafkaconnect cdc connector
  - download order.csv from instruction
  - create kafka moon cluster in user1 project
  - create mirror maker in user1 project
  - create kafka bridge
  - create kafka bridge route
  ```bash
  oc expose svc
  ```
- Demo Step
  - login to OpenShift Developer Console
  - review php application (project shared app earth)
  - review microsoft sql server (project shared db earth)
  - review kafka (project shared kafka earth)
  - review project user1 (kafka cluster moon, kafka connect, kafka connector, mirror maker, kafka bridge)
  - open php web enterprise system --> http://www-shared-app-earth.appcluster/#user1
  - open terminal ms sql database, check data in database
  ```bash
  $ /opt/mssql-tools/bin/sqlcmd -S mssql-server-linux -U sa -P Password! -d InternationalDB -Q "select top 5 * from dbo.Orders where OrderUser='user1'"
  ```
  - import data, used order.csv --> load file
  - check data in database again
  ```bash
  /opt/mssql-tools/bin/sqlcmd -S mssql-server-linux -U sa -P Password! -d InternationalDB -Q "select top 5 * from dbo.Orders where OrderUser='user1'"
  ```
  - check found topic in kafka earth, project shared kafka earth --> search topic user1 (with name, CRD = kafkatopic )
  - go to topology of kafka earth --> kafka stateful set --> kafka 0 --> terminal, check data in topic
  ```bash
  bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic user1.earth.dbo.Orders --from-beginning
  ```
  - show mirror maker in project user1
  - search found topic user1 in project user1 --> in kafka moon
  - test kafka http bridge --> in codeready terminal
  - create consumer
  ```bash
  $ cat << EOF | curl -X POST http://kafka-bridge-user1.apps.cluster-dqtrj.dqtrj.sandbox242.opentlc.com/consumers/user1-http-group -H 'content-type: application/vnd.kafka.v2+json' -d @-
  {
      "name": "user1",
      "format": "json",
      "auto.offset.reset": "earliest",
      "enable.auto.commit": false,
      "fetch.min.bytes": 1024,
      "consumer.request.timeout.ms": 30000
  }
  EOF
  ```
  - subscription topic
  ```bash
  $ curl -X POST http://kafka-bridge-user1.apps.cluster-dqtrj.dqtrj.sandbox242.opentlc.com/consumers/user1-http-group/instances/user1/subscription -H 'content-type: application/vnd.kafka.v2+json' -d '{"topics": ["user1.earth.dbo.Orders"]}'
  ```
  - consume data, run 2 time, 1st call return empty
  ```bash
  $ curl http://kafka-bridge-user1.apps.cluster-dqtrj.dqtrj.sandbox242.opentlc.com/consumers/user1-http-group/instances/user1/records -H 'accept: application/vnd.kafka.json.v2+json'
  ```




