Scenario: 
**"We have 100 SSIS packages that need to run simultaneously. How many Worker Threads are required? How should SSIS Scale Out be configured?"**

# Technical Answer

There is **no Microsoft recommendation** that says:

* **100 packages = 100 worker threads**
* **One Worker = X packages**

SSIS does **not** allocate one thread per package. Package execution concurrency depends on:

* Available Scale Out Workers
* CPU cores
* Memory
* SQL Server resource availability
* Package design
* `MaxConcurrentExecutables`
* SSIS runtime scheduler

Microsoft states that **SSIS Scale Out distributes package executions across multiple worker machines**, allowing many packages to execute in parallel. ([Microsoft Learn][1])

---

# Microsoft Scale Out Architecture

```
                 SSISDB
                    │
            Scale Out Master
                    │
      ┌─────────────┼─────────────┐
      │             │             │
 Worker 1      Worker 2      Worker 3
   │              │              │
Packages      Packages      Packages
```

The **Scale Out Master**:

* receives execution requests
* manages execution metadata
* assigns executions to workers

The **Scale Out Workers**:

* poll the Master
* execute packages
* report execution status

This is exactly how Microsoft documents the Scale Out architecture. ([Microsoft Learn][1])

---

# Running 100 Packages Concurrently

If 100 packages are submitted:

```
100 Execution Requests
        │
        ▼
Scale Out Master
        │
        ├────────► Worker A
        ├────────► Worker B
        ├────────► Worker C
        ├────────► Worker D
        └────────► Worker E
```

The Master distributes executions across all available workers.

Microsoft **does not require** assigning a fixed number of packages to each worker. Load distribution depends on worker availability and capacity. ([Microsoft Learn][1])

---

# Worker Threads

This is the point many people misunderstand.

A **Scale Out Worker is not a thread**.

A Worker is an **SSIS execution service running on a machine**.

Inside each Worker:

* multiple package executions can run concurrently
* each package creates its own execution threads
* Data Flow tasks create additional pipeline threads
* execution is limited by server resources

Therefore:

```
100 Packages
≠
100 Worker Threads
```

Instead:

```
100 Packages
      ↓
Distributed across Workers
      ↓
Each Worker executes multiple packages
      ↓
Each package internally uses multiple execution threads
```

Microsoft does not expose a setting such as:

```
WorkerThreads = 100
```

because Scale Out concurrency is managed by the runtime rather than by manually configuring worker-thread counts. ([Microsoft Learn][1])

---

# MaxConcurrentExecutables

Within each SSIS package, concurrency is governed by the `MaxConcurrentExecutables` property.

The default value is:

```
-1
```

which means:

```
Number of logical processors + 2
```

For example:

| CPU Cores | Default MaxConcurrentExecutables |
| --------- | -------------------------------: |
| 8         |                               10 |
| 16        |                               18 |
| 32        |                               34 |

This property controls how many executable tasks **inside a package** can run simultaneously. 
It **does not** control how many packages Scale Out can execute across workers. ([Microsoft Learn][1])

---

# Practical Enterprise Design

For an enterprise workload of **100 concurrent packages**, a typical architecture might be:

```
             SSISDB
                │
        Scale Out Master
                │
──────────────────────────────────────────
 Worker 1   Worker 2   Worker 3   Worker 4
 25 pkgs     25 pkgs     25 pkgs     25 pkgs
```

or

```
Worker A
 16 CPU
 64 GB RAM

Worker B
 16 CPU
 64 GB RAM

Worker C
 16 CPU
 64 GB RAM

Worker D
 16 CPU
 64 GB RAM
```

The exact number of workers depends on:

* package complexity
* ETL duration
* CPU utilization
* memory usage
* database throughput
* I/O bandwidth

Microsoft intentionally does **not** prescribe a fixed "packages per worker" ratio. 

Capacity planning should be based on performance testing and infrastructure sizing. ([Microsoft Learn][1])

---

# Interview-Ready Answer

> **SSIS Scale Out does not require 100 worker threads for 100 concurrent packages. Microsoft Scale Out uses a Master-Worker architecture where the Scale Out Master schedules package executions across one or more Scale Out Workers. Each Worker can execute multiple packages concurrently, and each package internally creates its own execution threads based on `MaxConcurrentExecutables` (default = logical processors + 2).
> Therefore, the number of concurrent packages is determined by the available workers and the hardware capacity—not by configuring one worker thread per package.** ([Microsoft Learn][1])

### Microsoft Learn references

* [Integration Services (SSIS) Scale Out](https://learn.microsoft.com/en-us/sql/integration-services/scale-out/integration-services-ssis-scale-out?view=sql-server-ver17)
* [Run packages in SSIS Scale Out](https://learn.microsoft.com/en-au/sql//integration-services/scale-out/run-packages-in-integration-services-ssis-scale-out?view=sql-server-ver17)

[1]: https://learn.microsoft.com/en-us/sql/integration-services/scale-out/integration-services-ssis-scale-out?view=sql-server-ver17&utm_source=chatgpt.com "SQL Server Integration Services (SSIS) Scale Out - SQL Server Integration Services (SSIS) | Microsoft Learn"
