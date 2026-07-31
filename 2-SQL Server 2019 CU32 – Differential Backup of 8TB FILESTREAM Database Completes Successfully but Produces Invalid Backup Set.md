Issue:
On SQL Server 2019 CU32 where a differential backup of an 8 TB FILESTREAM database completes successfully, but the resulting .bak file is invalid?
The Symptoms:
RESTORE HEADERONLY shows * INCOMPLETE *
RESTORE VERIFYONLY throws Msg 3242: "Not a valid Microsoft Tape Format backup set."
What we've already ruled out:
✅ DBCC CHECKDB is completely clean (no corruption). 
✅ Same error when backed up to different storage target disks. 
✅ Antivirus is completely disabled on the target drives. 
✅ Split backups across multiple .bak files fail with the same header issue. 
✅ Tweaked BUFFERCOUNT and memory settings without success.
Is this a known Microsoft Tape Format (MTF) limitation or CU32 bug specifically tied to large FILESTREAM metadata?

Source: 
https://www.linkedin.com/posts/jitesh-khilosia-21961b15_sqlfamily-share-7488838726685581312-PdQh/?utm_source=share&utm_medium=member_desktop&rcm=ACoAACKCDlIBwCyHwl-zdzyASa1BIP491KNZCLE

Short answer: 
**This would not classify this as a Microsoft Tape Format (MTF) limitation.** 
The symptoms point much more toward a **SQL Server backup stream generation problem or FILESTREAM backup bug/edge case** than an MTF size limitation.

The key clue is:

> Backup command reports SUCCESS, but `RESTORE HEADERONLY` returns `INCOMPLETE` and `RESTORE VERIFYONLY` says "Not a valid Microsoft Tape Format backup set."

A valid SQL Server backup file is structured as an MTF media set with SQL Server backup headers.
`RESTORE HEADERONLY` reads those headers; if they are incomplete, SQL Server did not write a usable backup set even though the BACKUP operation appeared to finish. ([Microsoft Learn][1])

### What this likely means

For an **8 TB FILESTREAM database**, the failure domain is usually one of these:

1. **FILESTREAM backup processing defect**

   * FILESTREAM data is not stored in normal MDF/NDF pages.
   * During backup, SQL Server must coordinate:

     * database file backup metadata
     * FILESTREAM container enumeration
     * transaction consistency
     * backup stream writing
   * Large FILESTREAM containers can expose bugs that do not appear with normal databases.

2. **Backup completes before final media header commit**

   * SQL Server writes:

     * media header
     * backup set header
     * data streams
     * backup completion metadata
   * If the finalization phase fails or is interrupted, you can get a `.bak` file that exists and has a large size but is not a valid backup set.

3. **CU32 regression possibility**

   * SQL Server 2019 CU32 has had reports of unusual issues, but I am not aware of a publicly documented CU32-specific FILESTREAM differential backup corruption fix.
   * I would not assume "known bug" without a Microsoft support confirmation.

---

## Additional checks I would run before opening a Microsoft case

### 1. Confirm build number

```sql
SELECT 
SERVERPROPERTY('ProductVersion') AS Version,
SERVERPROPERTY('ProductLevel') AS Level,
SERVERPROPERTY('ProductUpdateLevel') AS CU;
```

Confirm you are actually on:

```
SQL Server 2019 CU32
15.0.44xx.x
```

---

### 2. Test a FULL backup vs DIFFERENTIAL

This is important.

Run:

```sql
BACKUP DATABASE YourDB
TO DISK='X:\Backup\Test_FULL.bak'
WITH CHECKSUM, COMPRESSION, INIT;
```

Then:

```sql
RESTORE VERIFYONLY
FROM DISK='X:\Backup\Test_FULL.bak'
WITH CHECKSUM;
```

Results interpretation:

| Test                                 | Result                                       | Meaning |
| ------------------------------------ | -------------------------------------------- | ------- |
| Full works, Diff fails               | Differential + FILESTREAM path suspect       |         |
| Both fail                            | FILESTREAM backup engine/storage interaction |         |
| Full and Diff work after CU rollback | CU regression likely                         |         |

---

### 3. Try disabling compression

For an 8 TB FILESTREAM database:

```sql
BACKUP DATABASE YourDB
TO DISK='X:\Backup\Test.bak'
WITH CHECKSUM,
COMPRESSION = OFF,
INIT;
```

Compression normally helps, but FILESTREAM workloads can behave differently.

---

### 4. Check FILESTREAM metadata consistency

Run:

```sql
DBCC CHECKFILEGROUP('YourFilestreamFilegroup') WITH ALL_ERRORMSGS;
```

Also:

```sql
SELECT 
name,
type_desc,
physical_name
FROM sys.database_files;
```

Confirm the FILESTREAM container is online and accessible.

---

### 5. Look for SQL Server error log entries

Immediately after backup:

```sql
EXEC xp_readerrorlog;
```

Search for:

```
backup
filestream
I/O
operating system error
823
824
825
```

---

## My suspicion ranking

Based on your elimination steps:

| Suspect                                | Probability |
| -------------------------------------- | ----------: |
| Storage corruption                     |         Low |
| Antivirus interference                 |    Very Low |
| BUFFERCOUNT/memory issue               |         Low |
| MTF limitation                         |    Very Low |
| FILESTREAM differential backup defect  |        High |
| SQL Server 2019 CU32 regression        |      Medium |
| Hidden I/O timeout during finalization |      Medium |

---

## What I would do in production

I would **not continue relying on differential backups** until proven.

Temporary mitigation:

1. Take regular FULL backups with CHECKSUM.
2. Take FILESTREAM-aware validation restores.
3. Consider patching forward to the latest SQL Server 2019 CU rather than staying on CU32.
4. Open a Microsoft support case with:

   * SQL error log
   * ERRORLOG around backup completion time
   * `msdb.dbo.backupset` entry
   * output of:

     ```sql
     RESTORE LABELONLY
     FROM DISK='yourfile.bak'
     ```
   * dump of FILESTREAM configuration:

     ```sql
     EXEC sp_configure 'filestream access level';
     ```

---

DBA instinct: **the fact that splitting the backup files does not change the failure is the strongest indicator that this is not a file-size or MTF container issue. 
The failure is likely occurring before SQL Server commits the final backup metadata, most likely in the FILESTREAM differential backup path.** 
Its good to test the same workload on the latest CU in a clone before spending time on storage tuning.

[1]: https://learn.microsoft.com/en-us/sql/t-sql/statements/restore-statements-headeronly-transact-sql?view=sql-server-ver17&utm_source=chatgpt.com "RESTORE HEADERONLY (Transact-SQL) - SQL Server | Microsoft Learn"
