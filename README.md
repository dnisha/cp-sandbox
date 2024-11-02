# kafka-jmx-grafana-docker

Docker-compose file for Confluent Kafka with configuration mounted as properties files. Brings up Kafka and components with JMX metrics exposed and visualized using Prometheus and Grafana. The environment simulates running Confluent Platform on VMs/Bare metal servers using properties files but using docker containers. The various branches in the repository contains troubleshooting scenarios for Kafka adminstrators to practice production-like issues.

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

# Scenario 18

> Before starting ensure that there are no other versions of the sandbox running Run `docker-compose down -v` before starting

1. Start the scenario with docker-compose up -d
2. Wait for all services to be up and healthy `docker-compose ps -a`
3. **Wait for the `init` container to exit. It should exit with code 0. This should take a while, depending on your system.** (You can run `watch docker-compose ps -a`)
4. Verify that the following topics are created through the CLI or control center
- debit_transactions
- credit_transactions
- loan_payments
- new_accounts
- fraud_detection
- business_requests
- account_audit
- clickstream
- international_transactions
- app_telemetry

## Problem Statement

The customer notices a red color for one of the panels in the control-center. Troubleshoot the cause for the red color in the control-center and propose a solution to fix the issue.

![img](control-center.png)


## Solution

Renew all ssl certificates , truststore and keystore refer scenario6.


Check memory and cpu utilization of brokers

```
docker stats kafka1
```

![alt text](<Screenshot 2024-11-02 at 11.19.42 PM.png>)

Use below command to check for kafka topic configuration :-
```
cat scripts/start.sh | base64 -d;
```

![alt text](<./assets/Screenshot 2024-11-02 at 5.16.56 PM.png>)

Adjust the partition accourding to the resources avilable

```
echo -n '#!/bin/bash

until nc -z kafka1 19092
do
        echo "Waiting for Kakfa to be ready"
        sleep 5
done

cat <<EOF >>/tmp/client.properties
sasl.mechanism=PLAIN

security.protocol=SASL_PLAINTEXT

sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="bob" \
  password="bob-secret";
EOF

kafka-topics --bootstrap-server kafka1:19092 --command-config /tmp/client.properties --create --topic debit_transactions \
  --if-not-exists --replica-assignment 2:1,2:1  \
  --config min.insync.replicas=1

kafka-topics --bootstrap-server kafka2:29092 --command-config /tmp/client.properties --create --topic credit_transactions \
  --if-not-exists --replica-assignment 2,2  \
  --config min.insync.replicas=1

kafka-topics --bootstrap-server kafka3:39092 --command-config /tmp/client.properties --create --topic loan_payments \
  --if-not-exists --replica-assignment 2:1,2:1 \
  --config min.insync.replicas=1

kafka-topics --bootstrap-server kafka1:19092 --command-config /tmp/client.properties --create --topic new_accounts \
  --if-not-exists --replica-assignment 2,2 \
  --config min.insync.replicas=1

kafka-topics --bootstrap-server kafka2:29092 --command-config /tmp/client.properties --create --topic fraud_detection \
  --if-not-exists --replica-assignment 2,2  \
  --config min.insync.replicas=1

kafka-topics --bootstrap-server kafka2:29092 --command-config /tmp/client.properties --create --topic business_requests \
  --if-not-exists --replica-assignment 2:1,2:1  \
  --config min.insync.replicas=1

kafka-topics --bootstrap-server kafka3:39092 --command-config /tmp/client.properties --create --topic account_audit \
  --if-not-exists --replica-assignment 2,2 \
  --config min.insync.replicas=1

kafka-topics --bootstrap-server kafka3:39092 --command-config /tmp/client.properties --create --topic clickstream \
  --if-not-exists --replica-assignment 2,2 \
  --config min.insync.replicas=1

# for x in {1..50}; do echo $x; done | kafka-console-producer --bootstrap-server kafka1:19092 --producer.config /tmp/client.properties --topic account_audit

kafka-topics --bootstrap-server kafka1:19092 --command-config /tmp/client.properties --create --topic international_transactions \
  --if-not-exists --replica-assignment 2,1 \
  --config min.insync.replicas=1

kafka-topics --bootstrap-server kafka2:29092 --command-config /tmp/client.properties --create --topic app_telemetry \
  --if-not-exists --replica-assignment 2,1 \
  --config min.insync.replicas=1' | base64 > scripts/start.sh
```

Use above command to configure the kafka-topic , i have decresed the partitions because of resource contraint


Mount the volume to local to all brokers.

```
./kafka1/data:/var/lib/kafka/data
```

![alt text](<./assets/Screenshot 2024-11-02 at 11.27.48 PM.png>)

Rampdown the producer performance according to system requirement.

![alt text](<./assets/Screenshot 2024-11-02 at 11.27.57 PM.png>)