# Secure Kubernetes Native Application by OpenID with Keycloak
<!-- TOC -->

- [Secure Kubernetes Native Application by OpenID with Keycloak](#secure-kubernetes-native-application-by-openid-with-keycloak)
  - [Part I](#part-i)
  - [prepare](#prepare)
  - [demo](#demo)

<!-- /TOC -->


Part I
------------------------------------------------------------------------------
start local kafka with strimzi
1. show https://strimzi.io/

2. open /Users/ckongman/work/workspace/quarkus-quickstarts/kafka-quickstart
3. code . --> open vscode of kafka-quickstart

show pom.xml --> quarkus-smallrye-reactive-messaging-kafka

show src/main/java/org/acme/kafka/PriceGenerator.java put to kafka , producer

show src/main/java/org/acme/kafka/PriceConverter.java get from kafka , consumer

show src/main/java/org/acme/kafka/PriceResource.java sse/server send event to webui

show application.properties

show src/main/resources/META-INF/resources/prices.html display price

4. show docker-compose.yaml in kafka-quickstart folder
4.5 start docker desktop
5. docker-compose up --> start strimzi/kafka in local laptop
-1 broker
-1 zookeeper

open kafdrop --> workspace/kafdrop
run java --add-opens=java.base/sun.nio.ch=ALL-UNNAMED -jar target/kafdrop-3.28.0-SNAPSHOT.jar
http://localhost:9000/

run with live code --> mvn quarkus:dev

test with http://localhost:8080/prices.html on browser



---------------------------------------------------------------------------------
---------------------------------------------------------------------------------
Part II
CDC--> DIL Lab 2 user:user1/password:openshift
prepare
-----------------
open code ready workspace
open terminal
oc login
oc project user1
open openshift dev console
create kafka connect
ceate kafkaconnect cdc connector
order.csv in download folder
create kafka moon cluster in user 1 project
create mirror maker in user1 project
create kafka bridge, expose route 

demo
-----------------
appcluster=xxx
project shared app earth --> php
project shared db earth --> ms sql server
project shared kafka earth --> kafka
project user1 CDC, kafka cluster moon, mirror maker, kafka bridge

open php web enterprise system

http://www-shared-app-earth.appcluster/#user1

open terminal ms sql database 
run: /opt/mssql-tools/bin/sqlcmd -S mssql-server-linux -U sa -P Password! -d InternationalDB -Q "select top 5 * from dbo.Orders where OrderUser='user1'"

php web imported --> order.csv --> load file

run again: /opt/mssql-tools/bin/sqlcmd -S mssql-server-linux -U sa -P Password! -d InternationalDB -Q "select top 5 * from dbo.Orders where OrderUser='user1'"

project shared kafka earth --> search topic user1 (with name, kafkatopic)

go to topology of kafka --> kafka stateful set --> kafka 0 --> terminal
run: bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic user1.earth.dbo.Orders --from-beginning

show mirror maker --> project user1
project project user1 --> search topic user1 (with name, kafkatopic)

test kafka http bridge:-->codeready
-->create consumer-->
cat << EOF | curl -X POST http://kafka-bridge-user1.apps.cluster-dqtrj.dqtrj.sandbox242.opentlc.com/consumers/user1-http-group -H 'content-type: application/vnd.kafka.v2+json' -d @-
{
    "name": "user1",
    "format": "json",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": false,
    "fetch.min.bytes": 1024,
    "consumer.request.timeout.ms": 30000
}
EOF

--> subscription topic -->
curl -X POST http://kafka-bridge-user1.apps.cluster-dqtrj.dqtrj.sandbox242.opentlc.com/consumers/user1-http-group/instances/user1/subscription -H 'content-type: application/vnd.kafka.v2+json' -d '{"topics": ["user1.earth.dbo.Orders"]}'

--> consume data --> may be 2 time, 1st call return empty
curl http://kafka-bridge-user1.apps.cluster-dqtrj.dqtrj.sandbox242.opentlc.com/consumers/user1-http-group/instances/user1/records -H 'accept: application/vnd.kafka.json.v2+json'

