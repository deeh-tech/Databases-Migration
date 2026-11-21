# Migrating a Web App Database to Amazon RDS (MariaDB)

## Project Overview

Managing a self-hosted database on an Amazon EC2 instance introduces heavy operational overhead for manual backups, OS/database patching, high availability, and scaling.

This repository documents the end-to-end migration of a self-managed database hosted on an EC2 instance to a decoupled, fully managed **Amazon RDS for MariaDB** instance. The migration was executed using the **AWS CLI**, standard MariaDB client utilities (`mysqldump`), and **AWS Systems Manager (SSM) Parameter Store** to achieve a **zero-downtime** application cutover.

---

## Architecture Comparison

### Before Migration (Self-Managed on EC2)
* **Database Tier:** A self-managed MariaDB/MySQL database hosted directly on an EC2 instance.
* **Operational Profile:** Required manual administration for backups, security patches, failover handling, and storage scaling.

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
```

---

## Prerequisites & Environment

* **AWS Account** with IAM permissions for VPC, EC2, RDS, Systems Manager, and CloudWatch.
* **AWS CLI v2** configured on the bastion/management terminal or EC2 instance.
* **MariaDB / MySQL Client Tools** (`mariadb` / `mysql` and `mysqldump`) installed on the source EC2 instance.
* **IAM Instance Profile** attached to the EC2 web server with `ssm:GetParameter` and `ssm:GetParameters` permissions.

---

## Summary of Migration Steps

1. **Network & Security Setup:** Provisioned isolated private subnets across multiple Availability Zones, created a DB Subnet Group, and configured a dedicated Security Group restricting inbound MariaDB traffic (`TCP 3306`) solely to the EC2 web server.
2. **RDS Provisioning (AWS CLI):** Created a managed **Amazon RDS for MariaDB** instance (`db.t3.micro`, `gp3` storage, non-publicly accessible) programmatically via the AWS CLI.
3. **Database Export & Restore (`mysqldump`):** Generated a consistent SQL snapshot of the EC2 database using `mysqldump` and imported the schema and data into the newly provisioned RDS endpoint.
4. **Zero-Downtime Cutover (SSM Parameter Store):** Updated the database endpoint, username, and encrypted password (`SecureString`) in **AWS Systems Manager Parameter Store**, allowing the web app to switch databases dynamically without downtime, then disabled the old EC2 database service.
5. **Monitoring & Verification (CloudWatch):** Validated live read/write transactions on the application and established performance baselines in **Amazon CloudWatch** (`CPUUtilization`, `DatabaseConnections`, `FreeableMemory`, and `FreeStorageSpace`).

---

## Output & Verification

* **Data Parity Confirmed:** Querying the **Amazon RDS (MariaDB)** instance directly via the CLI returned the exact same records and row counts displayed on the live web application frontend.
* **Zero Data Loss:** Both pre-migration records (imported via `mysqldump`) and newly created records submitted through the web app after the **SSM Parameter Store** cutover matched identically between the web interface and the RDS database tables.

---

## Repository Structure

```text
Databases-Migration/
├── images/             # Lab architecture diagrams, CLI outputs, and CloudWatch screenshots
└── README.md           # Complete lab documentation and migration runbook
```
