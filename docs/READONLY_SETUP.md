# Read-Only Database Login Setup

The application validates SQL at the service layer (SELECT-only, single statement) **and**
requires a database login with no write or DDL privileges. The checks in
`verify_readonly_grants()` are a defense-in-depth gate — the grants below are the real
security boundary.

Run `verify_readonly_grants()` once when saving a connection in the admin UI. Query
execution is blocked until that check passes.

---

## PostgreSQL

Create a dedicated read-only role and grant `SELECT` only on schemas the analytics app
should see.

```sql
-- Run as a superuser or schema owner
CREATE ROLE text_to_sql_reader LOGIN PASSWORD 'choose-a-strong-password';

GRANT CONNECT ON DATABASE analytics TO text_to_sql_reader;
GRANT USAGE ON SCHEMA public TO text_to_sql_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO text_to_sql_reader;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO text_to_sql_reader;
```

**Do not grant:** `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `CREATE`, or membership in
`pg_write_all_data`, `pg_read_all_data` with write-capable defaults, or superuser.

**Verify:**

```sql
SELECT privilege_type
FROM information_schema.role_table_grants
WHERE grantee = 'text_to_sql_reader'
  AND privilege_type IN ('INSERT', 'UPDATE', 'DELETE', 'TRUNCATE');

SELECT has_database_privilege('text_to_sql_reader', 'analytics', 'CREATE');
```

Both queries should return no rows / `false`.

Use `ssl_mode: require` (or `verify-full` in production) in the connection form.

---

## MySQL / MariaDB

Create a user limited to `SELECT` on the target database.

```sql
CREATE USER 'text_to_sql_reader'@'%' IDENTIFIED BY 'choose-a-strong-password';

GRANT SELECT ON analytics.* TO 'text_to_sql_reader'@'%';
FLUSH PRIVILEGES;
```

**Do not grant:** `INSERT`, `UPDATE`, `DELETE`, `CREATE`, `DROP`, `ALTER`, `GRANT`,
`LOCK TABLES`, `TRIGGER`, or global `ALL PRIVILEGES`.

**Verify:**

```sql
SHOW GRANTS FOR 'text_to_sql_reader'@'%';
```

Output should list only `SELECT` on `analytics.*` (and `USAGE`).

Use `ssl_mode: require` when connecting over a network.

---

## Microsoft SQL Server

Create a login and a database user mapped to `db_datareader` only.

```sql
-- Server-level login (SQL authentication)
CREATE LOGIN text_to_sql_reader WITH PASSWORD = 'Choose-A-Strong-Password!';
GO

USE analytics;
GO

CREATE USER text_to_sql_reader FOR LOGIN text_to_sql_reader;
GO

ALTER ROLE db_datareader ADD MEMBER text_to_sql_reader;
GO
```

**Do not grant:** `db_owner`, `db_ddladmin`, `db_datawriter`, `db_securityadmin`, or
server roles such as `sysadmin`. Do not grant explicit `INSERT`, `UPDATE`, `DELETE`, or
`ALTER` on objects.

For Windows/AD authentication, create the user from the Windows login instead of
`CREATE LOGIN`, then add to `db_datareader` only:

```sql
CREATE USER [DOMAIN\\text_to_sql_reader] FROM LOGIN [DOMAIN\\text_to_sql_reader];
ALTER ROLE db_datareader ADD MEMBER [DOMAIN\\text_to_sql_reader];
```

Set `auth_mode: windows` in the connection form and omit username/password.

**Verify:**

```sql
SELECT DISTINCT permission_name
FROM fn_my_permissions(NULL, 'DATABASE')
WHERE permission_name IN ('INSERT', 'UPDATE', 'DELETE', 'ALTER', 'CONTROL');

SELECT IS_MEMBER('db_owner');
SELECT IS_SRVROLEMEMBER('sysadmin');
```

The first query should return nothing; the last two should return `0`.

---

## SQLite (local development)

SQLite has no server-side roles. Use a **read-only file connection**:

1. Point `file_path` at the database file.
2. Ensure the OS user running the app cannot write to that file.
3. The adapter opens the database with `mode=ro` and `PRAGMA query_only = ON`.

**Verify:** `verify_readonly_grants()` attempts `CREATE TEMP TABLE` and expects it to
fail.

For local testing, create and seed the database with a writable tool, then run the app
against the same path in read-only mode.

---

## Application secrets

Connection passwords are encrypted at rest with Fernet (`cryptography`). Set a stable
key in production:

```bash
export FERNET_KEY="$(python -c 'from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())')"
```

API responses mask password fields as `••••••••` after the initial save.
