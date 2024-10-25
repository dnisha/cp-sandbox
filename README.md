# kafka-jmx-grafana-docker

Docker-compose file for Confluent Kafka with configuration mounted as properties files. Brings up Kafka and components with JMX metrics exposed and visualized using Prometheus and Grafana

## Start

```
docker-compose up -d
```

## Usage

The docker-compose file brings up 3 node kafka cluster with security enabled. Each service in the compose file has its properties/configurations mounted as a volume from a directory with the same name as the service.

Check the kafka server.properties for more details about the Kafka setup.

### Health

Check if all components are up and running using

```bash
docker-compose ps -a
# Ensure there are no Exited services and all containers have the status `Up`
```


### Client

To use a kafka client, exec into the `kfkclient` container which contains the Kafka CLI and other tools necessary for troubleshooting Kafka. THe `kfkclient` container also contains a properties file mounted to `/opt/client`, which can be used to define the client properties for communicating with Kafka.

```
docker exec -it kfkclient bash
```

### Logs

Check the logs of the respective service by its container name.

```bash
docker logs <container_name> # docker logs kafka1
```

### Restarting services

To restart a particular service - 

```bash
docker-compose restart <service_name> # docker-compose restart kafka1
# OR
docker-compose up -d --force-recreate <service_name> # docker-compose up -d --force-recreate kafka1
```

# Scenario 2

> **Before starting ensure that there are no other versions of the sandbox running**
> Run `docker-compose down -v` before starting

1. Start the scenario with `docker-compose up -d`
2. Wait for all services to be up and healthy `docker-compose ps`
3. Wait for the topics to be created. Check the control center(localhost:9021) to see if the topic `europe_payments` is created

## Problem Statement

The client has created a new topic `europe_payments` but is unable to produce/consume from the topic from the host `kfkclient` using the user `kafkaclient1` using the following commands -

```
kafka-console-producer --bootstrap-server kafka1:19092 --producer.config /opt/client/client.properties --topic europe_payments

kafka-console-consumer --bootstrap-server kafka1:19092 --consumer.config /opt/client/client.properties --from-beginning --topic europe_payments
```

The client is using SASL/PLAIN over PLAINTEXT with the user `kafkaclient1`

The error message seen in the console producer and consumer for `europe_payments` - 

```
[2023-07-26 12:18:20,309] WARN [Producer clientId=console-producer] Error while fetching metadata with correlation id 4 : {europe_payments=TOPIC_AUTHORIZATION_FAILED} (org.apache.kafka.clients.NetworkClient)
[2023-07-26 12:18:20,409] ERROR [Producer clientId=console-producer] Topic authorization failed for topics [europe_payments] (org.apache.kafka.clients.Metadata)
[2023-07-26 12:18:20,411] ERROR Error when sending message to topic europe_payments with key: null, value: 6 bytes with error: (org.apache.kafka.clients.producer.internals.ErrorLoggingCallback)
org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics: [europe_payments]
```
## Solution
This document outlines the steps to generate certificates, keystores, truststores, and configure Kafka for the specified client. For the scripts and resources, refer to the repository: cp-sandbox-solutions.

#### Steps
 
## 1) Generate Certificates and Keystores

- Generate all necessary certificates, keystores, and truststores from script in the repo - https://github.com/dnisha/cp-sandbox-solutions

- Place them in their corresponding directories.

## 2) Configure Kafka Client

- Add the following line to the server.properties of kafka1, kafka2, kafka3:

```
broker.user=User:kafkaclient1
```

- Generate a certificate, keystore, and truststore for kafkaclient1 and place them in the client directory.

## 3) Create Configuration Files

- Inside the client folder, create the following properties files:

    - admin.properties

    - client.properties

- Use the admin.properties configuration file to apply ACL for the topic europe_payments.

```
sasl.mechanism=PLAIN

security.protocol=SASL_PLAINTEXT

sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="bob" \
  password="bob-secret";
```

- Use the client.properties to produce and consume messages.

```
security.protocol=SSL
ssl.truststore.location=/opt/client/kafka.kafkaclient1.truststore.jks
ssl.truststore.password=kafka-broker
ssl.keystore.location=/opt/client/kafka.kafkaclient1.keystore.jks
ssl.keystore.password=kafka-broker
ssl.key.password=kafka-broker
```

## 4) Start Docker Compose

- Run the following command to start the Docker containers:

```
docker-compose up -d
```

## 5) Set Access Control List (ACL)

- Log into the kfkclient container:

```
docker exec -it kfkclient bash
```

- Execute the following command to set the ACL:

```
/root/confluent-7.4.1/bin/kafka-acls --bootstrap-server kafka1:19092 --add --allow-principal User:kafkaclient1 --command-config /opt/client/admin.properties --allow-host kfkclient --operation WRITE --operation READ --topic europe_payments
```

## 6) Produce a Message

- Use the following command to produce a message:

```
/root/confluent-7.4.1/bin/kafka-console-producer --bootstrap-server kafka1:19093 --producer.config /opt/client/client.properties --topic europe_payments
```

## 7) Consume a Message

- Execute the following command to consume messages:

```
/root/confluent-7.4.1/bin/kafka-console-consumer --bootstrap-server kafka1:19093 --consumer.config /opt/client/client.properties --from-beginning --topic europe_payments
```

![alt text](<./assets/Screenshot 2024-10-25 at 1.32.43 PM.png>)