# Databases-Migration
# Migrating a Web App Database to Amazon RDS (MariaDB)

![AWS](https://img.shields.io/badge/AWS-Cloud_Architecture-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Amazon RDS](https://img.shields.io/badge/Amazon_RDS-MariaDB-527FFF?style=for-the-badge&logo=amazon-rds&logoColor=white)
![AWS Systems Manager](https://img.shields.io/badge/AWS_SSM-Parameter_Store-FF4F8B?style=for-the-badge)
![Amazon CloudWatch](https://img.shields.io/badge/Amazon_CloudWatch-Monitoring-FF4F8B?style=for-the-badge)

## Project Overview

Running a database directly on the same Amazon EC2 instance as a web application introduces single points of failure, resource contention (CPU/RAM spikes), and heavy operational overhead for backups, patching, and scaling.

This repository documents the end-to-end migration of a self-managed local database on an EC2 web server to a decoupled, highly available **Amazon RDS for MariaDB** instance. The migration was executed using the **AWS CLI**, standard MariaDB client utilities (`mysqldump`), and **AWS Systems Manager (SSM) Parameter Store** to achieve a **zero-downtime** application cutover.

---

## Architecture Comparison

### Before Migration (Tightly Coupled)
* **Compute & Storage:** Both the web application and the local MariaDB/MySQL daemon ran on a single EC2 instance.
* **Risk Profile:** High blast radius—instance failure or disk exhaustion takes down both the frontend and database simultaneously.

### After Migration (Decoupled & Multi-AZ Ready)
```text
                        ┌─────────────────────────────────────────┐
                        │                 AWS VPC                 │
                        │                                         │
┌──────────┐   HTTP/S   │  ┌──────────────────┐                   │
│  Users   │ ─────────► │  │  Public Subnet   │                   │
└──────────┘            │  │  Web App (EC2)   │                   │
                        │  └────────┬─────────┘                   │
                        │           │                             │
                        │           │ Pulls DB Config (IAM Role)  │
                        │           ▼                             │
                        │  ┌──────────────────┐                   │
                        │  │  SSM Parameter   │                   │
                        │  │      Store       │                   │
                        │  └──────────────────┘                   │
                        │           │                             │
                        │           │ TCP/3306 (SG Referenced)    │
                        │           ▼                             │
                        │  ┌───────────────────────────────────┐  │
                        │  │         DB Subnet Group           │  │
                        │  │  ┌─────────────┐ ┌─────────────┐  │  │
                        │  │  │ Private AZ1 │ │ Private AZ2 │  │  │
                        │  │  │ Amazon RDS  │ │  (Standby)  │  │  │
                        │  │  │  (MariaDB)  │ │             │  │  │
                        │  │  └──────┬──────┘ └─────────────┘  │  │
                        │  └─────────┼─────────────────────────┘  │
                        └────────────┼────────────────────────────┘
                                     │ Metrics & Logs
                                     ▼
                        ┌─────────────────────────┐
                        │    Amazon CloudWatch    │
                        └─────────────────────────┘
