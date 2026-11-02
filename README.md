# PostgreSQL Hands-On Lab: Backups, Point-in-Time Recovery and Replication

## 1. Project Overview

This project demonstrates PostgreSQL database administration techniques, including logical backups, physical backups, Write-Ahead Log (WAL) archiving, Point-in-Time Recovery (PITR), and streaming replication.

**Database:** `bootcamp`

**Environment:** PostgreSQL 18.6 on Windows

## 2. Logical Backup and Restore

Created a custom-format backup using `pg_dump`:

```powershell
pg_dump -h localhost -p 5432 -U postgres -d bootcamp -F c -f "C:\Users\Admin\backups\bootcamp.dump"
```

Inspected the backup:

```powershell
pg_restore --list "C:\Users\Admin\backups\bootcamp.dump"
```

Restored the backup into a separate database using `pg_restore`.

**Result:** Successful backup and restore verification.

## 3. WAL Archiving and Physical Backup

Configured PostgreSQL with:

```ini
wal_level = replica
archive_mode = on
archive_command = 'copy /Y "%p" "C:/Users/Admin/backups/wal/%f" >NUL'
```

Created a physical base backup:

```powershell
pg_basebackup -h localhost -p 5432 -U postgres -D "C:\Users\Admin\backups\base" -Ft -z -Xs -P
```

Verified the backup files and WAL archive.

**Result:** Physical base backup and WAL archiving completed successfully.

## 4. Point-in-Time Recovery (PITR)

Configured a separate recovery instance on port 5433 and used archived WAL files to recover the database.

The corrected recovery target was:

`2026-11-02 13:42:40 EAT`

The deletion transaction was committed at:

`2026-11-02 13:42:43 EAT`

The initial recovery target was after the deletion transaction and therefore did not restore the deleted rows. A separate recovery attempt using the corrected target successfully recovered the five student records.

Verification:

```sql
SELECT pg_is_in_recovery();
SELECT COUNT(*) FROM students;
SELECT * FROM students ORDER BY student_id;
```

**Result:** All five student records successfully recovered.

## 5. Streaming Replication

Configured a primary PostgreSQL server and a standby server.

| Server | Port |
|---|---:|
| Primary | 5432 |
| PITR recovery instance | 5433 |
| Standby | 5434 |

Created a replication account:

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN;
```

Configured client authentication in `pg_hba.conf`:

```text
host replication replicator 127.0.0.1/32 scram-sha-256
```

Created the standby using:

```powershell
pg_basebackup -h 127.0.0.1 -p 5432 -U replicator -D "C:\Users\Admin\backups\standby_data" -R -X stream -P
```

Verified the standby:

```sql
SELECT pg_is_in_recovery();
```

Verified replication on the primary:

```sql
SELECT application_name, client_addr, state, sync_state
FROM pg_stat_replication;
```

Observed a streaming connection with asynchronous replication.

## 6. Replication Testing

Inserted a test record into the primary database and confirmed it appeared on the standby.

Also inserted five sample student records into the primary and verified that all five appeared on the standby.

**Result:** Successful automatic data replication.

## 7. Technologies and Tools

- PostgreSQL 18.6
- pg_dump
- pg_restore
- pg_basebackup
- WAL archiving
- Point-in-Time Recovery
- Streaming replication
- Windows PowerShell
- SQL

## 8. Conclusion

This practical lab demonstrated database backup and recovery, WAL archiving, point-in-time recovery, and streaming replication.

All major practical tasks were completed and verified using PostgreSQL commands and SQL queries.

## 9. Security Note

Database passwords, `pgpass.conf`, private credentials, physical database backup files, and WAL archives are excluded from this public documentation repository.
