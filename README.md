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

# Scenario 10

> **Before starting ensure that there are no other versions of the sandbox running**
> Run `docker-compose down -v` before starting

1. Start the scenario with `docker-compose up -d`
2. Wait for all services to be up and healthy `docker-compose ps`
3. Wait for the topics to be created. Check the control center(localhost:9021) to see if the topics - `domestic_orders` and `international_orders` are created

## Problem Statement

The client is able to produce to the `international_orders` topic but receives an error while producing and consuming from `domestic_orders` using the following commands -

```
kafka-console-producer --bootstrap-server kafka1:19092 --producer.config /opt/client/client.properties --topic <topic>

kafka-console-consumer --bootstrap-server kafka1:19092 --consumer.config /opt/client/client.properties --from-beginning --topic <topic> 
```

The client is using SASL/PLAIN over PLAINTEXT with the user `bob` and acks=1

The error message seen in the console producer and consumer fro `domestic_orders` - 

```
WARN [Producer clientId=console-producer] Connection to node 2 (kafka2/172.20.0.6:19093) could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
```

## Solution

After producing and consuming message 

![alt text](<./assets/Screenshot 2024-10-27 at 10.36.51 PM.png>)

The logs states that there is a connectivity issue between kafka2


After checking kafka2 server.properties we found that port are difined different for CLIENT protocal in listner and advertised.listners 

```
advertised.listeners=CLIENT://kafka2:29092,BROKER://kafka2:29093,TOKEN://kafka2:29094
```

The ports specified in listeners and advertised.listeners should generally be the same for each protocol. This ensures that clients can connect to the correct endpoint without confusion.

For example:

If you have 
```
listeners=CLIENT://:29092, then advertised.listeners=CLIENT://kafka2:29092 is appropriate.
```

If the ports differ, clients may not be able to connect properly, leading to connection issues. So, while the hostnames or IPs can differ, the ports should match for the same listener type.

![alt text](<./assets/Screenshot 2024-10-27 at 11.05.50 PM.png>)

## Result
![alt text](<./assets/Screenshot 2024-10-27 at 10.40.46 PM.png>)