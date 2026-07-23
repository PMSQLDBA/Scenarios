## Migrating SQL Server Logins and Passwords to Amazon RDS for SQL Server Without `sysadmin`

### Short Answer

If you **do not have `sysadmin` permission on the source SQL Server**, you **cannot migrate SQL logins with their existing passwords/password hashes yourself**.

To preserve SQL login passwords, you need access to the source login metadata (`sys.sql_logins`) and the ability to script logins with password hashes. In most environments, this requires **`sysadmin`** or equivalent elevated server permissions.

Amazon RDS for SQL Server **does support creating logins with existing password hashes** using `CREATE LOGIN ... PASSWORD = <hash> HASHED`, but you must obtain those hashes from the source SQL Server first.

---

# 1. Requirements for Password-Preserving Migration

### Source SQL Server Requirements

You need permissions to access:

```sql
SELECT *
FROM sys.sql_logins;
```

The information required:

| Information      | Purpose                    |
| ---------------- | -------------------------- |
| Login name       | Recreate login             |
| SID              | Maintain user mapping      |
| Password hash    | Preserve existing password |
| Default database | Recreate login properties  |
| Policy settings  | Maintain password policy   |

Example login information:

```sql
SELECT
    name,
    sid,
    password_hash,
    is_disabled,
    default_database_name
FROM sys.sql_logins;
```

Without access to `password_hash`, password migration is not possible.

---

# 2. AWS Recommended Migration Method

AWS recommends generating login creation scripts from the source SQL Server.

The generated script looks like:

```sql
CREATE LOGIN [ApplicationUser]
WITH PASSWORD = 0x0200AABBCCDDEE... HASHED,
SID = 0x010500000000....,
CHECK_POLICY = ON,
CHECK_EXPIRATION = OFF;
```

The important part:

```sql
PASSWORD = <password_hash> HASHED
```

This allows the existing SQL password to continue working after migration.

---

# 3. Execute Login Scripts on Amazon RDS

Connect to the RDS SQL Server instance using the RDS master account.

Example:

```sql
CREATE LOGIN [ApplicationUser]
WITH PASSWORD = 0x0200AABBCCDDEE... HASHED,
SID = 0x010500000000....,
CHECK_POLICY = ON;
```

Then map the database user:

```sql
USE ApplicationDB;

CREATE USER [ApplicationUser]
FOR LOGIN [ApplicationUser];
```

If the user already exists:

```sql
ALTER USER [ApplicationUser]
WITH LOGIN = [ApplicationUser];
```

---

# 4. Your Case: No `sysadmin` Permission

Since you mentioned:

> "I don't have sysadmin permission"

You have the following limitations:

## You cannot:

❌ Extract password hashes:

```sql
sys.sql_logins.password_hash
```

❌ Generate password-preserving login scripts

❌ Use `sp_help_revlogin` successfully

❌ Recreate users with the same password

---

# 5. Available Options

## Option 1 — Request Source DBA Assistance (Recommended)

Ask the source DBA team to provide:

* SQL login migration scripts
* Password hashes
* SID information

They can run AWS's login migration script or equivalent.

You only execute the output on RDS.

---

## Option 2 — Create New Logins

If password preservation is not mandatory:

Create new logins:

```sql
CREATE LOGIN ReportingUser
WITH PASSWORD='TemporaryPassword@123';
```

Map users:

```sql
USE DatabaseName;

CREATE USER ReportingUser
FOR LOGIN ReportingUser;
```

Then provide new credentials to application owners.

---

## Option 3 — Request Temporary Elevated Access

For a migration activity, request:

* Temporary `sysadmin`
* Or appropriate server security permissions

After migration, remove elevated access.

---

# 6. Important Amazon RDS Limitations

Even after login migration:

| On-Prem SQL Server        | Amazon RDS SQL Server                    |
| ------------------------- | ---------------------------------------- |
| sysadmin role             | ❌ Not available                          |
| server-level control      | Limited                                  |
| sa account                | ❌ Not available                          |
| Windows/AD authentication | Supported with AD integration            |
| SQL logins                | Supported                                |
| Password hash migration   | Supported if source hashes are available |

The RDS master user has elevated privileges but is **not a true `sysadmin` account**.

---

# 7. AWS References

AWS documentation confirms:

* SQL Server logins, database users, roles, and permissions can be migrated using T-SQL scripts.
* Password hashes can be preserved when login scripts include hashed passwords.
* Instance-level privileges such as `sysadmin` cannot be migrated to RDS.

Sources:

* AWS Database Blog — *Migrate logins, database roles, users, and object-level permissions to Amazon RDS for SQL Server using T-SQL*
  [AWS Database Blog - Migrate logins, database roles, users, and object-level permissions to Amazon RDS for SQL Server using T-SQL]
  (https://aws.amazon.com/blogs/database/migrate-logins-database-roles-users-and-object-level-permissions-to-amazon-rds-for-sql-server-using-t-sql)

---

## Final Recommendation for Your Migration

Because you **do not have sysadmin**, follow this approach:

1. Request source DBA to generate login migration scripts with password hashes.
2. Review scripts for required application/service accounts.
3. Execute scripts on RDS using the RDS master account.
4. Validate:

   * Login authentication
   * Database user mapping
   * Application connectivity
   * Permissions

Without source login hash access, **password-preserving migration is not possible**. 
You must either obtain DBA assistance or reset passwords after migration.
