# Technical SOP: SQL Server Always On Availability Groups – Data Drive (MDF/NDF) Failure, Automatic Failover & Root Cause Analysis (RCA)

**Applies To:** SQL Server 2016, 2017, 2019, 2022, 2025 (Windows Server Failover Cluster)

**Reference:** Microsoft Learn (Always On Availability Groups)

---

Issue (Short Summary):

The primary SQL Server lost access to the data drive containing the MDF/NDF files due to a storage failure (such as a SAN/LUN/disk failure).


# 1. Objective

This SOP explains:

* What happens when the **Primary Replica loses the data drive (MDF/NDF)**.
* Whether **Automatic Failover** occurs.
* The **decision-making process** of Windows Server Failover Cluster (WSFC).
* How to perform a **Root Cause Analysis (RCA)**.
* Recovery and preventive actions.

---

# 2. Environment

| Component         | Value                                  |
| ----------------- | -------------------------------------- |
| SQL Server        | SQL Server Always On AG                |
| Cluster           | Windows Server Failover Cluster (WSFC) |
| Primary Replica   | SQLNODE01                              |
| Secondary Replica | SQLNODE02                              |
| Availability Mode | Synchronous Commit                     |
| Failover Mode     | Automatic                              |
| Listener          | AGListener                             |

---

# 3. Architecture

```text
                    Client Applications
                           │
                    AG Listener (Virtual Name)
                           │
          -----------------------------------------
          │                                       │
   SQLNODE01 (Primary)                  SQLNODE02 (Secondary)
        │                                      │
     MDF/NDF Files                      MDF/NDF Files
        │                                      │
        └──────────Windows Failover Cluster────┘
```

---

# 4. Normal Operation

1. Client connects to AG Listener.
2. Transactions are committed on Primary.
3. Log records are synchronously sent to Secondary.
4. Secondary hardens the log.
5. Transaction commits on Primary.
6. WSFC continuously monitors SQL Server health.

---

# 5. Failure Scenario

## Scenario

Suddenly,

The storage hosting the database files fails.

Examples:

* SAN failure
* Azure Managed Disk failure
* Storage controller failure
* Disk corruption
* LUN disconnected
* Storage path lost

Example:

```text
SQLNODE01

E:\SQLData

SalesDB.mdf

SalesDB.ndf

↓↓↓↓↓↓

Disk Lost
```

---

# 6. What Happens Internally?

## Step 1

SQL Server tries reading database pages.

```text
Read Page

↓

Windows Storage

↓

Disk Not Available
```

SQL Server starts receiving

```text
Operating system error 21

Device not ready
```

or

```text
Operating system error 1117

I/O device error
```

---

## Step 2

SQL Server retries IO.

If storage returns

```text
Failure

Failure

Failure
```

Database engine cannot continue normal processing.

Possible database states:

```text
ONLINE

↓

RECOVERY_PENDING

↓

SUSPECT

↓

OFFLINE
```

---

# 7. Decision Point

Now SQL Server health determines the next action.

There are **two possible outcomes**.

---

# Case A – SQL Server Instance Crashes

Example

The storage failure affects:

* master
* tempdb
* SQL binaries
* critical system databases

SQL Server service terminates.

Flow

```text
Storage Failure

↓

SQL Server Service Stops

↓

Windows Cluster detects SQL Resource Failure

↓

AG Resource Failed

↓

Automatic Failover

↓

Secondary becomes Primary
```

---

## Why Automatic Failover Occurs

Microsoft states automatic failover occurs when:

✔ SQL Resource fails

✔ Replica synchronized

✔ Automatic Failover configured

✔ WSFC healthy

---

# Root Cause

```text
Storage Failure

↓

SQL Instance Failure

↓

WSFC Resource Failure

↓

Automatic Failover
```

This is **instance-level failure**, not simply a database file issue.

---

# Case B – SQL Server Keeps Running

This is much more common.

Example

Only

```text
SalesDB.mdf
```

is affected.

Everything else works.

```text
master

OK

tempdb

OK

msdb

OK

Other Databases

OK
```

SQL Service

```text
Running
```

---

# Result

SalesDB becomes

```text
RECOVERY_PENDING
```

or

```text
SUSPECT
```

However,

WSFC checks

```text
SQL Server Service

Running

↓

Health API

Healthy

↓

Lease

Healthy
```

Cluster concludes

```text
Primary Replica

Still Alive
```

Therefore

## No Automatic Failover

---

# Why?

Microsoft monitors

* SQL Server Service
* Availability Group Lease
* Health Check Timeout
* `sp_server_diagnostics`
* WSFC Resource State

Microsoft does **not** monitor:

```text
SalesDB.mdf exists?

YES / NO
```

This is a common misconception.

---

# 8. What Does `sp_server_diagnostics` Check?

Always On continuously executes

```sql
EXEC sp_server_diagnostics;
```

It reports five health components:

| Component        | Purpose                      |
| ---------------- | ---------------------------- |
| System           | SQL Server process health    |
| Resource         | Memory and resource pressure |
| Query Processing | Query execution health       |
| IO               | I/O subsystem health         |
| Events           | Internal SQL Server events   |

If any component reports an **Error**, the Availability Group evaluates that information according to the configured **FailureConditionLevel**.

---

# 9. FailureConditionLevel

SQL Server has configurable sensitivity.

| Level | Description                                             |
| ----- | ------------------------------------------------------- |
| 1     | SQL Service stopped                                     |
| 2     | SQL Resource failure                                    |
| 3     | Critical SQL internal errors                            |
| 4     | Persistent moderate errors                              |
| 5     | Expanded health conditions reported through diagnostics |

Higher levels increase the types of failures that can trigger automatic failover, but they may also increase the likelihood of failover during severe transient conditions. Review Microsoft guidance before changing the default value.

---

# 10. RCA Flow

## Incident

Users report

```text
Application Down

↓

Database Not Accessible
```

---

## Investigation

### Step 1

Check SQL Service

```powershell
Get-Service MSSQLSERVER
```

Result

```text
Running
```

---

### Step 2

Check AG

```sql
SELECT
replica_server_name,
role_desc,
connected_state_desc,
synchronization_health_desc
FROM sys.dm_hadr_availability_replica_states ars
JOIN sys.availability_replicas ar
ON ars.replica_id = ar.replica_id;
```

---

### Step 3

Check Database State

```sql
SELECT
name,
state_desc
FROM sys.databases;
```

Result

```text
SalesDB

RECOVERY_PENDING
```

---

### Step 4

Check SQL Error Log

Typical errors

```text
Error 823

Error 824

Error 825

Operating system error 21

Operating system error 1117
```

---

### Step 5

Check Windows Event Viewer

Look for

```text
Disk

NTFS

StorPort

MPIO

HBA

SCSI

Storage Spaces
```

---

### Step 6

Check Cluster Log

```powershell
Get-ClusterLog
```

Verify:

* Resource failures
* Lease timeouts
* AG resource transitions

---

# 11. Root Cause Analysis (RCA)

## Problem

Production database unavailable.

---

## Impact

* Application outage
* User login failures
* Transactions stopped
* Database inaccessible

---

## Root Cause

Underlying storage failure caused SQL Server to lose access to MDF/NDF files.

Because the SQL Server service remained operational, the Windows Failover Cluster did **not** detect an Availability Group resource failure requiring automatic failover.

The affected database entered `RECOVERY_PENDING` (or `SUSPECT`), while the SQL Server instance remained healthy enough that AG health monitoring did not transition the primary replica into a failed state.

---

## Evidence

* SQL Error Log
* Windows System Event Log
* Cluster Log
* Storage Controller Logs
* SAN Alerts
* `sys.databases`
* `sys.dm_hadr_availability_replica_states`

---

## Resolution

* Restore storage connectivity.
* Validate database consistency (`DBCC CHECKDB` as appropriate).
* Resume data movement if suspended.
* If required, perform a manual failover to a healthy synchronized replica.
* Restore from backup if corruption cannot be repaired.

---

# 12. Preventive Measures

| Recommendation                                | Purpose                                            |
| --------------------------------------------- | -------------------------------------------------- |
| Use enterprise redundant storage              | Reduce storage failures                            |
| Monitor disk latency and I/O errors           | Early detection of issues                          |
| Review `FailureConditionLevel` configuration  | Align failover behavior with business requirements |
| Perform regular AG failover testing           | Validate HA readiness                              |
| Run `DBCC CHECKDB` on a regular schedule      | Detect corruption early                            |
| Monitor Windows Event Logs and SQL Error Logs | Identify storage issues proactively                |
| Validate SAN/Storage alerts                   | Respond before outages occur                       |

---

# 13. Final Conclusion

A **data drive (MDF/NDF) failure does not automatically trigger an Always On Availability Group failover**.

Automatic failover occurs **only when the failure causes the Availability Group resource or SQL Server instance to be considered unhealthy by Windows Server Failover Clustering and SQL Server health detection** (for example, SQL Server service failure, lease timeout, health check timeout, or qualifying diagnostics reported by `sp_server_diagnostics`), provided the AG is configured for **Synchronous Commit**, **Automatic Failover**, the secondary replica is synchronized, and the WSFC quorum is healthy.

If the SQL Server service continues running and only a user database loses access to its data files, the database may enter `RECOVERY_PENDING` or `SUSPECT`, but **automatic failover will not necessarily occur**. DBA intervention is typically required unless the overall instance health degrades enough to satisfy the configured AG failover policy.

This behavior is consistent with Microsoft's Always On Availability Group documentation and health detection model.
