# 🚀 MySQL to Amazon RDS Migration using AWS DMS

![AWS](https://img.shields.io/badge/AWS-DMS-orange?style=for-the-badge&logo=amazonaws)
![MySQL](https://img.shields.io/badge/MySQL-8.0-blue?style=for-the-badge&logo=mysql)
![RDS](https://img.shields.io/badge/Amazon-RDS-527FFF?style=for-the-badge&logo=amazonrds)
![Free Tier](https://img.shields.io/badge/AWS-Free%20Tier-green?style=for-the-badge&logo=amazonaws)

A end-to-end, production-style database migration from a **local MySQL instance** to **Amazon RDS (MySQL)** using **AWS Database Migration Service (DMS)** — with Full Load + Change Data Capture (CDC) for zero-downtime, zero-data-loss migration.

---

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [AWS Services Used](#aws-services-used)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Step-by-Step Setup](#step-by-step-setup)
  - [1. Prepare Source MySQL Database](#1-prepare-source-mysql-database)
  - [2. Create Amazon RDS Instance](#2-create-amazon-rds-instance)
  - [3. Configure VPC & Security Groups](#3-configure-vpc--security-groups)
  - [4. Set Up AWS DMS Replication Instance](#4-set-up-aws-dms-replication-instance)
  - [5. Configure Source & Target Endpoints](#5-configure-source--target-endpoints)
  - [6. Create & Run Migration Task](#6-create--run-migration-task)
  - [7. Post-Migration Hardening](#7-post-migration-hardening)
- [CDC Replication](#cdc-replication)
- [Security Hardening](#security-hardening)
- [Monitoring & Alerts](#monitoring--alerts)
- [Validation & Testing](#validation--testing)
- [Lessons Learned](#lessons-learned)
- [Cost Estimate (Free Tier)](#cost-estimate-free-tier)

---

## Project Overview

This project demonstrates a real-world cloud database migration scenario — moving a local on-premises MySQL database to a fully managed **Amazon RDS MySQL** instance using **AWS DMS**.

| Property | Details |
|---|---|
| **Migration Type** | Homogeneous (MySQL → MySQL) |
| **Strategy** | Full Load + CDC (Change Data Capture) |
| **Downtime** | Near-zero (CDC keeps source & target in sync) |
| **Data Loss** | None |
| **Environment** | AWS Free Tier |
| **Source** | Local MySQL 8.0 (on EC2 / local machine) |
| **Target** | Amazon RDS MySQL 8.0 (Private Subnet) |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                          AWS Cloud (VPC)                        │
│                                                                 │
│  ┌──────────────┐     ┌──────────────────┐    ┌─────────────┐  │
│  │  Source MySQL│────▶│  DMS Replication │───▶│  Amazon RDS │  │
│  │  (EC2/Local) │     │    Instance      │    │  (Private   │  │
│  │  Public Sub  │     │  (Private Sub)   │    │   Subnet)   │  │
│  └──────────────┘     └──────────────────┘    └─────────────┘  │
│                                │                      │         │
│                                ▼                      ▼         │
│                       ┌──────────────┐      ┌──────────────┐   │
│                       │  CloudWatch  │      │  Automated   │   │
│                       │   Alarms     │      │   Backups    │   │
│                       └──────────────┘      └──────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

> 📌 See [`architecture/architecture-diagram.png`](architecture/architecture-diagram.png) for the full visual diagram.

---

## AWS Services Used

| Service | Purpose |
|---|---|
| **AWS DMS** | Orchestrates full-load + CDC replication |
| **Amazon RDS (MySQL)** | Managed target database |
| **Amazon EC2** | Hosts source MySQL (or simulates on-prem) |
| **Amazon VPC** | Network isolation with public/private subnets |
| **Security Groups** | Firewall rules for DMS ↔ RDS communication |
| **Amazon CloudWatch** | Monitoring, metrics & alarms |
| **AWS IAM** | Roles and permissions for DMS |

---

## Prerequisites

- AWS Account (Free Tier eligible)
- AWS CLI configured (`aws configure`)
- MySQL 8.0 installed locally or on an EC2 instance
- Basic knowledge of VPC, subnets, and security groups

```bash
# Verify AWS CLI
aws --version

# Verify MySQL client
mysql --version
```

---

## Project Structure

```
mysql-to-rds-dms-migration/
│
├── README.md
├── architecture/
│   └── architecture-diagram.png
│
├── sql/
│   ├── create_sample_db.sql        # Sample DB schema for testing
│   ├── insert_sample_data.sql      # Sample data for validation
│   └── validate_migration.sql      # Row count + checksum queries
│
├── configs/
│   ├── dms-replication-instance.json   # DMS instance config reference
│   ├── source-endpoint.json            # Source endpoint settings
│   └── target-endpoint.json            # Target endpoint settings
│
├── cloudwatch/
│   └── alarm-config.json           # CloudWatch alarm definitions
│
└── docs/
    ├── step-by-step-guide.md       # Detailed walkthrough with screenshots
    └── troubleshooting.md          # Common errors & fixes
```

---

## Step-by-Step Setup

### 1. Prepare Source MySQL Database

Enable binary logging on your source MySQL — required for CDC.

```sql
-- Check if binary logging is enabled
SHOW VARIABLES LIKE 'log_bin';

-- In /etc/mysql/mysql.conf.d/mysqld.cnf, add:
-- [mysqld]
-- log_bin = /var/log/mysql/mysql-bin.log
-- binlog_format = ROW
-- expire_logs_days = 1
-- server-id = 1
```

Create a dedicated DMS user with required permissions:

```sql
CREATE USER 'dms_user'@'%' IDENTIFIED BY 'StrongPassword123!';

GRANT SELECT, RELOAD, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'dms_user'@'%';

-- For CDC (binary log reading):
GRANT REPLICATION SLAVE ON *.* TO 'dms_user'@'%';

FLUSH PRIVILEGES;
```

---

### 2. Create Amazon RDS Instance

Via AWS Console → RDS → Create Database:

| Setting | Value |
|---|---|
| Engine | MySQL 8.0 |
| Template | Free Tier |
| Instance class | `db.t3.micro` |
| Storage | 20 GB gp2 |
| Multi-AZ | Disabled (Free Tier) |
| VPC | Your custom VPC |
| Subnet Group | Private subnets only |
| Public Access | **No** |
| Backup Retention | 7 days |

```bash
# Verify RDS connectivity from DMS subnet (after setup)
mysql -h <rds-endpoint> -u admin -p -e "SHOW DATABASES;"
```

---

### 3. Configure VPC & Security Groups

**Security Group: `sg-dms-replication`**
- Inbound: None (DMS initiates outbound)
- Outbound: Port 3306 → `sg-rds-instance`

**Security Group: `sg-rds-instance`**
- Inbound: Port 3306 from `sg-dms-replication` only
- Outbound: None

```bash
# Allow DMS to reach RDS
aws ec2 authorize-security-group-ingress \
  --group-id <sg-rds-id> \
  --protocol tcp \
  --port 3306 \
  --source-group <sg-dms-id>
```

---

### 4. Set Up AWS DMS Replication Instance

AWS Console → DMS → Replication Instances → Create:

| Setting | Value |
|---|---|
| Instance class | `dms.t3.micro` (Free Tier) |
| Engine version | Latest |
| VPC | Same as RDS |
| Multi-AZ | No |
| Publicly accessible | No |

> ⚠️ Place the replication instance in the **same VPC** as your RDS to avoid cross-VPC latency and extra cost.

---

### 5. Configure Source & Target Endpoints

**Source Endpoint (MySQL on EC2):**
```json
{
  "EndpointType": "source",
  "EngineName": "mysql",
  "ServerName": "<ec2-private-ip-or-hostname>",
  "Port": 3306,
  "DatabaseName": "your_database",
  "Username": "dms_user",
  "Password": "StrongPassword123!"
}
```

**Target Endpoint (Amazon RDS):**
```json
{
  "EndpointType": "target",
  "EngineName": "mysql",
  "ServerName": "<rds-endpoint>",
  "Port": 3306,
  "DatabaseName": "your_database",
  "Username": "admin",
  "Password": "<rds-master-password>"
}
```

Test both endpoints using **"Test Connection"** in DMS console before proceeding.

---

### 6. Create & Run Migration Task

AWS Console → DMS → Database Migration Tasks → Create:

| Setting | Value |
|---|---|
| Replication instance | The one created above |
| Source endpoint | Your MySQL source |
| Target endpoint | Your RDS target |
| Migration type | **Full load + ongoing replication** |
| Start task on create | Yes |

**Table Mappings (JSON):**
```json
{
  "rules": [
    {
      "rule-type": "selection",
      "rule-id": "1",
      "rule-name": "include-all-tables",
      "object-locator": {
        "schema-name": "your_database",
        "table-name": "%"
      },
      "rule-action": "include"
    }
  ]
}
```

Monitor the task in **Table Statistics** — rows loaded, inserts, updates, deletes during CDC.

---

### 7. Post-Migration Hardening

```bash
# Enable automated backups (already set during RDS creation)
aws rds modify-db-instance \
  --db-instance-identifier your-rds-instance \
  --backup-retention-period 7 \
  --preferred-backup-window "02:00-03:00"

# Enable deletion protection
aws rds modify-db-instance \
  --db-instance-identifier your-rds-instance \
  --deletion-protection \
  --apply-immediately
```

---

## CDC Replication

Change Data Capture (CDC) keeps source and target **in sync** after the initial full load — capturing INSERT, UPDATE, DELETE events from MySQL binary logs in real time.

```
Source MySQL BinLog  →  DMS reads events  →  Applies to RDS target
     (ROW format)           (ongoing)            (in near real-time)
```

**To monitor CDC lag:**
```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/DMS \
  --metric-name CDCLatencySource \
  --dimensions Name=ReplicationInstanceIdentifier,Value=<your-instance> \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T01:00:00Z \
  --period 300 \
  --statistics Average
```

---

## Security Hardening

| Measure | Implementation |
|---|---|
| RDS in private subnet | No public access; only reachable from within VPC |
| Security group least privilege | Only DMS SG can reach RDS on port 3306 |
| Automated backups | 7-day retention window |
| Deletion protection | Enabled on RDS instance |
| Strong passwords | Used for DMS user and RDS master |
| IAM role for DMS | Separate role with minimal required permissions |

---

## Monitoring & Alerts

CloudWatch Alarms configured:

| Alarm | Metric | Threshold |
|---|---|---|
| High CPU | `CPUUtilization` | > 80% for 5 min |
| Low Free Storage | `FreeStorageSpace` | < 1 GB |
| DB Connections | `DatabaseConnections` | > 50 |
| CDC Latency | `CDCLatencySource` | > 60 seconds |

```bash
# Example: Create CPU alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "RDS-High-CPU" \
  --metric-name CPUUtilization \
  --namespace AWS/RDS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=DBInstanceIdentifier,Value=your-rds-instance \
  --evaluation-periods 2 \
  --alarm-actions <sns-topic-arn>
```

---

## Validation & Testing

After migration, verify data integrity:

```sql
-- Row count comparison (run on both source and target)
SELECT 
  table_name,
  table_rows
FROM information_schema.tables
WHERE table_schema = 'your_database'
ORDER BY table_name;

-- Checksum verification
CHECKSUM TABLE your_table;

-- Sample data spot check
SELECT COUNT(*) FROM orders;
SELECT * FROM orders ORDER BY id DESC LIMIT 10;
```

**Expected result:** Row counts, checksums, and sample records should match between source and target.

---

## Lessons Learned

- **Binary logging must be in ROW format** — STATEMENT or MIXED format causes CDC failures with DMS.
- **DMS replication instance must be in the same VPC as RDS** — saves cost and avoids connectivity issues.
- **Test endpoints before creating tasks** — saves debugging time later.
- **Monitor CDC latency** — spikes indicate the source is generating changes faster than DMS can apply them.
- **Free Tier limits** — `dms.t3.micro` is sufficient for small databases; larger tables may need a bigger class.
- **Avoid schema changes during migration** — DDL changes (ALTER TABLE) during CDC can cause task failures.

---

## Cost Estimate (Free Tier)

| Resource | Free Tier Limit | Used |
|---|---|---|
| RDS `db.t3.micro` | 750 hrs/month | ✅ Within limit |
| DMS `dms.t3.micro` | 750 hrs/month | ✅ Within limit |
| RDS Storage | 20 GB | ✅ Within limit |
| Data Transfer | Minimal (same region) | ✅ ~$0 |

> 💡 **Tip:** Stop the DMS replication instance when not actively migrating to stay within Free Tier hours.

---

## 📎 References

- [AWS DMS Documentation](https://docs.aws.amazon.com/dms/latest/userguide/)
- [MySQL Binary Log Configuration](https://dev.mysql.com/doc/refman/8.0/en/binary-log.html)
- [Amazon RDS MySQL](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_MySQL.html)
- [DMS Best Practices](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_BestPractices.html)

---

## 👤 Author

**Your Name**  
Cloud | DevOps Enthusiast  
[LinkedIn](#) · [GitHub](#)

---

> ⭐ If this project helped you, give it a star!
