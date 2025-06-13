# Explore Confluent Kafka With its Advance Features Like
- Schema Registry
- KSQL
- Kafka Stream 
- Control Center
- Kafka UI
- Kafka CLI

# Prerequisite
- Knowledge Of Docker
- Docker Desktop
- Docker Compose
- Docker Networking

## Execute Docker Compose file `docker-compose.yml`
```
docker compose up -d
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