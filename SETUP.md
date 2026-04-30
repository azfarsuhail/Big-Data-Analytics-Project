# Project Setup & Troubleshooting Guide: Big Data Analytics Pipeline

This guide documents the setup process, common pitfalls encountered with the Hive Metastore, and the final successful configuration of the Hadoop/Hive cluster.

---

## 1. Initial Infrastructure Setup
Start the containerized environment. Note that the `version` attribute in docker-compose is obsolete in modern Docker versions but can be safely ignored.
```bash
docker compose up -d
```

---

## 2. Troubleshooting the Hive Metastore
During initial setup, the Hive Server may fail to connect to the Metastore due to missing version information or incorrect database initialization.

### Symptoms
*   `MetaException(message:Version information not found in metastore.)`
*   `FAILED: SemanticException ... Unable to instantiate SessionHiveMetaStoreClient`
*   `Connection refused` when trying to connect via Beeline.

### The Fix: Manual Schema Initialization
If the automatic initialization fails, manually initialize the Postgres Metastore from within the `hive-server` container:
```bash
docker exec -it hive-server /opt/hive/bin/schematool -dbType postgres \
-url jdbc:postgresql://hive-metastore-postgresql:5432/metastore \
-driver org.postgresql.Driver -userName hive -passWord hive -initSchema
```

---

## 3. Data Ingestion to HDFS
Once the infrastructure is stable, create the directory structure in HDFS and upload the Olist E-Commerce datasets.

### Create Directories
```bash
docker exec -it namenode hdfs dfs -mkdir -p /user/hive/data/customers
docker exec -it namenode hdfs dfs -mkdir -p /user/hive/data/orders
docker exec -it namenode hdfs dfs -mkdir -p /user/hive/data/items
```

### Upload CSV Files
```bash
docker exec -it namenode hdfs dfs -put /shared_data/olist_customers_dataset.csv /user/hive/data/customers/
docker exec -it namenode hdfs dfs -put /shared_data/olist_orders_dataset.csv /user/hive/data/orders/
docker exec -it namenode hdfs dfs -put /shared_data/olist_order_items_dataset.csv /user/hive/data/items/
```

---

## 4. Hive Table Configuration
Connect to Hive via Beeline to map the HDFS files to relational tables.
```bash
docker exec -it hive-server beeline -u jdbc:hive2://hive-server:10000 -n root
```

### Table Mapping (Example: Customers)
```sql
CREATE EXTERNAL TABLE customers (
    customer_id STRING,
    customer_unique_id STRING,
    customer_zip_code_prefix INT,
    customer_city STRING,
    customer_state STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' 
STORED AS TEXTFILE LOCATION '/user/hive/data/customers/'
tblproperties ("skip.header.line.count"="1");
```

---

## 5. Execution & Optimization
To run complex joins on a local Docker environment without a full YARN cluster, enable **Local Mode**.

### Optimization Commands
```sql
SET mapreduce.framework.name=local;
SET hive.exec.mode.local.auto=true;
```

### The Analytical Join
```sql
SELECT 
    c.customer_state, 
    ROUND(SUM(i.price), 2) as total_revenue,
    COUNT(DISTINCT o.order_id) as total_orders
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items i ON o.order_id = i.order_id
GROUP BY c.customer_state
ORDER BY total_revenue DESC;
```

---

## Summary of Success
*   **Infrastructure:** 8/8 Containers Healthy.
*   **Data Volume:** ~42MB of Relational E-Commerce Data.
*   **Performance:** Complex 3-way Join completed in ~13 seconds.
