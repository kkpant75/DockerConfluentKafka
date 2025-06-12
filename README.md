# Explore Confluent Kafka With its Advance Features Like
- Schema Registry
- KSQL
- Kafka Stream 
- Control Center
- Kafka UI
- Kafka CLI

# Docker Compose File
```
version: '2.5'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    hostname: kafka
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1

  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    hostname: schema-registry
    container_name: schema-registry
    depends_on:
      - kafka
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: PLAINTEXT://kafka:9092

  ksqldb-server:
    image: confluentinc/ksqldb-server:0.29.0
    hostname: ksqldb-server
    container_name: ksqldb-server
    depends_on:
      - kafka
      - schema-registry
    ports:
      - "8088:8088"
    environment:
      KSQL_CONFIG_DIR: "/etc/ksql"
      KSQL_BOOTSTRAP_SERVERS: kafka:9092
      KSQL_HOST_NAME: ksqldb-server
      KSQL_LISTENERS: http://0.0.0.0:8088
      KSQL_KSQL_SCHEMA_REGISTRY_URL: http://schema-registry:8081

  ksqldb-cli:
    image: confluentinc/ksqldb-cli:0.29.0
    container_name: ksqldb-cli
    depends_on:
      - ksqldb-server
    entrypoint: /bin/sh
    tty: true

  control-center:
    image: confluentinc/cp-enterprise-control-center:7.5.0
    hostname: control-center
    container_name: control-center
    depends_on:
      - kafka
      - schema-registry
      - ksqldb-server
    ports:
      - "9021:9021"
    environment:
      CONTROL_CENTER_BOOTSTRAP_SERVERS: kafka:9092
      CONTROL_CENTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
      CONTROL_CENTER_KSQL_KSQLDB1_URL: http://ksqldb-server:8088
      CONTROL_CENTER_KSQL_KSQLDB1_ADVERTISED_URL: http://localhost:8088
      CONTROL_CENTER_CONNECT_CLUSTER: ""
      CONTROL_CENTER_REPLICATION_FACTOR: 1
      CONFLUENT_METRICS_ENABLE: "false"
      PORT: 9021
  
  kafka-cli:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka-cli
    depends_on:
    - kafka
    - schema-registry
    entrypoint: /bin/sh
    tty: true

  kafka-ui:
    image: provectuslabs/kafka-ui
    container_name: kafka-ui
    ports:
      - "8080:8080"
    environment:
        KAFKA_CLUSTERS_0_NAME: local
        KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
        KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081
```
# Kafka Avro Example: "users" Topic

This guide walks you through setting up a Kafka topic named `users` with Avro serialization and schema management using Confluent Schema Registry.

---

## 1. Create Kafka Topic

Create the `users` topic using Kafka CLI:

```
docker exec -it kafka   kafka-topics --create   --topic users   --bootstrap-server localhost:9092   --partitions 1  --replication-factor 1
```

## 2. Register Avro Schema
Define your Avro schema for the `users` topic:
```
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id", "type": "int"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": "string"}
  ]
}
```

### Register it with the Schema Registry:
```

curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \\
  --data '{"schema": "{\\"type\\":\\"record\\",\\"name\\":\\"User\\",\\"fields\\":[{\\"name\\":\\"id\\",\\"type\\":\\"int\\"},{\\"name\\":\\"name\\",\\"type\\":\\"string\\"},{\\"name\\":\\"email\\",\\"type\\":\\"string\\"}]}" }'   http://localhost:8081/subjects/users-value/versions
  ```
##  3. Produce Data (Avro Format) - Connect `schema-registry` Container
Produce messages using kafka-avro-console-producer:
```


docker exec -it schema-registry   

kafka-avro-console-producer   --broker-list kafka:9092   --topic users --property value.schema='{
    "type":"record",
    "name":"User",
    "fields":[
      {"name":"id","type":"int"},
      {"name":"name","type":"string"},
      {"name":"email","type":"string"}
    ]
  }' 
  --property schema.registry.url=http://schema-registry:8081
```
### Sample data to type manually:
```
json
{"id": 1, "name": "Alice", "email": "alice@example.com"}
{"id": 2, "name": "Bob", "email": "bob@example.com"}
```

## 4. Consume Avro Messages - Open An Another Terminal `schema-registry` Container
Read Avro-encoded messages from the topic:
```
docker exec -it schema-registry 

kafka-avro-console-consumer --bootstrap-server kafka:9092 --topic users   --from-beginning   --property schema.registry.url=http://schema-registry:8081
```

## 5. Create Stream with ksqlDB (Optional) -Connect `ksqldb-cli` Container
Enter the ksqlDB CLI:
```


docker exec -it ksqldb-cli 

ksql http://ksqldb-server:8088
```
### Create a stream from the topic:
```
CREATE STREAM users_stream (
  id INT,
  name STRING,
  email STRING
) WITH (
  KAFKA_TOPIC='users',
  VALUE_FORMAT='AVRO'
);
```
### Query data:
```
SELECT * FROM users_stream EMIT CHANGES;
```
## 6. View Raw Topic Data
To see raw topic data (not Avro-decoded):
```

docker exec -it kafka   kafka-console-consumer   --bootstrap-server localhost:9092   --topic users   --from-beginning
```

