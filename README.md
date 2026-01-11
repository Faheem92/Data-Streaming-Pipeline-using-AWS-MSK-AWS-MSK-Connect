# Data-Streaming-Pipeline-using-AWS-MSK-AWS-MSK-Connect

End-to-end real-time data streaming pipeline using **Amazon MSK** and **MSK Connect**. It captures **Change Data Capture (CDC)** events from **Amazon RDS (SQL Server)** using Debezium and delivers them to **Amazon S3** using the Kafka S3 Sink Connector.
Implements **IAM-based Kafka authentication**, private networking using **VPC Gateway Endpoints**, and a production-style MSK Connect setup.

---

##  1. Objective

This project demonstrates a **real-time CDC pipeline** where:

1. SQL Server RDS generates CDC events
2. Debezium SQL Server Source Connector publishes them to Kafka topics
3. Kafka S3 Sink Connector writes the events to Amazon S3 (JSON)
4. MSK IAM Auth ensures secure access
5. All traffic stays inside VPC (S3 VPC Endpoint)

It acts as a **reference architecture** for building scalable, serverless, managed streaming pipelines using AWS-native components.

---

##  2. High-Level Architecture

```
                ┌─────────────────────────────┐
                │     Amazon RDS (SQL Server) │
                │   CDC Enabled Tables         │
                └───────────────┬─────────────┘
                                │ CDC Events
                                ▼
                    ┌──────────────────────┐
                    │ Debezium SQL Server  │
                    │  Source Connector    │
                    └─────────┬────────────┘
                              │ Publishes
                              ▼
                    ┌──────────────────────┐
                    │   Amazon MSK (Kafka) │
                    │  IAM Authentication  │
                    └─────────┬────────────┘
                              │ Consumes
                              ▼
                    ┌──────────────────────┐
                    │   S3 Sink Connector  │
                    └─────────┬────────────┘
                              │ Writes
                              ▼
                    ┌──────────────────────┐
                    │ Amazon S3 (JSON Data)│
                    └──────────────────────┘
```

AWS Services Used:

* Amazon RDS (SQL Server)
* Amazon MSK (Provisioned)
* MSK Connect
* Amazon S3
* EC2 (optional validation)
* IAM
* VPC Gateway Endpoint for S3

---

##  3. Repository Structure

```
.
├── IAM/
│   ├── MSKConnectorRole.json
│   └── EC2MSKRole.json
│
├── connector_config/
│   ├── S3sink.txt
│   └── SQLsource.txt
│
├── database/
│   └── CDCscript.sql
│
├── kafka/
│   └── Kafka Commands.txt
│
└── README.md
```

---

##  4. Implementation Steps

### **Step 1: Create SQL Server RDS Instance**

* Engine: SQL Server Standard
* Port: 1433
* Enable SQL Server Agent (required for CDC)

Security Group (PoC only): allow inbound 1433
(Restrict in production)

---

### **Step 2: Prepare Database & Enable CDC**

Run the SQL script:

```
database/CDCscript.sql
```

It includes:

* Create DB + tables
* Enable CDC at DB and table level
* Start SQL Agent
* Insert sample rows

---

### **Step 3: Create S3 Bucket + VPC Endpoint**

1. Create an S3 bucket
2. Create S3 **Gateway VPC Endpoint**
3. Attach to MSK + connector subnet route tables

**Why?**
Because MSK Connect runs inside private subnets and must access S3 *without public internet*.

---

### **Step 4: Create MSK Cluster**

Recommended settings:

* Type: **Provisioned**
* Brokers: 3
* Authentication: **IAM**
* Storage: 1 GB
* Security Group: allow traffic between MSK ↔ connectors ↔ EC2

---

### **Step 5: IAM Role for MSK Connect**

Use the file:

```
IAM/MSKConnectorRole.json
```

This role:

* Allows connector to read/write Kafka topics via IAM
* Allows full S3 access

Upload trust policy:

```
"kafkaconnect.amazonaws.com"
```

---

### **Step 6: Upload Connector Plugins**

Upload the following (JAR bundles):

| Plugin                     | Link                                                                                                                                     |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Debezium SQL Server Source | [https://www.confluent.io/hub/debezium/debezium-connector-sqlserver](https://www.confluent.io/hub/debezium/debezium-connector-sqlserver) |
| Kafka S3 Sink Connector    | [https://www.confluent.io/hub/confluentinc/kafka-connect-s3](https://www.confluent.io/hub/confluentinc/kafka-connect-s3)                 |

---

### **Step 7: Create Debezium Source Connector**

Configuration stored at:

```
connector_config/SQLsource.txt
```

Key options include:

```
connector.class=io.debezium.connector.sqlserver.SqlServerConnector
database.hostname=<rds-host>
database.user=<user>
database.password=<password>
database.names=myapp
topic.prefix=CDC_DEMO

schema.history.internal.kafka.topic=schemahistory.DEMO_APP
schema.history.internal.kafka.bootstrap.servers=<bootstrap>
```

IAM Kafka auth is applied to both producer and consumer sides.

---

### **Step 8: (Optional) Create EC2 Instance for Testing**

Attach:

```
IAM/EC2MSKRole.json
```

Install Kafka and test:

* Create topics
* Consume messages
* Validate connectivity

Scripts provided in:

```
kafka/Kafka Commands.txt
```

---

### **Step 9: Install Kafka Client & Validate Topics**

Install Java + Kafka:

```
sudo yum install java-11 -y
wget https://archive.apache.org/dist/kafka/3.6.0/kafka_2.13-3.6.0.tgz
```

Create `client.properties` using IAM auth:

```
security.protocol=SASL_SSL
sasl.mechanism=AWS_MSK_IAM
```

Test topic creation, publishing, consuming.

---

### **Step 10: Create S3 Sink Connector**

Configuration stored in:

```
connector_config/S3sink.txt
```

Example:

```
connector.class=io.confluent.connect.s3.S3SinkConnector
topics=<topic>
s3.bucket.name=<bucket-name>
format.class=io.confluent.connect.s3.format.json.JsonFormat
flush.size=1
```

---

### **Step 11: Validate Data in S3**

Check S3 folder:

* JSON files generated
* Contains INSERT, UPDATE, DELETE CDC events
* Ensure they match source DB

---

##  5. Troubleshooting Guide

### **Kafka IAM Authentication Errors**

* Ensure EC2/connector role has:

  ```
  kafka-cluster:Connect
  kafka-cluster:ReadData
  kafka-cluster:WriteData
  ```
* Check `client.properties` file

### **CDC Not Capturing Changes**

* SQL Server Agent must be running
* Table must have a primary key
* CDC must be enabled on both DB and specific table

### **MSK Connect Connector Not Starting**

* Ensure plugin uploaded as ZIP (correct structure)
* Check CloudWatch logs → connector logs
* Check network connectivity to RDS/MSK/S3

### **No Data in S3**

* Verify topic exists
* Ensure IAM policy includes `s3:PutObject`
* Topic name must match MSK Connect config

---

##  6. Key Learnings

* IAM authentication removes need for keystores/truststores
* MSK Connect avoids managing Kafka Connect clusters
* CDC provides real-time change propagation
* VPC endpoints allow secure private data movement
* Decoupled architecture allows multiple downstream consumers

---

##  7. Future Enhancements

* Add AWS Glue Schema Registry
* Add hourly/daily partitioning strategy
* Enable encryption with CMK keys
* Add monitoring using Prometheus + Grafana
* Add error handling DLQs (Dead Letter Topics)
