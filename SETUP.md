# Project Setup & Troubleshooting Guide: Big Data Analytics Pipeline

This guide documents the setup process, common Data Engineering pitfalls encountered (such as Metastore corruption and Hadoop memory limits), and the final configuration of a Highly Available (HA) Hadoop/Hive cluster.

---

## 1. Initial Infrastructure Setup
Start the containerized environment. Note that the `version` attribute in `docker-compose.yml` is obsolete in modern Docker versions but can be safely ignored.
```bash
docker compose up -d
```
*(Note: If you encounter Docker network `Unknown host` errors between containers, running `docker compose down` followed by `docker compose up -d` clears the tangled DNS cache).*

---

## 2. Troubleshooting the Hive Metastore
During initial setup, or after restarting the cluster, the Hive Server may crash or fail to connect to its PostgreSQL brain.

### Symptoms
*   `MetaException(message:Version information not found in metastore.)`
*   `FAILED: SemanticException ... Unable to instantiate SessionHiveMetaStoreClient`
*   `Connection refused` when trying to connect via Beeline.
*   `Unknown host` network errors.

### The Fix: The Clean Database Reset
Manually forcing schema creation often leaves the database in a "half-baked" state with missing version rows or broken table permissions. The cleanest fix is to completely wipe the temporary Postgres container and let the image regenerate the schema automatically.
```bash
# 1. Nuke the corrupted database
docker stop hive-metastore-postgresql
docker rm hive-metastore-postgresql

# 2. Spin up a fresh database
docker compose up -d hive-metastore-postgresql

# 3. Restart Hive to connect to the clean database
docker restart hive-metastore hive-server
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

### Upload CSV Files (Avoiding BlockGhosts)
Always use the `-f` (force overwrite) flag when uploading. If a DataNode was previously killed improperly, the NameNode will hold onto ghost metadata and throw `BlockMissingException` errors. The `-f` flag completely overwrites corrupted blocks.
```bash
docker exec -it namenode hdfs dfs -put -f /shared_data/olist_customers_dataset.csv /user/hive/data/customers/
docker exec -it namenode hdfs dfs -put -f /shared_data/olist_orders_dataset.csv /user/hive/data/orders/
docker exec -it namenode hdfs dfs -put -f /shared_data/olist_order_items_dataset.csv /user/hive/data/items/
```

---

## 4. Hive Table Configuration
Connect to Hive via Beeline to map the HDFS files to relational tables. *(Note: HiveServer2 takes 60-90 seconds to fully boot before accepting connections).*
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

## 5. Execution & Optimization (Fixing Memory Crashes)
When executing complex, multi-table joins, Hive attempts to perform "MapJoins" by loading smaller tables directly into local container RAM. Because Docker strictly limits Java Heap Memory, this often results in a total task failure (`Return code 2 from org.apache.hadoop.hive.ql.exec.mr.MapredLocalTask`).

### The Optimization: Force Reduce-Side Joins
Run these commands in your Beeline session before querying to disable MapJoins. This forces Hadoop to use traditional, highly stable Reduce-side joins across the cluster.
```sql
SET hive.auto.convert.join=false;
SET hive.ignore.mapjoin.hint=true;
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
*   **Infrastructure:** Distributed 2-Node Hadoop architecture successfully networked.
*   **Storage Resiliency:** HDFS `dfs_replication=2` verified via `fsck`; data successfully survives complete DataNode hardware failure.
*   **Data Volume:** ~42MB of Relational E-Commerce Data gracefully ingested.
*   **Processing:** Complex 3-way MapReduce joins complete reliably without memory overflow.
