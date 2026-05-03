# Big Data Analytics Pipeline: Brazilian E-Commerce Analysis

This project is a containerized Hadoop and Hive environment for analyzing the Olist Brazilian E-Commerce dataset with SQL-on-Hadoop queries. It loads CSV data into HDFS, maps it to Hive external tables, and runs analytical joins through Beeline.

## Overview

The stack is defined in [docker-compose.yml](docker-compose.yml) and includes:

* HDFS with a dedicated NameNode and two DataNodes.
* HiveServer2 for query execution.
* A separate Hive metastore service.
* PostgreSQL as the persistent Hive metastore database.
* Beeline as the primary client for SQL queries.

## Dataset

The repository currently ships with these files in [shared_data](shared_data):

* [olist_customers_dataset.csv](shared_data/olist_customers_dataset.csv)
* [olist_orders_dataset.csv](shared_data/olist_orders_dataset.csv)
* [olist_order_items_dataset.csv](shared_data/olist_order_items_dataset.csv)

These are staged into HDFS under `/user/hive/data/` and mapped to Hive tables such as `customers`, `orders`, and `order_items`.

## Project Files

* [SETUP.md](SETUP.md) covers environment startup, metastore initialization, and HDFS ingestion.
* [COMMANDS.md](COMMANDS.md) contains the step-by-step demo commands for verification and analysis.
* [BEELINEvsHIVE.md](BEELINEvsHIVE.md) explains why Beeline is used instead of the legacy Hive CLI.

## Demo Videos

* [Fault Tolerance](Fault%20Tolerance.mp4) shows a DataNode being stopped while Beeline is still used to run SQL operations.
* [Demo Hadoop](Demo%20Hadoop.mp4) shows the Hadoop stack in action with MapReduce, Hive, and Beeline performing a large join.

## Prerequisites

* Docker and Docker Compose
* Windows 10/11 with WSL2 support
* At least 8 GB of RAM recommended

## Start the Cluster

```bash
docker compose up -d
```

Check that the services are running:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

## Load Data into HDFS

Create the HDFS folders:

```bash
docker exec -it namenode hdfs dfs -mkdir -p /user/hive/data/customers
docker exec -it namenode hdfs dfs -mkdir -p /user/hive/data/orders
docker exec -it namenode hdfs dfs -mkdir -p /user/hive/data/items
```

Upload the CSV files:

```bash
docker exec -it namenode hdfs dfs -put /shared_data/olist_customers_dataset.csv /user/hive/data/customers/
docker exec -it namenode hdfs dfs -put /shared_data/olist_orders_dataset.csv /user/hive/data/orders/
docker exec -it namenode hdfs dfs -put /shared_data/olist_order_items_dataset.csv /user/hive/data/items/
```

Verify the files:

```bash
docker exec -it namenode hdfs dfs -ls -R /user/hive/data/
```

## Connect to Hive

Use Beeline to connect to HiveServer2:

```bash
docker exec -it hive-server beeline -u jdbc:hive2://hive-server:10000 -n root
```

If the metastore schema has not been created yet, initialize it from the Hive server container:

```bash
docker exec -it hive-server /opt/hive/bin/schematool -dbType postgres -url jdbc:postgresql://hive-metastore-postgresql:5432/metastore -driver org.postgresql.Driver -userName hive -passWord hive -initSchema
```

## Example Hive Table

```sql
CREATE EXTERNAL TABLE customers (
	customer_id STRING,
	customer_unique_id STRING,
	customer_zip_code_prefix INT,
	customer_city STRING,
	customer_state STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/hive/data/customers/'
TBLPROPERTIES ("skip.header.line.count"="1");
```

## Example Analytics Query

For local execution inside Docker, enable Hive local mode first:

```sql
SET mapreduce.framework.name=local;
SET hive.exec.mode.local.auto=true;
```

Then run the revenue-by-state join:

```sql
SELECT
	c.customer_state,
	ROUND(SUM(i.price), 2) AS total_revenue,
	COUNT(DISTINCT o.order_id) AS total_orders
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items i ON o.order_id = i.order_id
GROUP BY c.customer_state
ORDER BY total_revenue DESC;
```

## Useful Notes

* The Hive metastore service uses PostgreSQL at `hive-metastore-postgresql:5432`.
* The default HDFS replication is set to 2 in [hadoop.env](hadoop.env), which matches the two DataNodes in the compose file.
* The old Hive CLI is not the recommended client for this project; use Beeline instead.

## Troubleshooting

* If Beeline cannot connect, confirm that the containers are healthy and that HiveServer2 has finished starting.
* If you see `MetaException(message:Version information not found in metastore.)`, run the schema initialization command above.
* If HDFS files are missing, re-check the `hdfs dfs -put` paths and confirm that `./shared_data` is mounted into the Namenode container.

## Result Snapshot

The demo query identifies the highest-revenue states in Brazil. In the current sample output, São Paulo leads the analysis, followed by Rio de Janeiro and Minas Gerais.

---

Developer: Azfar Suhail
Role: Data Engineering / Final Year Project 2026
