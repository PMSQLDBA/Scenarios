# Technical Standard Operating Procedure (SOP)

# Near-Zero Downtime Migration of SQL Server 2016 (On-Premises) to SQL Server 2019 on Azure Virtual Machines using Distributed Availability Groups

**Document Version:** 1.0
**Technology:** Microsoft SQL Server 2016, SQL Server 2019, Windows Server, Azure Virtual Machines, Always On Availability Groups, Distributed Availability Groups (DAG), ExpressRoute
**Migration Type:** Cross-Version Migration (SQL Server 2016 → SQL Server 2019)
**Downtime Objective:** Less than 2 Minutes
**Recovery Point Objective (RPO):** Zero Data Loss
**Recovery Time Objective (RTO):** < 2 Minutes

<img width="1232" height="732" alt="image" src="https://github.com/user-attachments/assets/ef3a6a78-0308-44dc-a66b-ff9df12f685d" />

<img width="1445" height="410" alt="image" src="https://github.com/user-attachments/assets/f0d99cf9-50f7-4200-abf3-06375a982b3c" />

<img width="1437" height="677" alt="image" src="https://github.com/user-attachments/assets/82b3e8d7-15ff-4834-b678-61ce29a664b0" />

<img width="1455" height="258" alt="image" src="https://github.com/user-attachments/assets/abb43e64-d3b5-4a64-9231-5fb77c4ada48" />

Alternative Image: 

<img width="913" height="500" alt="image" src="https://github.com/user-attachments/assets/26863664-7a24-4645-bceb-978f9be52046" />

Image Reference: 

https://www.linkedin.com/posts/bibhuti-bhusan-singh-1b96251ab_azuremigration-sqlserver-dataengineering-activity-7488089007080407040-N5Wv?utm_source=share&utm_medium=member_desktop&rcm=ACoAACKCDlIBwCyHwl-zdzyASa1BIP491KNZCLE


---

# 1. Purpose

This SOP describes the complete process for performing a near-zero downtime migration of multiple SQL Server databases from an on-premises SQL Server 2016 Always On Availability Group to SQL Server 2019 hosted on Azure Virtual Machines.

The migration leverages **Distributed Availability Groups (DAG)** to continuously synchronize databases between two independent Availability Groups before performing a controlled failover.

This procedure is suitable for environments hosting:

* Mission-critical production workloads
* ERP systems
* Banking applications
* Healthcare systems
* E-commerce platforms
* Manufacturing systems
* Databases ranging from hundreds of GB to multiple TB

---

# 2. Business Scenario

Customer Environment

| Component           | Details                       |
| ------------------- | ----------------------------- |
| Source SQL Version  | SQL Server 2016 Enterprise    |
| Target SQL Version  | SQL Server 2019 Enterprise    |
| Source Environment  | On-Premises Datacenter        |
| Target Environment  | Azure Virtual Machines        |
| Number of Databases | 50+                           |
| Database Size       | 1 TB+ each                    |
| HA Configuration    | Always On Availability Groups |
| Downtime SLA        | Less than 2 minutes           |
| Data Loss Allowed   | Zero                          |

---

# 3. Why Distributed Availability Groups?

Traditional migration approaches require long outages because databases must be backed up, restored, and synchronized during maintenance windows.

Distributed Availability Groups eliminate most of this downtime by continuously replicating transaction logs to the target environment before the migration.

Advantages include:

* Continuous data synchronization
* Cross-version support
* Cross-datacenter replication
* Minimal application downtime
* Zero data loss when synchronized
* No shared WSFC required between environments

---

# 4. High-Level Migration Architecture

```
               On-Premises

      SQL Server 2016 AG
      +--------------------+
      | Primary Replica    |
      | Secondary Replica  |
      +--------------------+
                │
                │
       Distributed AG
      (Async Replication)
                │
                ▼

          Azure ExpressRoute

                │
                ▼

        SQL Server 2019 AG

      +--------------------+
      | Primary Replica    |
      | Secondary Replica  |
      +--------------------+

          Azure Virtual Machines
```

---

# 5. Prerequisites

## Infrastructure

* Azure Subscription
* Virtual Network
* SQL Server 2019 Enterprise
* Windows Server Failover Cluster
* Azure Load Balancer
* Availability Set or Availability Zone
* Azure Storage
* DNS Configuration

---

## SQL Server Requirements

Source

* SQL Server 2016 Enterprise
* Always On enabled
* Availability Group healthy

Target

* SQL Server 2019 Enterprise
* Always On enabled
* AG already created
* Listener configured

---

## Networking

Recommended:

* Azure ExpressRoute

Alternative:

* Site-to-Site VPN

Network Requirements

* Low latency
* High bandwidth
* Stable connectivity
* SQL Ports open
* WSFC ports
* Endpoint connectivity

---

# 6. Migration Phases

## Phase 1 – Assessment

Tasks

Inventory

* Databases
* Logins
* SQL Agent Jobs
* Linked Servers
* SSIS Packages
* CLR Assemblies
* Endpoints
* Certificates
* SQL Credentials

Health Checks

* DBCC CHECKDB
* Backup validation
* AG Health
* Replica latency
* Transaction log growth
* VLF count
* Wait statistics
* Disk performance

Deliverables

* Migration Assessment Report
* Compatibility Report
* Risk Assessment

---

## Phase 2 – Azure Environment Build

Deploy

* Azure VMs
* SQL Server 2019
* Storage
* TempDB
* Backup drives
* Availability Group
* WSFC

Configure

* MAXDOP
* Cost Threshold
* Instant File Initialization
* Memory
* Trace Flags (if applicable)
* Maintenance Jobs

---

## Phase 3 – Database Seeding

Perform

Full Backup

↓

Copy to Azure

↓

Restore WITH NORECOVERY

↓

Restore Differential

↓

Restore Transaction Logs

↓

Join Database to AG

↓

Validate LSN

Purpose

Reduce synchronization time before enabling replication.

---

## Phase 4 – Create Distributed Availability Group

Configure

Source AG

↓

Target AG

↓

Distributed AG

↓

Asynchronous Replication

At this stage

* Production remains on-premises.
* Azure continuously receives transaction logs.
* Users continue working without interruption.

---

# 7. Synchronization Monitoring

Monitor

DMVs

```
sys.dm_hadr_database_replica_states

sys.dm_hadr_availability_replica_states

sys.dm_hadr_availability_group_states
```

Monitor

* Send Queue
* Redo Queue
* LSN synchronization
* Replica health
* Latency
* Synchronization state

Target State

```
Healthy

Synchronized

No backlog
```

---

# 8. Cutover Procedure

Maintenance Window Begins

### Step 1

Notify users

↓

Stop application services

↓

Disable scheduled jobs

↓

Stop ETL

↓

Pause integrations

---

### Step 2

Switch DAG

```
Asynchronous

↓

Synchronous
```

Wait until

```
Synchronization = SYNCHRONIZED
```

Validate

```
Last Commit LSN

Current LSN

Redo Queue

Send Queue

Synchronization Health
```

---

### Step 3

Perform Failover

```
Failover Distributed AG

↓

Azure becomes Primary

↓

Applications reconnect
```

Expected Duration

Less than 30 seconds

---

### Step 4

Update

* DNS
* Listener
* Connection Strings
* Application Configurations
* Firewall Rules

---

### Step 5

Start

* SQL Agent Jobs
* Applications
* ETL
* Reporting
* Monitoring

---

# 9. Post Migration Tasks

Validate

Database

* Online
* Read/Write
* Compatibility Level
* Recovery Model

Validate

Security

* Logins
* Users
* Permissions
* SQL Roles

Validate

Application

* Connectivity
* Transactions
* Reports
* APIs
* Batch Jobs

Validate

Performance

* CPU
* Memory
* Disk
* Wait Statistics
* Query Store
* Blocking

---

# 10. Automation

PowerShell should automate:

* Backup copy
* Restore operations
* AG creation
* DAG creation
* Health validation
* Synchronization monitoring
* Failover
* DNS updates
* SQL Agent enable/disable
* Application health checks
* Logging
* Rollback triggers

Benefits

* Reduced human error
* Consistent execution
* Faster cutover
* Repeatable deployments

---

# 11. Rollback Strategy

Rollback is only required if validation fails immediately after cutover.

Procedure

1. Stop applications
2. Fail back to on-premises AG
3. Redirect DNS
4. Restart applications
5. Resume production

Rollback Time

Typically under 5 minutes.

---

# 12. Validation Checklist

### Infrastructure

* Azure VM healthy
* Storage healthy
* Network healthy
* Cluster healthy

### SQL Server

* AG synchronized
* Database online
* No Suspect databases
* SQL Agent running

### Application

* Login successful
* Transactions successful
* Reports operational
* APIs operational

### Performance

* CPU acceptable
* Memory stable
* Disk latency acceptable
* No blocking
* No excessive waits

---

# 13. Advantages

* Near-zero downtime
* Zero data loss
* Cross-version migration (2016 → 2019)
* No lengthy maintenance windows
* Continuous synchronization
* High availability during migration
* Reduced business impact
* Supports multi-terabyte databases
* Easily repeatable process
* Minimal manual intervention through automation

---

# 14. Limitations

* Requires SQL Server Enterprise Edition.
* Distributed Availability Groups require Always On Availability Groups to be configured at both source and target.
* Stable, low-latency network connectivity (preferably ExpressRoute) is essential for timely synchronization.
* Initial seeding of very large databases can take significant time depending on available bandwidth.
* Applications using server-specific names or hard-coded connection strings may require additional planning.
* Cross-version support is limited to supported SQL Server upgrade paths; validate compatibility before implementation.
* Comprehensive testing is mandatory before production cutover.

---

# 15. Best Practices

* Perform multiple end-to-end rehearsal migrations in a non-production environment.
* Validate application compatibility with SQL Server 2019 before migration.
* Use full, differential, and transaction log backups for efficient initial seeding.
* Monitor send queue, redo queue, and synchronization health throughout the migration.
* Keep the Distributed AG in asynchronous mode during normal synchronization and switch to synchronous mode only immediately before cutover.
* Disable scheduled SQL Agent jobs and ETL processes during the maintenance window to prevent conflicts.
* Script and automate all migration activities using PowerShell to ensure consistency and reduce manual errors.
* Validate logins, SQL Agent jobs, linked servers, certificates, and permissions after migration.
* Maintain a documented rollback plan and confirm failback procedures through testing.
* Continue enhanced monitoring for at least 48–72 hours after migration to identify any post-cutover performance or application issues.

---

# 16. Estimated Migration Timeline

| Phase                          |                                             Estimated Duration |
| ------------------------------ | -------------------------------------------------------------: |
| Assessment and Planning        |                                                      1–2 Weeks |
| Azure Environment Preparation  |                                                       2–5 Days |
| Initial Database Seeding       | Hours to Several Days (depends on database size and bandwidth) |
| Distributed AG Synchronization |                                       Continuous until cutover |
| Final Synchronization          |                       Typically a few seconds to a few minutes |
| Production Cutover             |                                            Less than 2 Minutes |
| Post-Migration Validation      |                                                      2–4 Hours |
| Hypercare and Monitoring       |                                                    48–72 Hours |

---

# 17. Conclusion

Using **Distributed Availability Groups (DAGs)** provides a robust and proven approach for migrating SQL Server workloads from **SQL Server 2016 on-premises to SQL Server 2019 on Azure Virtual Machines** with **near-zero downtime**. 
By pre-seeding databases, continuously synchronizing transaction logs, and executing a controlled synchronous failover during the maintenance window, organizations can migrate large-scale, multi-terabyte environments while meeting stringent business continuity objectives. 
When combined with comprehensive planning, automation, and validation, this methodology minimizes operational risk and delivers a seamless transition to Azure.

Reference: 

