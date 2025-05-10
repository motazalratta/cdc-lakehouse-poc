cdc-lakehouse-poc
This porject is a poc which demostraze the  data architure to load data from rdbms data base to lakehouse and use it in BI
please be awre that this project doesn't consider the fail toilcance and security ascpets .. the goal of it is to have some hands on for some new open sorces tools

the data flow is 
postgres db as cdc via debizum -> kafka-> flinksql -> hdfs (iceberg) via hive metastore (that use mysql as rdbms) -> then query the data via Treno -> and use treno in metabease


archicutre diagram


the dockercompse which include the following contains :
broker (Confluent Kafka Broker): A Kafka broker that handles Kafka messages and serves as the central component in a Kafka-based messaging system. It’s part of Confluent's Kafka stack.

schema-registry (Confluent Schema Registry): Manages and validates schemas for Kafka messages. This ensures that the data structure of messages is consistent across producers and consumers.

connect (Confluent Kafka Connect): Provides Kafka Connect for integrating Kafka with external systems (like databases, file systems, etc.). It uses connectors for data pipelines.

control-center (Confluent Control Center): A UI and monitoring platform for managing and monitoring Kafka clusters, connectors, and streams. It provides insights and control over Kafka components.

flink-sql-client (Flink SQL Client for Kafka): A SQL client interface for Flink, allowing you to execute SQL queries against data in Kafka topics and other connected systems.

flink-jobmanager (Flink Job Manager): Coordinates and manages the execution of Flink jobs. It tracks the job’s state and supervises task execution.

flink-taskmanager (Flink Task Manager): Executes the actual computation tasks of Flink jobs. Multiple task managers can be scaled for distributed processing.

namenode (Hadoop NameNode): The master server in the Hadoop HDFS (Hadoop Distributed File System). It manages file system metadata, like file locations and directories.

datanode (Hadoop DataNode): A worker node in HDFS that stores actual data blocks. Multiple datanodes are used for redundancy and scalability in HDFS.

postgres (PostgreSQL Database for Debezium): A PostgreSQL instance used as a source for the Debezium connector to capture changes and send them to Kafka.

hive-metastore-db (Hive Metastore Database): A MySQL instance used by Apache Hive to store metadata for the Hive data warehouse system.

hive-metastore-init (Hive Metastore Initialization): Initializes the Hive Metastore schema in the MySQL database. It runs schema initialization scripts for setting up the database.

hive-metastore (Hive Metastore Service): The service that manages Hive metadata, enabling query engines (like Trino) to access metadata of Hive tables and partitions.

trino-coordinator (Trino Coordinator): The main node in a Trino (formerly Presto) cluster. It coordinates queries across worker nodes and handles query execution planning.

trino-worker (Trino Worker): Executes distributed queries in a Trino cluster. Workers process portions of a query assigned by the coordinator.

metabase (Metabase): A business intelligence tool that connects to databases (including Kafka) to visualize and analyze data through a web interface.

web UI for the porjects:
1. kafka http://localhost:9021/
2. Flink http://localhost:9081/
3. HDFS http://localhost:9870/
4. Trino http://localhost:8080/


stratup steps 
1. run the docker-composer
docker-compser up -d
2. create the tables in postgres databases
connect to the postgtes dtabase via any clinet and create two tables under the defulst database postges and schema public 


CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    order_date TIMESTAMP WITH TIME ZONE NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL,
    order_status VARCHAR(50) NOT NULL
);

CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE,
    registration_date TIMESTAMP WITH TIME ZONE NOT NULL
);

INSERT INTO orders (customer_id, order_date, total_amount, order_status) VALUES
(1, NOW(), 125.50, 'PROCESSING'),
(2, NOW() - INTERVAL '1 day', 78.99, 'SHIPPED'),
(1, NOW() - INTERVAL '2 hours', 200.00, 'PROCESSING'),
(3, NOW() - INTERVAL '3 days', 45.20, 'DELIVERED');

INSERT INTO customers (first_name, last_name, email, registration_date) VALUES
('Alice', 'Smith', 'alice.smith@example.com', NOW() - INTERVAL '7 days'),
('Bob', 'Johnson', 'bob.johnson@example.com', NOW() - INTERVAL '10 days'),
('Charlie', 'Brown', 'charlie.brown@example.com', NOW() - INTERVAL '5 days');

enable the REPLICA IDENTITY FULL for the tables by applyiing the following commands

ALTER TABLE orders REPLICA IDENTITY FULL;

ALTER TABLE customers REPLICA IDENTITY FULL;

3. create the kafka connection 
    open http://localhost:9021/clusters/MkU3OEVBNTcwNTJENDM2Qk/management/connect/connect-default/connectors
    add new connector 
    select the file under [config/kafka/debezium_postgres.json](config/kafka/debezium_postgres.json)
    verfy step: open the topics you should see two new topics debezium.public.customers and debezium.public.orders


4. create the flink tables and do small tansformation by  join the orders and customers streams based on customer_id and enrich the order data with customer names
    1. connect to flink sql by executing ```docker compose exec -it flink-sql-client bash -c "sql-client.sh"``` 
    2. create the stream raw tables for order and customer topics
    
CREATE TABLE OrdersRaw (
    order_id BIGINT,
    customer_id INTEGER,
    order_date TIMESTAMP_LTZ(3),
    total_amount DECIMAL(10, 2),
    order_status VARCHAR,
    PRIMARY KEY (order_id) NOT ENFORCED -- Use NOT ENFORCED for Kafka-based tables
) WITH (
    'connector' = 'kafka',
    'topic' = 'debezium.public.orders',
    'properties.bootstrap.servers' = 'broker:29092',
    'properties.client.dns.lookup' = 'use_all_dns_ips',
    'scan.startup.mode' = 'earliest-offset',
    'key.format' = 'json',
    'key.json.ignore-parse-errors' = 'true',
    'key.fields' = 'order_id',
    'value.format' = 'debezium-json',
    'value.debezium-json.ignore-parse-errors' = 'true',
    'value.debezium-json.timestamp-format.standard' = 'ISO-8601'
);

CREATE TABLE CustomersRaw (
    customer_id INTEGER,
    first_name VARCHAR,
    last_name VARCHAR,
    email VARCHAR,
    registration_date TIMESTAMP_LTZ(3),
    PRIMARY KEY (customer_id) NOT ENFORCED -- Use NOT ENFORCED
) WITH (
    'connector' = 'kafka',
    'topic' = 'debezium.public.customers',
    'properties.bootstrap.servers' = 'broker:29092',
    'properties.client.dns.lookup' = 'use_all_dns_ips',
    'scan.startup.mode' = 'earliest-offset',
    'key.format' = 'json',
    'key.json.ignore-parse-errors' = 'true',
    'key.fields' = 'customer_id',
    'value.format' = 'debezium-json',
    'value.debezium-json.ignore-parse-errors' = 'true',
    'value.debezium-json.timestamp-format.standard' = 'ISO-8601'
);

3. verfication step: ```select * from OrdersRaw;``` and ```select * from CustomersRaw;```
4. create hive catalog and database
CREATE CATALOG hive_catalog WITH (
  'type'='iceberg',
  'catalog-type'='hive',
  'uri'='thrift://hive-metastore:9083',
  'clients'='5',
  'property-version'='1',
  'warehouse'='hdfs://namenode:9000/warehouse'
);
USE CATALOG hive_catalog;
create database iceberg_db;
use iceberg_db;
5. create three iceberg tables
