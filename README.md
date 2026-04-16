# RAT Capture and Replay Procedure

## 1. Objective

This document describes the technical procedure required to execute Oracle Real Application Testing (RAT) capture and replay activities, including workload capture, AWR export, SQL Tuning Set (STS) preparation, replay initialization, replay execution, and validation.

## 2. Scope

This procedure applies to both the source environment used for workload capture and the target environment used for workload replay.

## 3. Important Naming Note

All fields highlighted in red must be reviewed and adapted to the environment where the RAT procedure will be executed.

Examples of fields that must be reviewed include:

- `_YourApp`
- `YOUR_APP`
- `YOUR_TABLE`
- `YOUR_SCHEMA`
- `YOUR_TABLESPACE`
- `DBNAME`
- `SYSTEM_TABLESPACE`
- `SYSAUX_TABLESPACE`
- `nnnnn`
- `RECO_NAME`

## 4. General Prerequisites

- The source and target environments must be available.
- Oracle RAT must be properly licensed and configured.
- Required database privileges must be granted.
- Sufficient filesystem space must be available for capture, export, flashback, archive logs, and replay files.
- Connectivity between source and target servers must be available for file transfer.
- The users or sessions to be monitored must be clearly identified.

## 5. RAT Capture Procedure

### 5.1 Source Environment Information

- **Platform:** ExaCC X10M-2 Gen2
- **Database:** `Primary Database`

### 5.2 Create the Directory for Capture Files

Create the OS directory:

```bash
mkdir -p /acfs/capture_YourApp
```

Create the Oracle directory object:

```sql
CREATE OR REPLACE DIRECTORY capture_YourApp AS '/acfs/capture_YourApp';
GRANT ALL ON DIRECTORY capture_YourApp TO public;
```

### 5.3 Create Capture Filters (If Required)

If it is necessary to restrict the capture to specific users, create filters such as:

```sql
BEGIN
   DBMS_WORKLOAD_CAPTURE.ADD_FILTER (
      fname      => 'USER_XXXXX',
      fattribute => 'USER',
      fvalue     => 'F_XXXXX'
   );
END;
/

BEGIN
   DBMS_WORKLOAD_CAPTURE.ADD_FILTER (
      fname      => 'YYYYYY',
      fattribute => 'USER',
      fvalue     => 'F_YYYYYY'
   );
END;
/
```

### 5.4 Create an AWR Snapshot Before Starting the Capture

```sql
EXEC DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();
```

### 5.5 Stop Redo Apply on the Standby Database

On standby server `exacc0305`:

```sql
sqlplus / as sysdba
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
```

### 5.6 Start the Capture on the Source Database

On the source database:

```sql
sqlplus / as sysdba

BEGIN
   DBMS_WORKLOAD_CAPTURE.START_CAPTURE (
      name           => 'CAPTURE_RAT_YourApp',
      dir            => 'CAPTURE_YourApp',
      default_action => 'INCLUDE'
   );
END;
/
```

### 5.7 Monitor the Capture

```sql
SET LINESIZE 300
COL NAME FOR A22
COL DIRECTORY FOR A15
COL STATUS FOR A20

SELECT name,
       id,
       status,
       start_time,
       ROUND(capture_size / POWER(1024,3), 0) AS cap_size_gb,
       user_calls
FROM DBA_WORKLOAD_CAPTURES;
```

### 5.8 Finish the Capture

Allow the capture to run for approximately 2 hours, then finish it:

```sql
BEGIN
   DBMS_WORKLOAD_CAPTURE.FINISH_CAPTURE;
END;
/
```

Validate the final status:

```sql
SET LINESIZE 300
COL NAME FOR A22
COL DIRECTORY FOR A15
COL STATUS FOR A20

SELECT name,
       id,
       status,
       start_time,
       capture_size,
       user_calls
FROM DBA_WORKLOAD_CAPTURES;
```

### 5.9 Export the AWR Data

Retrieve the `capture_id`:

```sql
CONNECT / AS SYSDBA
SELECT id, name
FROM dba_workload_captures;
```

Export AWR data for the selected capture:

```sql
BEGIN
   DBMS_WORKLOAD_CAPTURE.EXPORT_AWR(capture_id => 1);
END;
/
```

### 5.10 Create the SQL Tuning Set for SPA

Create the staging table:

```sql
BEGIN
   DBMS_SQLTUNE.CREATE_STGTAB_SQLSET(
      table_name      => 'YOUR_TABLE',
      schema_name     => 'YOUR_SCHEMA',
      tablespace_name => 'YOUR_TABLESPACE'
   );
END;
/
```

Create the SQL tuning set:

```sql
BEGIN
   DBMS_SQLTUNE.CREATE_SQLSET(
      sqlset_name => 'STS_YourApp',
      description => '19C STS YourApp'
   );
END;
/
```

Query the SQL set if required:

```sql
SELECT sql_text
FROM dba_sqlset_statements
WHERE sqlset_name = 'STS_YourApp';
```

### 5.11 Identify the AWR Snapshot Range

```sql
SET LINES 200
COL NAME FOR A32
COL STATUS FOR A13

SELECT id,
       name,
       status,
       TO_CHAR(start_time,'dd-mm-yy hh24:mi:ss') AS start_time,
       TO_CHAR(end_time,'dd-mm-yy hh24:mi:ss') AS end_time,
       awr_dbid,
       awr_begin_snap,
       awr_end_snap
FROM dba_workload_captures
ORDER BY 4;
```

Use the snapshot IDs returned above to create the baseline:

```sql
EXECUTE DBMS_WORKLOAD_REPOSITORY.CREATE_BASELINE(
   start_snap_id => nnnnn,
   end_snap_id   => nnnnnn,
   baseline_name => 'YOUR_APP_baseline'
);
```

### 5.12 Load the Workload into the SQL Tuning Set

```sql
DECLARE
   baseline_ref_cur DBMS_SQLTUNE.SQLSET_CURSOR;
BEGIN
   OPEN baseline_ref_cur FOR
      SELECT VALUE(p)
      FROM TABLE(
         DBMS_SQLTUNE.SELECT_WORKLOAD_REPOSITORY(
            nnnnn,
            nnnnnn,
            NULL,NULL,NULL,NULL,NULL,NULL,NULL,
            'ALL'
         )
      ) p;

   DBMS_SQLTUNE.LOAD_SQLSET('STS_YourApp', baseline_ref_cur);
END;
/
```

### 5.13 Pack the SQL Tuning Set

```sql
BEGIN
   DBMS_SQLTUNE.PACK_STGTAB_SQLSET(
      sqlset_name          => 'STS_YourApp',
      sqlset_owner         => 'SYS',
      staging_table_name   => 'YOUR_TABLE',
      staging_schema_owner => 'YOUR_SCHEMA'
   );
END;
/
```

### 5.14 Export the STS Table

```bash
exp system/pwd file=/acfs/dump/export_sts_YOUR_APP.dmp tables=YOUR_TABLE
```

Transfer the exported `.dmp` file to the target environment.

### 5.15 Prepare the Target Environment for Import

- **Platform:** ExaCC X10M-2

```bash
mkdir -p /acfs/dump
mkdir -p /acfs/replay_YourApp
```

Copy the dump file to `/acfs/dump`, then execute:

```bash
imp system/pwd file=/acfs/dump/export_sts_YOUR_APP.dmp log=imp_YOUR_APP.log full=y
```

Transfer the capture files:

```bash
cd /acfs/replay_YourApp
scp oracle@exacc01db01:/acfs/capture_YourApp .
```

### 5.16 Unpack the SQL Tuning Set

```sql
sqlplus / as sysdba

BEGIN
   DBMS_SQLTUNE.UNPACK_STGTAB_SQLSET(
      sqlset_name          => 'STS_YourApp',
      sqlset_owner         => 'SYS',
      replace              => TRUE,
      staging_table_name   => 'YOUR_TABLE',
      staging_schema_owner => 'YOUR_SCHEMA'
   );
END;
/
```

### 5.17 Validate the Imported SQL Set

```sql
SELECT SQL_ID, SQL_TEXT
FROM TABLE(DBMS_SQLTUNE.SELECT_SQLSET('STS_YourApp'));

SELECT sqlset_name,
       sql_id,
       executions
FROM dba_sqlset_statements
WHERE sqlset_name = 'STS_YourApp'
ORDER BY 3 DESC;
```

## 6. RAT Replay Procedure

### 6.1 Target Environment Information

- **Server / Cluster:** `Standby`
- **Database:** `DBNAME`

### 6.2 Verify Standby Synchronization

Verify that the primary and standby databases are synchronized before starting the replay procedure.

### 6.3 Enable ARCHIVELOG and FLASHBACK

Set the database to `ARCHIVELOG` and enable `FLASHBACK`:

```sql
sqlplus / as sysdba
alter system set cluster_database=FALSE scope=spfile sid='*';

sqlplus / as sysdba
startup mount
alter database archivelog;
alter database set db_recovery_flash_area_size=10000g scope=both sid='*';
alter database set db_recovery_flash_area='+RECO_NAME' scope=both sid='*';
alter database flashback on;
alter database open;
archive log list
shutdown immediate
exit;

srvctl start database -d DBNAME
```

### 6.4 Activate the Standby Database as Read/Write

```sql
sqlplus / as sysdba
alter database activate physical standby database;
```

### 6.5 Increase SYSTEM and SYSAUX Tablespaces if Required

```sql
col file_name for a55
set lines 132
select file_name, bytes/power(1024,3)
from dba_data_files
where tablespace_name in ('SYSTEM','SYSAUX');

set echo on
alter tablespace system add datafile 'SYSTEM_TABLESPACE' size 5g autoextend on next 1g maxsize 10g;
alter tablespace sysaux add datafile 'SYSAUX_TABLESPACE' size 5g autoextend on next 1g maxsize 10g;
```

### 6.6 Create the Replay Directory

Create the directory object used to process the files generated by RAT:

```sql
CREATE DIRECTORY replay_YOUR_APP AS '/acfs/replay_YOUR_APP';
GRANT ALL ON DIRECTORY replay_YOUR_APP TO public;
```

If necessary, create user filters according to the replay requirements.

- Review the filter definition used during the capture process.
- If the replay follows the same workload scope, create the same filters used during capture.

### 6.7 Process the Capture Metadata

Execute capture processing. This step generates the metadata required for replay processing:

```sql
EXEC DBMS_WORKLOAD_REPLAY.PROCESS_CAPTURE(
   capture_dir    => 'REPLAY_YOUR_APP',
   parallel_level => 64
);
```

### 6.8 Calibrate the Replay

Run calibration to verify the number of processes required to execute the captured workload:

```bash
wrc system/pwd mode=calibrate replaydir=/acfs/replay_YOUR_APP
```

Based on the example report, the recommendation is to use at least 3 clients.

### 6.9 Create Restore Points

Restore points are critical if the replay must be executed multiple times without rebuilding the environment.

```sql
drop restore point before_rat_replay;
create restore point before_rat_preprocess guarantee flashback database;
create restore point restore_after_statistics guarantee flashback database;
```

### 6.10 Initialize the Replay

```sql
BEGIN
   DBMS_WORKLOAD_REPLAY.INITIALIZE_REPLAY(
      replay_name => 'DBREPLAY_YOUR_APP',
      replay_dir  => 'REPLAY_YOUR_APP'
   );
END;
/
```

### 6.11 Remap Connections for CDB/PDB Environments

In CDB environments, remap the replay connections so they point to the target PDB. RAT execution is performed at the CDB level while pointing to the PDB.

Reference: `How to Setup and Run a Database Testing Replay in an Oracle Multitenant Environment (Real Application Testing - RAT) (Doc ID 1937920.1) / KB133988`

Step 6: Remap connections to the respective PDBs.

```sql
select conn_id, schedule_cap_id, capture_conn, replay_conn
from dba_workload_connection_map;

exec DBMS_WORKLOAD_REPLAY.REMAP_CONNECTION(
   connection_id     => 1,
   SCHEDULE_CAP_ID   => 1,
   replay_connection => '(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1522))(CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=pdb1)))'
);

exec DBMS_WORKLOAD_REPLAY.REMAP_CONNECTION(
   connection_id     => 2,
   SCHEDULE_CAP_ID   => 2,
   replay_connection => '(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1522))(CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=pdb2)))'
);

-- Verify remapping.
select conn_id, schedule_cap_id, capture_conn, replay_conn
from dba_workload_connection_map;
```

### 6.12 Prepare the Replay

```sql
BEGIN
   DBMS_WORKLOAD_REPLAY.PREPARE_REPLAY(
      synchronization    => TRUE,
      connect_time_scale => 100,
      think_time_scale   => 100
   );
END;
/
```

### 6.13 Validate Database Links and TNS Configuration

Terminate or remove all database links if required:

```sql
select * from dba_db_links;
```

The expected result in the example is `no rows selected`.

Rename the `tnsnames.ora` files on both cluster nodes as required by the replay setup.

### 6.14 Create an Additional Restore Point Before Replay

```sql
create restore point before_rat_replay_2 guarantee flashback database;
```

### 6.15 Start the Replay Clients

Start the replay clients according to the calibration result.

Example with 3 clients distributed across 2 cluster nodes:

```bash
vm01:
nohup wrc userid=system password=pwd mode=replay replaydir=/acfs/replay_pix DSCN_OFF=true > cliente_rat_1.out &
nohup wrc userid=system password=pwd mode=replay replaydir=/acfs/replay_pix DSCN_OFF=true > cliente_rat_2.out &

vm02:
nohup wrc userid=system password=pwd mode=replay replaydir=/acfs/replay_pix DSCN_OFF=true > cliente_rat_3.out &
```

### 6.16 Start the Replay

```sql
BEGIN
   DBMS_WORKLOAD_REPLAY.START_REPLAY;
END;
/
```

### 6.17 Monitor the Replay

```sql
col name for a30
col status for a15
set lines 200
col num_clients for 99999999

select name,
       status,
       start_time,
       sysdate,
       num_clients,
       dbtime,
       duration_secs/3600,
       user_calls
from dba_workload_replays;
```

### 6.18 Generate AWR Information for the Replay

```sql
set lines 200
col name for a32
col status for a13
select id,
       name,
       status,
       to_char(start_time,'dd-mm-yy hh24:mi:ss') start_time,
       to_char(end_time,'dd-mm-yy hh24:mi:ss') end_time,
       AWR_DBID,
       AWR_BEGIN_SNAP,
       AWR_END_SNAP
from dba_workload_replays
where AWR_END_SNAP is not null
order by 4;
```

## 7. Notes

- Replace all environment-specific values before execution.
- All hostnames, passwords, service names, restore point names, and directory names must be validated before execution.
- Flashback configuration is critical if the replay must be repeated multiple times.
