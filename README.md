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

# Scenario 1

> **Before starting ensure that there are no other versions of the sandbox running**
> Run `docker-compose down -v` before starting

1. Start the scenario with `docker-compose up -d`
2. Wait for all services to be up and healthy `docker-compose ps`

## Problem Statement

The client notices that the connect cluster is down on control-center(localhost:9021). This is after an security audit removed some users from the super users list.

Troubleshoot and provide a secure solution for the client.

## Solution1

Below are the steps I took to debug the solution:

Start Docker Compose

Run the following command to start the services:

```
docker-compose up -d
```

![alt text](<./assets/Screenshot 2024-10-23 at 12.01.49 PM.png>)

2) Check Kafka 1 Logs

View the logs for Kafka 1 to identify issues:

```
docker logs kafka1
```

The logs clearly indicate that the SSL handshake is failing. To resolve this, we need to generate the certificates and place them in the corresponding folders for each broker (i.e., kafka1, kafka2, kafka3).

Use the script I created to generate the Certificate Authority (CA), Certificate Signing Request (CSR), truststore, and keystore for the corresponding brokers.

Link - https://github.com/dnisha/cp-sandbox-solutions

![alt text](<./assets/Screenshot 2024-10-23 at 12.37.06 PM.png>)



3) Check for All Containers That Go Down
Monitor the status of all containers:

I observed that the Schema Registry goes down because not all Kafka brokers have started up yet.

![alt text](<./assets/Screenshot 2024-10-23 at 12.02.02 PM.png>)


4) Check Connect Logs
View the logs for the Connect service:

```
docker logs connect
```

![alt text](<./assets/Screenshot 2024-10-23 at 12.12.18 PM.png>)

The logs indicate that the Connect service is not authorized to access the topic. After checking server.properties, I noticed that the user connectAdmin is missing from the super.users list. We need to add this user to all Kafka brokers.

5) Restart the Compose File

Bring up the services again:

```
docker-compose up -d
```

![alt text](<./assets/Screenshot 2024-10-23 at 12.19.36 PM.png>)

![alt text](<./assets/Screenshot 2024-10-23 at 12.20.00 PM.png>)
