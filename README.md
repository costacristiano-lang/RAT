# RAT
RAT Capture and Replay Procedure
=================================
1. Objective
------------
This document describes the procedure to capture and replay an Oracle Real
Application Testing (RAT) workload, including AWR export and SQL Tuning Set
(STS) preparation for SQL Performance Analyzer (SPA).
2. Scope
--------
This procedure applies to RAT workload capture in the source environment and
replay preparation in the target environment.
3. Important Naming Note
------------------------
All fields highlighted in red must be reviewed and adapted to the environment
where the RAT procedure will be executed.
Examples of fields that must be reviewed include:
- _YourApp
- YOUR_TABLE
- YOUR_SCHEMA
- YOUR_TABLESPACE
- YourApp
- YOUR_APP
- nnnnn
- nnnnnn
In this specific example, the application name used is PIX. Any object names,
directories, capture names, SQL tuning set names, baseline names, and exported
file names that include PIX must be adjusted to match the application being
analyzed in your environment.
Examples of names that should be reviewed and adapted:
- CAPTURE_RAT_PIX
- capture_pix
- STS_PIX
- TABLE_STS_PIX
- PIX_baseline
- export_sts_pix.dmp
- /acfs/capture_pix
- /acfs/replay_pix
4. Prerequisites
----------------
- The source and target environments are available.
- Oracle RAT is properly licensed and configured.
- Required database privileges are granted.
- Sufficient filesystem space is available for capture and export files.
- Connectivity between source and target servers is available for file transfer.
- The users or sessions to be monitored are clearly identified.
5. Source Environment Information
---------------------------------
- Platform: ExaCC X10M-2 Gen2
- Database: p3e10c
6. Execution Steps
------------------
6.1 Create the Directory for Capture Files
Create the OS directory:
mkdir -p /acfs/capture_pix
Create the Oracle directory object:
CREATE OR REPLACE DIRECTORY capture_pix AS '/acfs/capture_pix';
GRANT ALL ON DIRECTORY capture_pix TO public;
6.2 Create Capture Filters (If Required)
If it is necessary to restrict the capture to specific users, create filters
such as:
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
6.3 Create an AWR Snapshot Before Starting the Capture
EXEC DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();
6.4 Stop Redo Apply on the Standby Database
On standby server exacc0305:
sqlplus / as sysdba
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
6.5 Start the Capture on the Source Database
On the source database:
sqlplus / as sysdba
BEGIN
   DBMS_WORKLOAD_CAPTURE.START_CAPTURE (
      name           => 'CAPTURE_RAT_PIX',
      dir            => 'CAPTURE_PIX',
      default_action => 'INCLUDE'
   );
END;
/
6.6 Monitor the Capture
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
6.7 Finish the Capture
Allow the capture to run for approximately 2 hours, then finish it:
BEGIN
   DBMS_WORKLOAD_CAPTURE.FINISH_CAPTURE;
END;
/
Validate the final status:
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
6.8 Export the AWR Data
Retrieve the capture_id:
CONNECT / AS SYSDBA
SELECT id, name
FROM dba_workload_captures;
Export AWR data for the selected capture:
BEGIN
   DBMS_WORKLOAD_CAPTURE.EXPORT_AWR(capture_id => 1);
END;
/
6.9 Create the SQL Tuning Set for SPA
Create the staging table:
BEGIN
   DBMS_SQLTUNE.CREATE_STGTAB_SQLSET(
      table_name      => 'YOUR_TABLE',
      schema_name     => 'YOUR_SCHEMA',
      tablespace_name => 'YOUR_TABLESPACE'
   );
END;
/
Create the SQL tuning set:
BEGIN
   DBMS_SQLTUNE.CREATE_SQLSET(
      sqlset_name => 'STS_YourApp',
      description => '19C STS YourApp'
   );
END;
/
Query the SQL set if required:
SELECT sql_text
FROM dba_sqlset_statements
WHERE sqlset_name = 'STS_YourApp';
6.10 Identify the AWR Snapshot Range
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
Use the snapshot IDs returned above to create the baseline:
EXECUTE DBMS_WORKLOAD_REPOSITORY.CREATE_BASELINE(
   start_snap_id => nnnnn,
   end_snap_id   => nnnnnn,
   baseline_name => 'YOUR_APP_baseline'
);
6.11 Load the Workload into the SQL Tuning Set
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
6.12 Pack the SQL Tuning Set
BEGIN
  DBMS_SQLTUNE.PACK_STGTAB_SQLSET(
     sqlset_name          => 'STS_YourApp',
     sqlset_owner         => 'SYS',
     staging_table_name   => 'YOUR_TABLE',
     staging_schema_owner => 'YOUR_SCHEMA'
  );
END;
/
6.13 Export the STS Table
exp system/pwd file=/acfs/dump/export_sts_YOUR_APP.dmp tables=YOUR_TABLE
Transfer the exported .dmp file to the target environment.
7. Target Environment Preparation
---------------------------------
- Platform: ExaCC X10M-2
- VM Cluster: exacc0305
7.1 Create the Required Directories
mkdir -p /acfs/dump
mkdir -p /acfs/replay_YourApp
7.2 Import the Exported Data
Copy the dump file to /acfs/dump, then execute:
imp system/pwd file=/acfs/dump/export_sts_YOUR_APP.dmp log=imp_YOUR_APP.log full=y
7.3 Transfer the Capture Files
cd /acfs/replay_YourApp
scp oracle@exacc01db01:/acfs/capture_YourApp .
7.4 Unpack the SQL Tuning Set
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
8. Validation
-------------
Validate that the SQL set is available in the target environment:
SELECT SQL_ID, SQL_TEXT
FROM TABLE(DBMS_SQLTUNE.SELECT_SQLSET('STS_YourApp'));
SELECT sqlset_name,
       sql_id,
       executions
FROM dba_sqlset_statements
WHERE sqlset_name = 'STS_YourApp'
ORDER BY 3 DESC;
9. Notes
--------
- Replace all placeholders such as nnnnn, nnnnnn, usernames, passwords,
  hostnames, and application-specific names before execution.
- In this example, PIX is used only as a reference application name.
- For other applications, all names and identifiers should be updated
  consistently throughout the procedure.
- It is recommended to adopt a standard naming convention aligned with the
  application or project being tested.
