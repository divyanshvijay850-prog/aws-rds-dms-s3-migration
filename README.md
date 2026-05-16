<div align="center">

# 🚀 AWS DMS — MySQL → S3 Migration Project

![AWS Cloud](https://img.shields.io/badge/AWS-Cloud-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![AWS DMS](https://img.shields.io/badge/AWS-DMS-7B4F9E?style=for-the-badge&logo=amazonaws&logoColor=white)
![Amazon S3](https://img.shields.io/badge/Amazon-S3-3F8624?style=for-the-badge&logo=amazons3&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-8.x-00618A?style=for-the-badge&logo=mysql&logoColor=white)
![CDC](https://img.shields.io/badge/CDC-Enabled-00A1C9?style=for-the-badge)

**End-to-end data migration pipeline from RDS MySQL to Amazon S3 with real-time Change Data Capture replication.**

*Divyansh Vijay · AWS Cloud Engineer · May 2026*

</div>

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AWS Cloud                                    │
│                                                                     │
│  ┌──────────────┐  SQL   ┌──────────┐  DMS Task  ┌─────────────┐  │
│  │MySQL         │───────▶│  AWS RDS │───────────▶│   AWS DMS   │  │
│  │Workbench     │        │  Source  │            │   Engine    │  │
│  │(SQL Ops)     │        │   DB     │            │             │  │
│  └──────────────┘        └──────────┘            └──────┬──────┘  │
│                                                    CDC + │         │
│                                                 Full Load│         │
│                                                          ▼         │
│                                                   ┌─────────────┐  │
│                                                   │  Amazon S3  │  │
│                                                   │   Target    │  │
│                                                   │   Storage   │  │
│                                                   └─────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Tech Stack

| Service | Role |
|---|---|
| 🗄️ **AWS RDS MySQL** | Source Database |
| 🔄 **AWS DMS** | Migration Service (engine) |
| 🪣 **Amazon S3** | Target Storage (CSV output) |
| 🔐 **AWS IAM** | Role-based permissions |
| 🛠️ **MySQL Workbench** | SQL Operations & schema setup |

---

## 📋 Step-by-Step Setup

---

### Step 1 — Create RDS MySQL Database

`AWS Console → RDS → Create Database`

| Setting | Value |
|---|---|
| Create Method | Standard Create |
| Engine | MySQL (latest version) |
| Template | Free Tier |
| Instance Class | `db.t3.micro` |
| Public Access | Yes |

> 💡 **Free Tier** (db.t3.micro) is perfect for development and testing. Public Access is required so MySQL Workbench can connect from your local machine.

---

### Step 2 — Configure Security Group

Allow inbound MySQL traffic on port 3306:

| Type | Protocol | Port | Source | Use Case |
|---|---|---|---|---|
| MySQL/Aurora | TCP | `3306` | `0.0.0.0/0` | Allow MySQL connections |

> ⚠️ Setting source to `0.0.0.0/0` opens MySQL to all IPs. For production, restrict to your IP or VPC CIDR only.

---

### Step 3 — SQL Table Creation

Connect via **MySQL Workbench** and create the schema:

```sql
CREATE DATABASE company_db;
USE company_db;

-- Departments table
CREATE TABLE departments (
    dept_id      INT AUTO_INCREMENT PRIMARY KEY,
    dept_name    VARCHAR(100) NOT NULL,
    location     VARCHAR(100),
    manager_name VARCHAR(100),
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Employees table with FK to departments
CREATE TABLE employees (
    emp_id     INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name  VARCHAR(50),
    gender     VARCHAR(10),
    age        INT,
    email      VARCHAR(100) UNIQUE,
    phone      VARCHAR(15),
    hire_date  DATE,
    salary     DECIMAL(10,2),
    job_title  VARCHAR(100),
    dept_id    INT,
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
);
```

---

### Step 4 — Insert Sample Data

Seed employees and departments for migration testing:

```sql
INSERT INTO employees
    (first_name, last_name, gender, age, email,
     phone, hire_date, salary, job_title, dept_id)
VALUES
    ('Ravi', 'Kumar', 'Male', 26,
     'ravi@gmail.com', '9876500000',
     '2026-05-12', 65000,
     'Data Engineer', 1);
```

---

### Step 5 — Create S3 Bucket

Target destination for migrated data.

> ℹ️ S3 bucket names must be globally unique. Keep them lowercase, no spaces. Create a folder inside the bucket to organize migrated data by source or dataset name.

```
Bucket Name:  my-dms-bucket-divyansh
Folder Path:  my-dms-bucket-divyansh/airline_data/
```

---

### Step 6 — Create IAM Role

Grant DMS permission to write to S3:

| Setting | Value |
|---|---|
| Role Name | `dms-s3-role` |
| Trusted Entity | `dms.amazonaws.com` |
| Policy Attached | `AmazonS3FullAccess` |

> 💡 In production, use a custom policy with least-privilege access to specific S3 bucket paths instead of `AmazonS3FullAccess`.

---

### Step 7 — Create DMS Replication Instance

The compute engine that runs migration tasks:

| Setting | Value | Notes |
|---|---|---|
| Instance Class | `dms.t3.micro` | Free tier eligible |
| Storage | `20 GB` | SSD, default |
| Public Access | `Yes` | Required to reach RDS |
| Engine Version | `Latest` | Auto-selected |

---

### Step 8 — Create Source Endpoint

Connect DMS to RDS MySQL:

| Field | Value |
|---|---|
| Endpoint Type | `Source` |
| Engine | `MySQL` |
| Server Name | `<RDS Endpoint URL>` |
| Port | `3306` |
| SSL Mode | `none` |
| Database | `company_db` |

---

### Step 9 — Create Target Endpoint

Connect DMS to Amazon S3:

| Field | Value |
|---|---|
| Endpoint Type | `Target` |
| Engine | `Amazon S3` |
| Output Format | `CSV` |
| IAM Role ARN | `arn:aws:iam::<id>:role/dms-s3-role` |
| Bucket Name | `my-dms-bucket-divyansh` |
| Bucket Folder | `airline_data` |

---

### Step 10 — Create CDC Migration Task

Full load + ongoing change replication:

> ✅ **CDC (Change Data Capture)** ensures that any INSERT, UPDATE, or DELETE made to your MySQL source is automatically replicated to S3 — enabling near real-time data sync.

**Migration Type:** `Migrate existing data and replicate ongoing changes`

**Table Mapping (table-mapping.json):**

```json
{
  "rules": [
    {
      "rule-type":    "selection",
      "rule-id":      "1",
      "rule-name":    "1",
      "object-locator": {
        "schema-name": "%",
        "table-name":  "%"
      },
      "rule-action": "include"
    }
  ]
}
```

> 📌 The `%` wildcard in schema-name and table-name means *all tables from all schemas* are included in the migration. Narrow this down to specific tables in production workloads.

---

## ✅ Final Summary

| # | Task | Status |
|---|---|---|
| 1 | Create RDS MySQL Database | ✅ Done |
| 2 | Configure Security Group (port 3306) | ✅ Done |
| 3 | Create SQL Schema (company_db) | ✅ Done |
| 4 | Insert Sample Data | ✅ Done |
| 5 | Create S3 Bucket | ✅ Done |
| 6 | Create IAM Role (`dms-s3-role`) | ✅ Done |
| 7 | Create DMS Replication Instance | ✅ Done |
| 8 | Configure Source Endpoint (RDS MySQL) | ✅ Done |
| 9 | Configure Target Endpoint (Amazon S3) | ✅ Done |
| 10 | Create CDC Migration Task | ✅ Done |

**Results:** ✅ Full Load Complete &nbsp;|&nbsp; 🔁 CDC Active &nbsp;|&nbsp; 🪣 S3 Data Written &nbsp;|&nbsp; 📊 CSV Format &nbsp;|&nbsp; 🔐 IAM Secured

---

## 📁 Project Structure

```
aws-rds-dms-s3-migration/
├── README.md                  ← This file
├── sql/
│   ├── schema.sql             ← Table creation (departments + employees)
│   └── seed_data.sql          ← Sample data inserts
├── config/
│   └── table-mapping.json     ← DMS table selection rules
└── screenshots/
    ├── rds_setup.png
    ├── security_group.png
    ├── iam_role.png
    ├── dms_instance.png
    ├── source_endpoint.png
    ├── target_endpoint.png
    └── s3_output.png
```

---

<div align="center">

**Successfully built an end-to-end AWS DMS pipeline with CDC replication from MySQL RDS to Amazon S3.**

*Created by **Divyansh Vijay** — AWS RDS MySQL → Amazon S3 Migration using AWS DMS*

</div>
