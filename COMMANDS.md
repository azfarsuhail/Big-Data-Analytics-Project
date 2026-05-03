# Big Data Analytics Pipeline: Olist E-Commerce Demo

This guide contains the sequential commands to demonstrate the full Big Data lifecycle—from infrastructure health checks to complex analytical joins using a Highly Available (HA) Hadoop and Hive cluster.

---

## Phase 1: Infrastructure & Storage Verification

### 1. Check Container Status
Verify that the NameNode, both DataNodes, Hive-Server, and the Postgres Metastore are actively running.
```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### 2. Inspect HDFS Storage
Ensure the Olist raw datasets are successfully ingested into the Hadoop Distributed File System.
```bash
docker exec -it namenode hdfs dfs -ls -R /user/hive/data/
```

### 3. Verify High Availability (HA) Replication
Prove that the cluster is fault-tolerant by checking the block distribution (look for `Live_repl=2` and two distinct target nodes).
```bash
docker exec -it namenode hdfs fsck /user/hive/data/customers/olist_customers_dataset.csv -files -blocks -locations
```

---

## Phase 2: Chaos Monkey (Fault Tolerance Test)

Demonstrate the robustness of the 2-node storage layer by simulating a sudden hardware failure.

### 1. Kill a DataNode
Simulate a server crash by stopping one of the data nodes.
```bash
docker stop datanode2
```

### 2. Prove Data Availability
Run a query to prove the NameNode seamlessly reroutes traffic to the surviving node without dropping data.
```bash
docker exec -it hive-server beeline -u jdbc:hive2://hive-server:10000 -n root -e "SELECT * FROM customers LIMIT 5;"
```

### 3. Resurrect the Node (Self-Healing)
Bring the node back online to let Hadoop auto-sync and heal the cluster.
```bash
docker start datanode2
```

---

## Phase 3: Hive Data Interaction

### 1. Connect to Hive Server
Enter the Hive environment using the Beeline interactive client.
```bash
docker exec -it hive-server beeline -u jdbc:hive2://hive-server:10000 -n root
```

### 2. Verify Schema & Metadata
Once inside Beeline, confirm the external tables are correctly mapped to HDFS.
```sql
-- List all tables in the default database
SHOW TABLES;

-- Inspect the 'orders' table structure
DESCRIBE orders;
```

---

## Phase 4: Analytical Queries (MapReduce)

### 1. Execute Multi-Table Revenue Join
This query joins the Customers, Orders, and Items tables to find the highest revenue-generating states in Brazil.
```sql
-- Step A: Set optimizations 
-- (Crucial: Disables MapJoins to prevent Java heap memory crashes in containerized environments, forcing a stable Reduce-side join)
SET hive.auto.convert.join=false;
SET hive.ignore.mapjoin.hint=true;

-- Step B: Run the State-wise Revenue Join
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

### 2. Bonus Insight: Top Premium Products
Identify the highest-priced products that have a significant sales volume.
```sql
SELECT 
    product_id, 
    ROUND(AVG(price), 2) as avg_price, 
    COUNT(*) as sales_volume
FROM order_items 
GROUP BY product_id 
HAVING sales_volume > 10
ORDER BY avg_price DESC 
LIMIT 5;
```