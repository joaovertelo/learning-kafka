# Learning Streaming ETL with ksqlDB
## Side Note: Under construction...

## Project Architecture

![architecture](https://github.com/paulodnobre/learning-kafka/blob/main/streaming-etl-with-ksqldb/images/1.png)

## Step 1. Prepare Dev Environment

```sql
CREATE TABLE departments
(
	depid int primary key,
	depname text
	);
```

```sql
CREATE TABLE employees
(
	empid int primary key,
	empfname text,
	empmname text,
	emplname text,
	empemail text,
    lastupdate date,
	depid int
	);
```

## Step 2. Create the Source Connectors

### Using JDBC Source Connector
```sql
CREATE SOURCE CONNECTOR `postgres-source-departments-jdbc` WITH (
    "connector.class"='io.confluent.connect.jdbc.JdbcSourceConnector',
    "connection.url"='jdbc:postgresql://postgres-source:5432/data_sources',
	"connection.user"='postgres',
	"connection.password"='Postgres2020!',
    "mode"='incrementing',
	"incrementing.column.name"='depid',
    "topic.prefix"='topic-',
    "table.whitelist"='departments',
    "key"='depid',
	"key.converter" = 'org.apache.kafka.connect.storage.StringConverter',
	"value.converter" = 'io.confluent.connect.avro.AvroConverter',
	"value.converter.schema.registry.url"='http://schema-registry:8081');
  ```
```sql
CREATE SOURCE CONNECTOR `postgres-source-employees-jdbc` WITH (
    "connector.class"='io.confluent.connect.jdbc.JdbcSourceConnector',
    "connection.url"='jdbc:postgresql://postgres-source:5432/data_sources',
	"connection.user"='postgres',
	"connection.password"='Postgres2020!',
    "mode"='incrementing',
	"incrementing.column.name"='empid',
    "topic.prefix"='topic-',
    "table.whitelist"='employees',
    "key"='empid',
	"key.converter" = 'org.apache.kafka.connect.storage.StringConverter',
	"value.converter" = 'io.confluent.connect.avro.AvroConverter',
	"value.converter.schema.registry.url"='http://schema-registry:8081');
   ```
   
  ### Using Debezium
  
  ```sql
  CREATE SOURCE CONNECTOR `postgres-source-departments` WITH (
  'connector.class'               = 'io.debezium.connector.postgresql.PostgresConnector',
  'database.dbname'               = 'data_sources',
  'database.hostname'             = 'postgres-source',
  'database.password'             = 'Postgres2020!',
  'database.port'                 = '5432',
  'database.server.name'          = 'postgres-source',
  'database.user'                 = 'postgres',
  'plugin.name'                   = 'pgoutput',
  'snapshot.mode'                 = 'always',
  'slot.name'					            = 'slot_dep',
  'table.whitelist'               = 'public.departments',
  'transforms'                    = 'extractKey,extractValue',
  'transforms.extractKey.field'   = 'depid',
  'transforms.extractKey.type'    = 'org.apache.kafka.connect.transforms.ExtractField$Key',
  'transforms.extractValue.field' = 'after',
  'transforms.extractValue.type'  = 'org.apache.kafka.connect.transforms.ExtractField$Value'
);
```

```sql
CREATE SOURCE CONNECTOR `postgres-source-employees` WITH (
  'connector.class'               = 'io.debezium.connector.postgresql.PostgresConnector',
  'database.dbname'               = 'data_sources',
  'database.hostname'             = 'postgres-source',
  'database.password'             = 'Postgres2020!',
  'database.port'                 = '5432',
  'database.server.name'          = 'postgres-source',
  'database.user'                 = 'postgres',
  'plugin.name'                   = 'pgoutput',
  'snapshot.mode'                 = 'always',
  'slot.name'                     = 'slot_emp',
  'table.whitelist'               = 'public.employees',
  'transforms'                    = 'extractKey,extractValue',
  'transforms.extractKey.field'   = 'empid',
  'transforms.extractKey.type'    = 'org.apache.kafka.connect.transforms.ExtractField$Key',
  'transforms.extractValue.field' = 'after',
  'transforms.extractValue.type'  = 'org.apache.kafka.connect.transforms.ExtractField$Value'
);
```

### Optional: If you are using Debezium, check the replication slots that were created

```sql
SELECT * FROM pg_replication_slots;
```

### Discussion: Differences between JDBC and Debezium source connectors
### Discussion 2: Differences between creating a topic in Kafka and directly from the Connector

## Step 3. Create the Sink Connectors to the Landing Zone

### From JDBC generated topics

```sql
CREATE SINK CONNECTOR `postgres-sink-departments-raw` WITH (
    "connector.class"='io.confluent.connect.jdbc.JdbcSinkConnector',
    "connection.url"='jdbc:postgresql://postgres-sink:5432/data_raw',
	"connection.user"='postgres',
	"connection.password"='Postgres2020!',
    "mode"='incrementing',
	"topics"='topic-departments',
	"auto.create"='true',
	"auto.evolve"='true',
	"insert.mode"='upsert',
	"pk.mode"='kafka',
	"key.converter" = 'org.apache.kafka.connect.storage.StringConverter',
	"value.converter" = 'io.confluent.connect.avro.AvroConverter',
	"value.converter.schema.registry.url"='http://schema-registry:8081');
```

```sql
CREATE SINK CONNECTOR `postgres-sink-employees-raw` WITH (
    "connector.class"='io.confluent.connect.jdbc.JdbcSinkConnector',
    "connection.url"='jdbc:postgresql://postgres-sink:5432/data_raw',
	"connection.user"='postgres',
	"connection.password"='Postgres2020!',
    "mode"='incrementing',
	"topics"='topic-employees',
	"auto.create"='true',
	"auto.evolve"='true',
	"insert.mode"='upsert',
	"pk.mode"='kafka',
	"key.converter" = 'org.apache.kafka.connect.storage.StringConverter',
	"value.converter" = 'io.confluent.connect.avro.AvroConverter',
	"value.converter.schema.registry.url"='http://schema-registry:8081');
```
  
 ### From Debezium generated topics

```sql
 CREATE SINK CONNECTOR `postgres-sink-raw-departments` WITH (
    "connector.class"					= 'io.confluent.connect.jdbc.JdbcSinkConnector',
    "connection.url"					= 'jdbc:postgresql://postgres-sink:5432/data_raw',
	"connection.user"					= 'postgres',
	"connection.password"				= 'Postgres2020!',
	"topics"							= 'postgres-source.public.departments',
	"auto.create"						= 'true',
	"auto.evolve"						= 'true',
	"insert.mode"						= 'upsert',
	"pk.mode"							= 'kafka',
	"transforms"						= 'unwrap',
    "transforms.unwrap.type"			= 'io.debezium.transforms.ExtractNewRecordState',
    "transforms.unwrap.drop.tombstones" = 'false');
 ```
 
```sql
CREATE SINK CONNECTOR `postgres-sink-raw-employees` WITH (
    "connector.class"					= 'io.confluent.connect.jdbc.JdbcSinkConnector',
    "connection.url"					= 'jdbc:postgresql://postgres-sink:5432/data_raw',
	"connection.user"					= 'postgres',
	"connection.password"				= 'Postgres2020!',
	"topics"							= 'postgres-source.public.employees',
	"auto.create"						= 'true',
	"auto.evolve"						= 'true',
	"insert.mode"						= 'upsert',
	"pk.mode"							= 'kafka',
	"transforms"						= 'unwrap',
    "transforms.unwrap.type"			= 'io.debezium.transforms.ExtractNewRecordState',
    "transforms.unwrap.drop.tombstones" = 'false');
 ```
### Discussion: Troubleshooting Sink Connector

## Step 4. Create the Streams

### Discussion: Understand Streams vs Tables

### STREAM-DEPARTMENTS
```sql
CREATE STREAM stream_departments
WITH (KAFKA_TOPIC='topic-departments', VALUE_FORMAT='AVRO');
```

### STREAM-EMPLOYEES
```sql
CREATE STREAM stream_employees
WITH (KAFKA_TOPIC='topic-employees', VALUE_FORMAT='AVRO');
```

```sql
SELECT * from STREAM_EMPLOYEES EMIT CHANGES;
```

```sql
SELECT * from STREAM_DEPARTMENTS EMIT CHANGES;
```

## Step 5. Create the Derived Stream

```sql
CREATE STREAM EMPLOYEES_DEPARTMENTS AS
SELECT	se.empid,
		se.empfname, 
		se.empemail,
		se.emplname,
		se.empmname,
		se.lastupdate,
		se.depid,
		sd.depid,
		sd.depname
FROM  STREAM_EMPLOYEES se
JOIN  STREAM_DEPARTMENTS sd 
WITHIN 7 DAYS 
ON se.depid = sd.depid;
```

### Discussion: Understand keys on streams
### Discussion 2: Performing Joins on the Source Connector or on the Streams?

## Step 6. Sink the EMPLOYEES_DEPARTMENTS Topic

### From JDBC generated data streams
```sql
CREATE SINK CONNECTOR `postgres-sink-empdep` WITH (
    "connector.class"					= 'io.confluent.connect.jdbc.JdbcSinkConnector',
    "connection.url"					= 'jdbc:postgresql://postgres-sink:5432/data_curated',
	"connection.user"					= 'postgres',
	"connection.password"				= 'Postgres2020!',
	"topics"							= 'EMPLOYEES_DEPARTMENTS',
	"auto.create"						= 'true',
	"auto.evolve"						= 'true',
	"insert.mode"						= 'upsert',
	"pk.mode"							= 'kafka');
```

### From Debezium generated data streams
```sql
CREATE SINK CONNECTOR `postgres-sink-empdep` WITH (
    "connector.class"					= 'io.confluent.connect.jdbc.JdbcSinkConnector',
    "connection.url"					= 'jdbc:postgresql://postgres-sink:5432/data_curated',
	"connection.user"					= 'postgres',
	"connection.password"				= 'Postgres2020!',
	"topics"							= 'EMPLOYEES_DEPARTMENTS',
	"auto.create"						= 'true',
	"auto.evolve"						= 'true',
	"insert.mode"						= 'upsert',
	"pk.mode"							= 'kafka',
	"transforms"						= 'unwrap',
    "transforms.unwrap.type"			= 'io.debezium.transforms.ExtractNewRecordState',
    "transforms.unwrap.drop.tombstones" = 'false');
 ```

## Step 7. Let's test!

### Discussion: How updates can work?
### Discussion 2: How to handle the date in an integer format?
