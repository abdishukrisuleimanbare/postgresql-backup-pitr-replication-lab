# PostgreSQL Hands-On Lab: Backups, Point-in-Time Recovery (PITR), and Replication

## 1. Project Overview

This hands-on lab demonstrates practical PostgreSQL database administration skills involving logical backups, Write-Ahead Logging (WAL) archiving, physical base backups, Point-in-Time Recovery (PITR), streaming replication, and replication monitoring.

The main objective was to protect a PostgreSQL database against data loss, recover it to a specific point in time after a simulated disaster, and configure a standby server to receive and replay changes from the primary server.

The lab was completed using PostgreSQL 18.6 on Windows 64-bit.

### Learning Objectives

- Create and verify a logical database backup.
- Configure WAL archiving and create a physical base backup.
- Simulate accidental data deletion.
- Recover a database to a specific point in time.
- Configure a streaming replication standby.
- Monitor replication status and lag.
- Verify that changes on the primary server are available on the standby.

---

## 2. Environment and Tools

| Component | Configuration |
|---|---|
| Database system | PostgreSQL 18.6 |
| Operating system | Windows 64-bit |
| Primary database | `bootcamp` |
| Primary server port | 5432 |
| PITR recovery server port | 5433 |
| Standby server port | 5434 |
| Backup utility | `pg_dump` |
| Restore utility | `pg_restore` |
| Physical backup utility | `pg_basebackup` |
| WAL inspection utility | `pg_waldump` |
| Administration tools | PowerShell and SQL |
| Version control | Git and GitHub |

### Database Used

The `bootcamp` database contained a `students` table with five sample student records.

The records included student names, courses, and marks.

---

## 3. Step 1: Take and Verify a Logical Backup

### Objective

Create a logical backup of the `bootcamp` database and verify that it can be restored successfully into a separate database.

### 3.1 Create the Backup Directory

```powershell
New-Item -ItemType Directory -Force -Path "C:\Users\Admin\backups"
```

### 3.2 Create the Logical Backup

The following command created a PostgreSQL custom-format backup:

```powershell
pg_dump -U postgres -d bootcamp -F c -f "C:\Users\Admin\backups\bootcamp.dump"
```

### 3.3 Verify the Backup Contents

```powershell
pg_restore --list "C:\Users\Admin\backups\bootcamp.dump"
```

This confirmed that PostgreSQL could read the backup archive and list its contents.

### 3.4 Create a Test Database

```sql
CREATE DATABASE bootcamp_check;
```

### 3.5 Restore the Backup

```powershell
pg_restore -U postgres -d bootcamp_check --exit-on-error "C:\Users\Admin\backups\bootcamp.dump"
```

### 3.6 Verify the Restored Records

```sql
SELECT * FROM students ORDER BY student_id;
```

The restored database contained all five student records.

### Step 1 Result

**Successful.** The logical backup was created, inspected, restored into `bootcamp_check`, and verified.

### Key Concepts Learned

- `pg_dump -Fc` creates a PostgreSQL custom-format archive.
- `pg_restore --list` displays the objects contained in the backup.
- Restoring into a separate database provides a practical way to test backup integrity.
- A backup should always be tested through restoration.

---

## 4. Step 2: Enable WAL Archiving and Create a Base Backup

### Objective

Configure PostgreSQL to archive Write-Ahead Log files and create a physical base backup for recovery.

### 4.1 Create the WAL Archive Directory

```powershell
New-Item -ItemType Directory -Force -Path "C:\Users\Admin\backups\wal"
```

### 4.2 Configure PostgreSQL WAL Archiving

The following settings were configured in `postgresql.conf`:

```conf
wal_level = replica
archive_mode = on
archive_command = 'copy /Y "%p" "C:/Users/Admin/backups/wal/%f" >NUL'
```

**Explanation:**

- `wal_level = replica` enables the WAL information required for physical replication.
- `archive_mode = on` enables WAL archiving.
- `archive_command` copies completed WAL segments into the archive directory.

The archive command uses Windows syntax because the lab was performed on Windows.

### 4.3 Restart PostgreSQL

The PostgreSQL Windows service was restarted to apply the configuration:

```powershell
Restart-Service postgresql-x64-18
```

### 4.4 Verify WAL Archiving

The archive directory was inspected to confirm that WAL files were being generated and archived.

Archiver statistics were checked using:

```sql
SELECT * FROM pg_stat_archiver;
```

The archiver statistics showed no failed archive attempts during the recorded verification.

### 4.5 Create the Physical Base Backup

```powershell
pg_basebackup.exe -h localhost -p 5432 -U postgres -D "C:\Users\Admin\backups\base" -Ft -z -Xs -P
```

**Explanation:**

- `-h localhost` connects to the local PostgreSQL server.
- `-p 5432` specifies the primary server port.
- `-U postgres` specifies the PostgreSQL user.
- `-D` specifies the destination directory.
- `-Ft` creates a tar-format backup.
- `-z` compresses the tar files.
- `-Xs` streams WAL during the backup.
- `-P` displays backup progress.

### 4.6 Verify the Base Backup

The backup directory contained:

| File | Purpose |
|---|---|
| `base.tar.gz` | Compressed physical database backup |
| `pg_wal.tar.gz` | WAL files included with the backup |
| `backup_manifest` | Backup manifest used for integrity verification |

WAL archive files were also observed in the archive directory.

### Step 2 Result

**Successful.** WAL archiving was enabled, archive activity was monitored, and a physical base backup was created successfully.

### Key Concepts Learned

- WAL records database changes before they are permanently applied to data files.
- WAL archiving preserves completed WAL segments for recovery.
- A physical base backup provides a starting point for restoring a PostgreSQL cluster.
- WAL replay makes point-in-time recovery possible.

---

## 5. Step 3: Simulate a Disaster and Perform Point-in-Time Recovery

### Objective

Simulate accidental deletion of database records and recover the database to a point in time before the deletion transaction committed.

### 5.1 Simulate Accidental Data Deletion

The `students` table was used to demonstrate accidental data loss.

```sql
DELETE FROM students;
```

This removed the five student records from the primary database.

### 5.2 Inspect the WAL Records

The WAL records were examined using PostgreSQL's `pg_waldump` utility.

The inspection identified the transaction responsible for deleting the student records.

The deletion transaction committed at:

```text
2026-11-02 13:42:43.282373 EAT
```

This timestamp was important because the recovery target had to be before the deletion transaction committed.

### 5.3 Configure a Separate Recovery Environment

A separate recovery data directory was created:

```text
C:\Users\Admin\backups\pitr_recovery_data_corrected
```

The base backup was extracted into this directory, together with the required WAL files.

A WAL restore script was also configured to retrieve archived WAL segments from:

```text
C:\Users\Admin\backups\wal
```

### 5.4 Configure Point-in-Time Recovery

The recovery settings were configured as follows:

```conf
port = 5433
listen_addresses = '127.0.0.1'
archive_mode = off
archive_command = ''
restore_command = 'C:/Users/Admin/backups/restore_wal.cmd "%f" "%p"'
recovery_target_time = '2026-11-02 13:42:40+03'
recovery_target_action = 'promote'
```

A `recovery.signal` file was created to initiate recovery.

### 5.5 Explain the Correct Recovery Timestamp

During the first recovery attempt, the selected recovery target was:

```text
2026-11-02 13:50:44.342686+03
```

This target was after the deletion transaction had committed. Therefore, the recovered database correctly showed the deletion.

After examining the WAL records, the target time was corrected to:

```text
2026-11-02 13:42:40 EAT
```

The corrected target was before the deletion transaction commit time:

```text
2026-11-02 13:42:43.282373 EAT
```

This allowed PostgreSQL to stop WAL replay before the deletion transaction committed.

### 5.6 Start and Verify the Recovered Database

The recovered PostgreSQL instance was started separately on port `5433`.

The recovery state was verified using:

```sql
SELECT pg_is_in_recovery();
```

The result was:

```text
false
```

This confirmed that the recovered instance had been promoted.

### 5.7 Verify the Recovered Student Records

The number of recovered records was checked using:

```sql
SELECT COUNT(*) FROM students;
```

The result was:

```text
5
```

The recovered student records were displayed using:

```sql
SELECT * FROM students ORDER BY student_id;
```

All five original student records were recovered successfully.

### Step 3 Result

**Successful.** Point-in-Time Recovery restored all five student records by using a recovery target before the deletion transaction committed.

### Key Concepts Learned

- WAL replay applies recorded database changes during recovery.
- PITR allows recovery to a selected point in time.
- The recovery target must be before the destructive transaction commits if the objective is to undo that transaction.
- A separate recovery environment allows recovery testing without replacing the original server.
- `pg_is_in_recovery()` helps identify whether a PostgreSQL instance is still in recovery mode.

---

## 6. Step 4: Set Up Streaming Replication

### Objective

Create a dedicated replication role, configure client authentication, and build a standby server using `pg_basebackup`.

### 6.1 Identify the Primary Server

The primary PostgreSQL server was running on:

```text
Host: 127.0.0.1
Port: 5432
Database: bootcamp
```

The following settings were verified:

```sql
SHOW wal_level;
SHOW listen_addresses;
```

The results confirmed that the primary was configured for replication.

### 6.2 Create a Replication Role

On the primary server, the following role was created:

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN;
```

A password was configured interactively.

For security, the actual password is not included in this repository.

### 6.3 Configure PostgreSQL Client Authentication

The following entry was added to `pg_hba.conf`:

```conf
host    replication    replicator    127.0.0.1/32    scram-sha-256
```

This allowed the replication user to connect from the local host using SCRAM authentication.

### 6.4 Reload PostgreSQL Configuration

```sql
SELECT pg_reload_conf();
```

The replication role was also tested by connecting to the primary server.

### 6.5 Create the Standby Data Directory

The standby data directory was created at:

```text
C:\Users\Admin\backups\standby_data
```

### 6.6 Build the Standby Using pg_basebackup

```powershell
pg_basebackup.exe -h 127.0.0.1 -p 5432 -U replicator -D "C:\Users\Admin\backups\standby_data" -R -X stream -P
```

**Explanation:**

- `-h 127.0.0.1` connects to the primary.
- `-p 5432` specifies the primary server port.
- `-U replicator` uses the replication role.
- `-D` specifies the standby data directory.
- `-R` automatically creates standby configuration information.
- `-X stream` streams WAL during the base backup.
- `-P` displays backup progress.

The standby configuration included a `standby.signal` file and connection information.

### 6.7 Configure the Standby Port

The standby was configured to use:

```conf
port = 5434
listen_addresses = '127.0.0.1'
```

The primary remained on port `5432`.

The separate PITR recovery instance remained on port `5433`.

### 6.8 Start the Standby Server

```powershell
pg_ctl.exe -D "C:\Users\Admin\backups\standby_data" -l "C:\Users\Admin\backups\standby.log" start
```

### 6.9 Verify Standby Recovery Mode

On the standby server, the following query was executed:

```sql
SELECT pg_is_in_recovery();
```

The result was:

```text
true
```

This confirmed that the standby was operating in recovery mode.

### 6.10 Verify Replication on the Primary

On the primary server, the following query was executed:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state
FROM pg_stat_replication;
```

The recorded result was:

| Application | Client address | State | Sync mode |
|---|---|---|---|
| walreceiver | 127.0.0.1 | streaming | async |

The `streaming` state confirmed that the standby was receiving WAL from the primary.

### Step 4 Result

**Successful.** The replication role was configured, the standby was initialized, and streaming replication was established between the primary and standby servers.

### Key Concepts Learned

- A replication role provides the required permissions for streaming replication.
- `pg_hba.conf` controls which clients can connect.
- `pg_basebackup -R` creates the standby configuration automatically.
- A standby continuously receives and replays WAL from its primary server.
- `pg_is_in_recovery()` verifies that the standby remains in recovery mode.

---

## 7. Step 5: Monitor Replication Health and Lag

### Objective

Monitor the replication connection, identify connected replicas, check the replication state, and calculate the WAL-position difference.

### 7.1 Check Replication Status

On the primary PostgreSQL server, the following query was executed:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state
FROM pg_stat_replication;
```

The recorded output confirmed:

```text
application_name: walreceiver
client_addr:      127.0.0.1
state:            streaming
sync_state:       async
```

### 7.2 Calculate Replication Lag in Bytes

The assignment specifically requires the following SQL query:

```sql
SELECT
    application_name,
    state,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

**Actual PostgreSQL query output:**

```text
 application_name |  state    | lag_bytes
------------------+-----------+-----------
 walreceiver      | streaming |         0
(1 row)
```

### 7.3 Explain the Query Results

| Field | Actual result | Explanation |
|---|---|---|
| `application_name` | `walreceiver` | Identifies the connected standby process. |
| `state` | `streaming` | Confirms that the standby is actively receiving WAL. |
| `lag_bytes` | `0` | Shows that the reported sent and replayed WAL positions matched at the time of measurement. |

**Understanding the replication lag result:**

- `sent_lsn` represents the WAL position sent by the primary.
- `replay_lsn` represents the WAL position replayed by the standby.
- `pg_wal_lsn_diff()` calculates the difference between the two WAL positions.
- `lag_bytes` displays the difference in bytes.

The query returned `0` bytes while the replication state was `streaming`. This indicates that the reported sent and replayed WAL positions matched at the time of measurement.

This is a byte-based measurement of WAL-position difference and should not be confused with replication delay measured in seconds. It is a point-in-time result and may change as new database transactions occur.

### 7.4 Test Data Replication

To confirm that changes on the primary were available on the standby, a test table was created on the primary server:

```sql
CREATE TABLE IF NOT EXISTS replication_check (
    id INTEGER PRIMARY KEY,
    message TEXT
);
```

A test record was inserted:

```sql
INSERT INTO replication_check (id, message)
VALUES (1, 'Streaming replication verified')
ON CONFLICT (id)
DO UPDATE SET message = EXCLUDED.message;
```

### 7.5 Verify the Test Record on the Standby

The standby server was queried using:

```sql
SELECT * FROM replication_check;
```

The inserted test record appeared on the standby.

The five student records were also restored on the primary server and subsequently verified on the standby.

### Step 5 Result

**Successful.** The primary reported the standby as `streaming`, the required replication lag query executed successfully, and the measured WAL-position difference was zero bytes at the time of verification. Test data inserted on the primary was also confirmed on the standby.

### Key Concepts Learned

- `pg_stat_replication` provides information about connected standby servers.
- Replication state helps identify whether a standby is actively streaming.
- WAL-position differences provide a way to monitor replication lag in bytes.
- A zero-byte difference indicates matching reported sent and replayed WAL positions at the time of measurement.
- Testing replicated records confirms that primary changes reach the standby.

---

## 8. Final Lab Results

| Assignment requirement | Result |
|---|---|
| Create a logical backup | Completed |
| Verify the backup contents | Completed |
| Restore into a separate database | Successful |
| Configure WAL archiving | Successful |
| Create a physical base backup | Successful |
| Simulate accidental deletion | Completed |
| Perform Point-in-Time Recovery | Successful |
| Recover all five student records | Successful |
| Create a replication role | Completed |
| Configure `pg_hba.conf` | Completed |
| Build the standby using `pg_basebackup` | Successful |
| Verify standby recovery mode | Successful |
| Monitor replication status | Successful |
| Execute the required replication lag query | Successful |
| Record actual replication lag output | 0 bytes at time of measurement |
| Verify replicated test data | Successful |
| Verify five student records on standby | Successful |

---

## 9. Practical Evidence Screenshots

The following screenshots provide evidence of the practical work completed during the lab.

### 9.1 Logical Backup and Restore

![Logical Backup and Restore](01-logical-backup-restore.png)

### 9.2 WAL Archiving and Physical Base Backup

![WAL Archiving and Physical Base Backup](02-wal-archiving-base-backup.png)

### 9.3 Point-in-Time Recovery

![Point-in-Time Recovery](03-pitr-five-recovered-students.png)

### 9.4 Streaming Replication Status

![Streaming Replication Status](04-streaming-replication-status.png)

### 9.5 Standby Database Verification

![Standby Database Verification](05-standby-five-students.png)

---

## 10. Technologies and Tools Used

- PostgreSQL 18.6
- SQL
- Windows PowerShell
- `pg_dump`
- `pg_restore`
- `pg_basebackup`
- `pg_waldump`
- PostgreSQL WAL archiving
- PostgreSQL Point-in-Time Recovery
- PostgreSQL streaming replication
- Git
- GitHub

---

## 11. Security Considerations

The repository contains project documentation and selected practical evidence screenshots.

For security reasons, the following sensitive files and information are not included:

- Database passwords.
- PostgreSQL password files such as `pgpass.conf`.
- Authentication tokens and private credentials.
- Complete PostgreSQL data directories.
- Physical database backup archives.
- WAL archive files.

Credentials were configured locally and were not published in the repository.

---

## 12. Conclusion

This hands-on lab provided practical experience in PostgreSQL backup, recovery, and replication administration.

The project demonstrated how to:

1. Create and verify logical database backups.
2. Configure WAL archiving and create physical base backups.
3. Simulate accidental data loss and perform Point-in-Time Recovery.
4. Configure a primary PostgreSQL server and a streaming standby.
5. Monitor replication health and verify data synchronization.

The lab demonstrated how logical backups, physical backups, WAL archiving, PITR, and streaming replication work together to protect database systems and improve availability.

All five practical assignment steps were completed and documented. The recovery and replication results were verified using PostgreSQL commands and SQL queries, including the required replication lag query, which returned a zero-byte WAL-position difference at the time of measurement.

**Project Status: All five practical assignment steps completed and documented.**
