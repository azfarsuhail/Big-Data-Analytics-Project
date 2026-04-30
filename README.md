# Big Data Analytics Pipeline: Brazilian E-Commerce Analysis

A containerized Big Data infrastructure leveraging the **Hadoop Ecosystem** to process and analyze real-world e-commerce data. This project demonstrates the ingestion, storage, and relational modeling of over 100,000 records using a **Star Schema** architecture.

## 🏗️ Architecture
The pipeline is built using a multi-container Docker orchestration:
*   **HDFS (Hadoop 3.2.1):** Distributed storage layer with a dedicated NameNode and DataNode.
*   **Apache Hive (2.3.2):** Data warehousing layer for SQL-on-Hadoop abstraction.
*   **PostgreSQL:** External persistent Metastore for Hive metadata management.
*   **Beeline:** Client interface for executing high-performance analytical queries.

## 📊 Dataset
The project utilizes the **Olist Brazilian E-Commerce Dataset**, consisting of:
*   **Customers:** Demographics and geographic locations.
*   **Orders:** Transactional status and timestamps.
*   **Order Items:** Product-level pricing and freight details.

## 🚀 Key Features
*   **Relational Mapping:** Conversion of raw, unstructured CSV data into optimized Hive tables.
*   **Local MapReduce Execution:** Custom configuration to enable complex distributed joins in a resource-constrained environment.
*   **State-wise Revenue Analytics:** A 3-way table join identifying top-performing geographic markets.

## 📂 Project Structure
*   `setup.md`: Technical guide for cluster initialization and Metastore troubleshooting.
*   `demo.md`: Step-by-step commands for reproducing the analytical results.
*   `shared_data/`: Local staging area for raw CSV datasets.

## 📈 Sample Insight: Revenue by State
By executing distributed joins, the pipeline identifies regional economic concentration:

| State | Total Revenue (BRL) | Total Orders |
| :--- | :--- | :--- |
| **SP (São Paulo)** | **5,202,955.05** | **41,375** |
| RJ (Rio de Janeiro) | 1,824,092.67 | 12,762 |
| MG (Minas Gerais) | 1,585,308.03 | 11,544 |

## 🛠️ Requirements
*   Docker & Docker Compose
*   Minimum 8GB RAM (Optimized via Hive Local Mode)
*   Windows 10/11 with WSL2 Support

---
**Developer:** Azfar Suhail  
**Role:** Data Engineering / Final Year Project 2026