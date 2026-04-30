# Big Data Analytics Pipeline: Olist E-Commerce Demo

This guide contains the sequential commands to demonstrate the full Big Data lifecycle—from infrastructure health checks to complex analytical joins using Hadoop and Hive.

---

## Phase 1: Infrastructure & Storage Verification

### 1. Check Container Status
Verify that the Namenode, Datanode, Hive-Server, and Postgres Metastore are running.
```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### 2. Inspect HDFS Storage
Ensure the Olist raw datasets are successfully ingested into the Hadoop Distributed File System.
```bash
docker exec -it namenode hdfs dfs -ls -R /user/hive/data/
```

---

## Phase 2: Hive Data Interaction

### 3. Connect to Hive Server
Enter the Hive environment using the Beeline client.
```bash
docker exec -it hive-server beeline -u jdbc:hive2://hive-server:10000 -n root
```

### 4. Verify Schema & Metadata
Once inside Beeline, confirm the tables exist and the schema is correctly mapped.
```sql
-- List all tables in the default database
SHOW TABLES;

-- Inspect the 'Fact' table structure
DESCRIBE order_items;

-- Preview the first 5 records of the Customers table
SELECT * FROM customers LIMIT 5;
```

---

## Phase 3: Analytical Queries (MapReduce)

### 5. Execute Multi-Table Revenue Join
This query joins the Customers, Orders, and Items tables to find the highest revenue-generating states in Brazil.
```sql
-- Step A: Set optimizations (Crucial for a smooth demo in containerized environments)
SET mapreduce.framework.name=local;
SET hive.exec.mode.local.auto=true;

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

### 6. Bonus Insight: Top Premium Products
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