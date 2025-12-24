# 📦 Change Data Capture (CDC) Pipeline

This project sets up a **CDC pipeline** using **PostgreSQL → Kafka (Debezium) → MongoDB & Elasticsearch**.

* PostgreSQL changes (INSERT / UPDATE / DELETE)
* Debezium watches those changes
* Kafka sends the changes
* MongoDB and Elasticsearch receive and store them

---

## 🧱 Architecture

```
PostgreSQL
   ↓ (Debezium)
Kafka Topic: cdc.public.products
   ↓                ↓
MongoDB         Elasticsearch
```

---

## 🔧 Prerequisites

Make sure you already have:

* PostgreSQL (logical replication enabled)
* Apache Kafka
* Kafka Connect
* Debezium PostgreSQL Connector
* MongoDB Kafka Sink Connector
* Elasticsearch Sink Connector

---

## 1️⃣ PostgreSQL Source Connector (Debezium)

**Purpose:**

* Captures changes from `public.products` table
* Publishes events to Kafka topic: `cdc.public.products`

### ➕ Create / Update Connector

**PUT** `http://localhost:8083/connectors/pg-source-connector-01/config`

```json
{
  "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
  "database.hostname": "cdc-postgres",
  "database.port": "5432",
  "database.user": "postgres",
  "database.password": "postgres",
  "database.dbname": "postgres_db",
  "database.server.name": "cdc-postgres",
  "topic.prefix": "cdc",
  "plugin.name": "pgoutput",
  "slot.name": "debezium_slot",
  "publication.autocreate.mode": "filtered",
  "table.include.list": "public.products",

  "key.converter": "org.apache.kafka.connect.json.JsonConverter",
  "value.converter": "org.apache.kafka.connect.json.JsonConverter",
  "key.converter.schemas.enable": "false",
  "value.converter.schemas.enable": "false",

  "transforms": "unwrap,extractKey",
  "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
  "transforms.unwrap.drop.tombstones": "false",
  "transforms.unwrap.delete.handling.mode": "rewrite",

  "transforms.extractKey.type": "org.apache.kafka.connect.transforms.ExtractField$Key",
  "transforms.extractKey.field": "id"
}
```

### 📝 Notes

* `ExtractNewRecordState` removes Debezium envelope
* Primary key is extracted from `id`
* DELETE events are rewritten instead of dropped

---

## 2️⃣ MongoDB Sink Connector

**Purpose:**

* Consumes Kafka topic `cdc.public.products`
* Writes data into MongoDB collection `products`

### ➕ Create Connector

**POST** `http://localhost:8083/connectors`

```json
{
  "name": "mongo-sink-connector-01",
  "config": {
    "connector.class": "com.mongodb.kafka.connect.MongoSinkConnector",
    "topics": "cdc.public.products",
    "connection.uri": "mongodb://mongodb:password123@cdc-mongodb:27017/?authSource=admin",
    "database": "postgres_db",
    "collection": "products",

    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "value.converter.schemas.enable": "false",

    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "true",
    "transforms.unwrap.delete.handling.mode": "rewrite"
  }
}
```

### 📝 Notes

* Automatically syncs PostgreSQL data into MongoDB
* DELETE events remove documents
* Schema-less JSON storage

---

## 3️⃣ Elasticsearch Sink Connector

**Purpose:**

* Indexes product data into Elasticsearch
* Enables fast searching & analytics

### ➕ Create / Update Connector

**PUT** `http://localhost:8083/connectors/elastic-sink-connector/config`

```json
{
  "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
  "tasks.max": "1",
  "topics": "cdc.public.products",
  "connection.url": "http://elasticsearch:9200",
  "type.name": "_doc",

  "key.ignore": "false",
  "schema.ignore": "true",
  "behavior.on.null.values": "DELETE",

  "key.converter": "org.apache.kafka.connect.json.JsonConverter",
  "key.converter.schemas.enable": "false",
  "value.converter": "org.apache.kafka.connect.json.JsonConverter",
  "value.converter.schemas.enable": "false"
}
```

### 📝 Notes

* Uses Kafka message key as Elasticsearch document ID
* DELETE events remove documents from index
* Optimized for schema‑less JSON data


Happy Coding! 🚀
