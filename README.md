# Subscriber daily activity streaming APP
Using KSQL DB technology, implement stream processing of data from cassandra CDR storage to subscriber activity table.<br>
In this table, there should be zeros even if there is no data for a subscriber for a day. <br>
The output should be a stream of objects for each subscriber on each date.<br>
Each object should include the subscriber's duration of calls for the day, and the number of SMS messages for the day.<br>
If a subscriber had no calls or messages on a given day, the resulting table should still display zeros.<br>

# 1. Cassandra: Init CDR storage

### 1.1 Init key space
```
CREATE KEYSPACE IF NOT EXISTS billing
WITH replication = {
'class': 'SimpleStrategy',
'replication_factor': 1
};
```
### 1.1 Init CDR table
```
USE billing;

CREATE TABLE IF NOT EXISTS billing_cdr_logs (
  id timeuuid,
  subscriptionId BIGINT,
  created TIMESTAMP,
  amount DOUBLE,
  unit TEXT,
  type TEXT,
  PRIMARY KEY (subscriptionId, created)
) WITH CLUSTERING ORDER BY (created asc);
```

### 1.2 Insert some data
```
INSERT INTO billing_cdr_logs (id, subscriptionId, created, amount, unit, type) VALUES (now(), 100, toTimestamp(now()),  100, 'minutes', 'CALL');
INSERT INTO billing_cdr_logs (id, subscriptionId, created, amount, unit, type) VALUES (now(), 101, toTimestamp(now()),  200, 'minutes', 'CALL');
INSERT INTO billing_cdr_logs (id, subscriptionId, created, amount, unit, type) VALUES (now(), 102, toTimestamp(now()),  300, 'minutes', 'CALL');
INSERT INTO billing_cdr_logs (id, subscriptionId, created, amount, unit, type) VALUES (now(), 103, toTimestamp(now()),  400, 'minutes', 'CALL');
INSERT INTO billing_cdr_logs (id, subscriptionId, created, amount, unit, type) VALUES (now(), 104, toTimestamp(now()),  500, 'minutes', 'CALL');
```

# 2. Kafka connect: Setup cassandra as event source

### 2.1 Create topic for raw CDRs
```
kafka-topics --bootstrap-server localhost:9092 --create --partitions 1 --replication-factor 1 --topic cdr_raw_events
```

### 2.2 Setup kafka connect with cassandra as event source
```
CREATE SOURCE CONNECTOR CASSANDRA_CDR_SOURCE WITH(
  "connector.class"='com.datamountaineer.streamreactor.connect.cassandra.source.CassandraSourceConnector',
  "connect.cassandra.contact.points"='localhost',
  "connect.cassandra.port"=9042,
  "connect.cassandra.username"='cassandra',
  "connect.cassandra.password"='cassandra',
  "connect.cassandra.consistency.level"='LOCAL_ONE',
  "connect.cassandra.key.space"='billing',
  "connect.cassandra.import.mode"='incremental',
  "connect.cassandra.kcql"='INSERT INTO cdr_raw_events SELECT * FROM billing_cdr_logs PK id INCREMENTALMODE=TIMEUUID'
  );
```

### 2.3 Check source connector

```
show connectors;
print cdr_raw_events from beginning;
```

# 3. Ksql: Build CDR aggregation APP

### 3.1 Create stream of raw casandra CDR events

```
create stream str_cdr_raw_events with (kafka_topic='cdr_raw_events', value_format='avro');
```

### 3.2 Create main stream for CDR events aggregations

```
kafka-topics --bootstrap-server localhost:9092 --create --partitions 1 --replication-factor 1 --topic billing_cdr_logs

create stream str_cdr_logs with (kafka_topic='billing_cdr_logs', value_format='avro');

ksql-datagen schema=configuration/cdr.avro format=avro topic=billing_cdr_logs key=subscriptionId msgRate=1 iterations=10
```

### 3.3 Merge cassandra CDR stream with main
```
SET 'auto.offset.reset'='earliest';
insert into str_cdr_logs select SUBSCRIPTIONID, FORMAT_TIMESTAMP(created, 'yyyy-MM-dd') as EVENTDATE, AMOUNT, UNIT, TYPE from STR_CDR_RAW_EVENTS;
```

### 3.4 Create table with aggregated results
```
CREATE TABLE CDR_AGGREGATED WITH (KEY_FORMAT='AVRO', VALUE_FORMAT='AVRO')
as SELECT subscriptionId, SUM (amount) as total_amount, eventDate, type
FROM str_cdr_logs 
GROUP BY subscriptionId, eventDate, type;
```

### 3.5 Test results

```
 select * from CDR_AGGREGATED;
 print CDR_AGGREGATED;
```

# 4. Kafka connect: Setup sink result to postgres

### 4.1 Init table with results
```
DROP TABLE IF EXISTS public."CDR_AGGREGATED";
CREATE TABLE public."CDR_AGGREGATED" (
    "SUBSCRIPTIONID" int8 NOT NULL,
	"TYPE" text NOT NULL,
	"EVENTDATE" text NOT NULL,
	"TOTAL_AMOUNT" float8 NULL,
	CONSTRAINT "CDR_AGGREGATED_pkey" PRIMARY KEY ("EVENTDATE", "SUBSCRIPTIONID", "TYPE")
);
```

### 4.2 Setup kafka connect with postgres as aggregation storage
```
CREATE SOURCE CONNECTOR JBDC_CDR_SINK WITH(
    "connector.class"='io.confluent.connect.jdbc.JdbcSinkConnector',
    "connection.url"='jdbc:postgresql://localhost:5432/kafka_sink',
    "connection.user"='postgres',
    "connection.password"='postgres',
    "topics"='CDR_AGGREGATED',
    "insert.mode"='upsert',
    "batch.size"='100',
    "table.name.format"='CDR_AGGREGATED',
    "pk.mode"='record_key',
    "pk.fields"='SUBSCRIPTIONID,EVENTDATE,TYPE',
    "auto.create"='true',
    "auto.evolve"='true',
    "max.retries"='10',
    "retry.backoff.ms"=3000,
    "mode"='bulk',
    "key.converter"='io.confluent.connect.avro.AvroConverter',
    "key.converter.schemas.enable"='true',
    "key.converter.schema.registry.url"='http://localhost:8081',
    "value.converter"='io.confluent.connect.avro.AvroConverter',
    "value.converter.schemas.enable"='true',
    "value.converter.schema.registry.url"='http://localhost:8081'
  );
```

### 4.3 End-to-end test
```
ksql-datagen schema=configuration/cdr.avro format=avro topic=billing_cdr_logs key=subscriptionId msgRate=2 iterations=1000

INSERT INTO billing_cdr_logs (id, subscriptionId, created, amount, unit, type) VALUES (now(), 100, toTimestamp(now()),  99999, 'minutes', 'CALL');
```

# 5. Solving the problem with user without daily activity

### 5.1 Create topic for missing events
```
kafka-topics --bootstrap-server localhost:9092 --create --partitions 1 --replication-factor 1 --topic cdr_missing_events
```

### 5.2 Kafka connect: Setup missing event source using REST API
```
curl --location 'localhost:8083/connectors' \
--header 'Content-Type: application/json' \
--data '{
    "name": "JBDC_MISSING_EVENT_SOURCE",
    "config": {
        "tasks.max": "1",
        "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
        "connection.url": "jdbc:postgresql://localhost:5432/kafka_sink",
        "connection.user": "postgres",
        "connection.password": "postgres",
        "mode": "bulk",
        "poll.interval.ms": 30000,
        "topic.prefix": "cdr_missing_events",
        "query": "select * from (select s.id as SUBSCRIPTIONID, now()::date::varchar as EVENTDATE, 0.0::float as AMOUNT, '\''minutes'\'' as UNIT, '\''CALL'\'' as TYPE from \"SUBSCRIPTION\" s left join \"CDR_AGGREGATED\" ca on s.id =ca.\"SUBSCRIPTIONID\" where ca.\"SUBSCRIPTIONID\" is null union select s.id as SUBSCRIPTIONID, now()::date::varchar as EVENTDATE, 0.0::float as AMOUNT, '\''quantity'\'' as UNIT, '\''SMS'\'' as TYPE from \"SUBSCRIPTION\" s  left join \"CDR_AGGREGATED\" ca on s.id =ca.\"SUBSCRIPTIONID\" where ca.\"SUBSCRIPTIONID\" is null) as foo",
        "transforms": "createKey,extractInt",
        "transforms.createKey.type": "org.apache.kafka.connect.transforms.ValueToKey",
        "transforms.createKey.fields": "subscriptionid",
        "transforms.extractInt.type": "org.apache.kafka.connect.transforms.ExtractField$Key",
        "transforms.extractInt.field": "subscriptionid"
    }
}'
```

### 5.3 SQL for missing events in details
It should be wrapped with fake SELECT since JdbcSinkConnector require to have 
WHERE clause that can be correctly appended to the query.

```
select *
from
	(
	select
		s.id as SUBSCRIPTIONID,
		now()::date::varchar as EVENTDATE,
		0.0::float as AMOUNT,
		'minutes' as UNIT,
		'CALL' as type
	from
		"SUBSCRIPTION" s
	left join "CDR_AGGREGATED" ca on s.id = ca."SUBSCRIPTIONID"
	where
		ca."SUBSCRIPTIONID" is null
    union
    select
        s.id as SUBSCRIPTIONID,
        now()::date::varchar as EVENTDATE,
        0.0::float as AMOUNT,
        'quantity' as UNIT,
        'SMS' as type
    from
        "SUBSCRIPTION" s
    left join "CDR_AGGREGATED" ca on s.id = ca."SUBSCRIPTIONID"
    where
        ca."SUBSCRIPTIONID" is null
	) 
	as foo
```

### 5.3 Create stream for missing events

```
create stream str_cdr_missing_events with (kafka_topic='cdr_missing_events', value_format='avro') ;
```

### 5.4 Merge stream of missing events with main CDR stream

```
SET 'auto.offset.reset'='earliest';
insert into str_cdr_logs select * from str_cdr_missing_events;
```

# 6. Useful links
* https://docs.confluent.io/platform/6.2/quickstart/ce-quickstart.html
* https://docs.confluent.io/confluent-cli/current/command-reference/connect/plugin/confluent_connect_plugin_install.html
* https://docs.confluent.io/kafka-connectors/jdbc/current/source-connector/overview.html
* https://docs.lenses.io/5.2/connectors/sources/cassandrasourceconnector/
* https://docs.ksqldb.io/en/latest/developer-guide/ksqldb-reference/
* https://docs.confluent.io/kafka/design/log_compaction.html

# 7. Useful info

### Show KSQL configuration
```
ksql> LIST PROPERTIES;

 Property               | Effective Value 
--------------------------------------------
 . . .
 ksql.schema.registry.url              | SERVER           | http://localhost:8081
 ksql.streams.auto.offset.reset        | SERVER           | latest
 ksql.extension.dir                    | SERVER           | ext                 
 . . .
 ```
it will refer to ${CONFLUENT_HOME}/ext

### Location of kafka connect plugins

${CONFLUENT_HOME}/share/java
