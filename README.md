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

# Scenario 9

> **Before starting ensure that there are no other versions of the sandbox running**
> Run `docker-compose down -v` before starting

1. Start the scenario with `docker-compose up -d`
2. Wait for all services to be up

## Problem Statement

The client just performed a lift and shift on their entire platform to different machines. The brokers and several other components are down.

The brokers have the following error log - 

```
java.lang.RuntimeException: Received a fatal error while waiting for all of the authorizer futures to be completed.
	at kafka.server.KafkaServer.startup(KafkaServer.scala:950)
	at kafka.Kafka$.main(Kafka.scala:114)
	at kafka.Kafka.main(Kafka.scala)
Caused by: java.util.concurrent.CompletionException: org.apache.kafka.common.errors.SslAuthenticationException: SSL handshake failed
	at java.base/java.util.concurrent.CompletableFuture.encodeRelay(CompletableFuture.java:367)
	at java.base/java.util.concurrent.CompletableFuture.completeRelay(CompletableFuture.java:376)
	at java.base/java.util.concurrent.CompletableFuture$AnyOf.tryFire(CompletableFuture.java:1663)
	at java.base/java.util.concurrent.CompletableFuture.postComplete(CompletableFuture.java:506)
	at java.base/java.util.concurrent.CompletableFuture.completeExceptionally(CompletableFuture.java:2088)
	at io.confluent.security.auth.provider.ConfluentProvider.lambda$null$10(ConfluentProvider.java:543)
	at java.base/java.util.concurrent.CompletableFuture.uniExceptionally(CompletableFuture.java:986)
	at java.base/java.util.concurrent.CompletableFuture$UniExceptionally.tryFire(CompletableFuture.java:970)
	at java.base/java.util.concurrent.CompletableFuture.postComplete(CompletableFuture.java:506)
	at java.base/java.util.concurrent.CompletableFuture.completeExceptionally(CompletableFuture.java:2088)
	at io.confluent.security.store.kafka.clients.KafkaReader.lambda$start$1(KafkaReader.java:102)
	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:829)
```
## Solution

# Troubleshooting step

Checking the logs of all the brokers from below command 

```
docker logs kafka1
```
It is showing ssl handshake is failing.

![alt text](<./assets/Screenshot 2024-10-26 at 10.09.30 PM.png>)

Lets check the keystore and truststore and check for validity and CN 
```
keytool -list -v -keystore kafka.server.keystore.jks
```

![alt text](<./assets/Screenshot 2024-10-26 at 10.08.15 PM.png>)

# Certificate is expired Lets generate new CA , keystores and truststore for all brokers .


## Create Your Own CA Generate a new CA key and certificate:

```
openssl req -new -x509 -days 365 -keyout ca-key -out ca-cert -subj "/C=IN/ST=Karnataka/L=Bengaluru/O=Platformatory Labs Pvt Ltd/OU=kafka/CN=Root-CA" -passout pass:confluent;
```

## Create Truststore and Import Root CA Create a truststore for Kafka brokers and import the CA certificate into it:
```
keytool -keystore kafka.server.truststore.jks -storepass kafka-broker -import -alias ca-root -file ca-cert -noprompt
```

## Create Keystore Generate a new keystore and private key for a Kafka broker:
```
keytool -keystore kafka.server.keystore.jks -storepass kafka-broker -alias kafka3 -validity 365 -keyalg RSA  -genkeypair -keypass kafka-broker -dname "CN=kafka3,OU=kafka,O=Platformatory Labs Pvt Ltd,L=Bengaluru,ST=Karnataka,C=IN"
```

## Create Certificate Signing Request (CSR) Generate a CSR for the broker:
```
keytool -alias kafka3 -keystore kafka.server.keystore.jks -certreq -file cert-file -storepass kafka-broker -keypass kafka-broker
```
## Sign the Certificate Using the CA Sign the CSR with the CA certificate:
```
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:confluent -extensions SAN -extfile <(printf "\n[SAN]\nsubjectAltName=DNS:kafka3,DNS:localhost");
```
## Import CA Certificate into Keystore Import the CA certificate into the broker's keystore:
```
keytool -importcert -keystore kafka.server.keystore.jks -alias ca-root -file ca-cert -storepass kafka-broker -keypass kafka-broker -noprompt
```
## Import Signed Certificate into Keystore Import the signed certificate into the broker's keystore:
```
keytool -keystore kafka.server.keystore.jks -alias kafka3 -import -file cert-signed -storepass kafka-broker -keypass kafka-broker -noprompt
```

Note - Repeate above steps for all brokers and place truststure to there corresponding folders.