# GoldenGate-Oracle2Oracle-RemoteCapture
Oracle DB Source (without GG services) -> Oracle GoldenGate -> Oracle DB Target (without GG services)

![Alt text](Diagram.png?raw=true "Diagram")  

# 0. Prerequisites

## DB credentials
```console
user@bash:~$ sqlplus system/oracle@datasource:1521/xe  
```
```console
user@bash:~$ sqlplus system/oracle@datatarget:1521/xe 
```

## Used Docker images
* oracle/goldengate-standard:12.3.0.1.4 (Read here https://github.com/oracle/docker-images/tree/master/OracleGoldenGate)  
* sath89/oracle-12c

## Docker consoles for Terminal
> docker exec -it GG-datasource bash  
> docker exec -it GG-goldengate bash  
> docker exec -it GG-datatarget bash

# 1. Oracle DB Source Init

## Database params
```splus
alter system set enable_goldengate_replication=TRUE;  
alter database add supplemental log data;  
alter database force logging;  
alter system switch logfile;  
```

## Enabling ARCHIVELOG mode

```console
su oracle
/u01/app/oracle/product/12.1.0/xe/bin/sqlplus /nolog
connect / as sysdba
SQL> shutdown immediate;
SQL> Startup mount;
SQL> Alter database archivelog;
SQL> alter database open;
```

### Check archive mode and where archive logs are stored.

```console
/u01/app/oracle/product/12.1.0/xe/bin/sqlplus /nolog
SQL> connect / as sysdba
SQL> archive log list;
SQL> select dest_name, status, destination from v$archive_dest;
SQL> show parameter db_recovery_file_dest
```

## Credentials

### OGG user
```sql
CREATE USER gg_extract IDENTIFIED BY gg_extract;  
GRANT CREATE SESSION, CONNECT, RESOURCE, ALTER ANY TABLE, ALTER SYSTEM, DBA, SELECT ANY TRANSACTION TO gg_extract;
```

### Transaction user
```sql
CREATE USER trans_user IDENTIFIED BY trans_user;  
GRANT CREATE SESSION, CONNECT, RESOURCE TO trans_user;  
ALTER USER trans_user QUOTA UNLIMITED ON USERS;
```

## Data source objects
```sql
CREATE TABLE trans_user.test (  
         empno      NUMBER(5) PRIMARY KEY,  
         ename      VARCHAR2(15) NOT NULL);  


 COMMENT ON TABLE test IS 'Testing GoldenGate';
```

# 2. Oracle DB Target Init

```sql
alter system set enable_goldengate_replication=TRUE;
```

## Credentials

### OGG user
```sql
CREATE USER gg_replicat IDENTIFIED BY gg_replicat;  
GRANT CREATE SESSION, CONNECT, RESOURCE, CREATE TABLE, DBA, LOCK ANY TABLE TO GG_REPLICAT;
```

### Replication user
```sql
CREATE USER trans_user IDENTIFIED BY trans_user;  
GRANT CREATE SESSION, CONNECT, RESOURCE TO trans_user;  
ALTER USER trans_user QUOTA UNLIMITED ON USERS;
```

## Data target objects

```sql
CREATE TABLE trans_user.test (
         empno        NUMBER(5),
         ename        VARCHAR2(15),
         gg_ts        TIMESTAMP,
         gg_OP_TYPE   CHAR(1) NULL,
         gg_RSN       number,
         BEFORE_AFTER CHAR(1) NULL
);
```

# 3. Oracle GoldenGate Init

## Credentials of sources

### Connect as oracle to GoldenGate instance and run GGSCI:
```console
user@bash:~$ su oracle
user@bash:~$ cd /u01/app/ogg
user@bash:~$ ggsci
```

### Run GGSCI:
```console
GGSCI@(a3abfded7bc7):~$ add credentialstore  
Credential store created.  
GGSCI@(a3abfded7bc7):~$ alter credentialstore add user gg_extract@datasource:1521/xe password gg_extract alias oggadmin  
Credential store altered.  
GGSCI@(a3abfded7bc7):~$ alter credentialstore add user gg_replicat@datatarget:1521/xe password gg_replicat alias oggrepl  
Credential store altered.  
GGSCI@(a3abfded7bc7):~$ info credentialstore  
Reading from credential store:  
Default domain: OracleGoldenGate  
  Alias: oggadmin  
  Userid: gg_extract@datasource:1521/xe
```

## Change metadata in source

### Connect as oracle to GoldenGate instance:
> su oracle  

### Run GGSCI and change source schema configuration:
> **GGSCI (a3abfded7bc7) 2>** dblogin useridalias oggadmin  
Successfully logged into database.  
> **GGSCI (a3abfded7bc7) 2>** add schematrandata trans_user ALLCOLS  
2018-10-09 10:48:56  INFO    OGG-01788  SCHEMATRANDATA has been added on schema "trans_user".  

### Run GGSCI and add checkpoint table:
> **GGSCI (a3abfded7bc7) 2>** dblogin useridalias oggrepl  
Successfully logged into database.  
> **GGSCI (a3abfded7bc7) 2>** ADD CHECKPOINTTABLE gg_replicat.oggchkpt  
Successfully created checkpoint table gg_replicat.oggchkpt.  



# 3. Oracle GoldenGate - Configure replication

## Extract configuration

### Connect as oracle to GoldenGate instance:
> su oracle  

### Run GGSCI and edit extract params file(e.g. VIM will be runned):
> **GGSCI (a3abfded7bc7) 2>** edit params getExt  
```
EXTRACT getExt
USERIDALIAS oggadmin
LOGALLSUPCOLS
TRANLOGOPTIONS EXCLUDEUSER gg_extract
TRANLOGOPTIONS DBLOGREADER
EXTTRAIL ./dirdat/in
TABLE trans_user.test;
```

### Run GGSCI and register&start extract params file:
> **GGSCI (a3abfded7bc7) 2>** ADD EXTRACT getExt, TRANLOG, BEGIN NOW  
> **GGSCI (a3abfded7bc7) 2>** ADD EXTTRAIL ./dirdat/in, EXTRACT getext  
> **GGSCI (a3abfded7bc7) 2>** START EXTRACT getExt  
> **GGSCI (a3abfded7bc7) 2>** info extract getext, detail  

## Replicat configuration

### Run GGSCI and edit replicat params file(e.g. VIM will be runned):
> **GGSCI (a3abfded7bc7) 2>** edit params putExt  
```
REPLICAT putext
INSERTALLRECORDS
USERIDALIAS oggrepl
DISCARDFILE ./dirrpt/putext.dsc, Purge
MAP trans_user.*, TARGET trans_user.*,
COLMAP (
 USEDEFAULTS,
 gg_TS = @GETENV('GGHEADER','COMMITTIMESTAMP'),
 gg_OP_TYPE = @CASE(@GETENV('GGHEADER', 'OPTYPE'), 'INSERT', 'I', 'UPDATE', 'U',
  'PK UPDATE', 'P', 'SQL COMPUPDATE', 'S', 'ENSCRIBE COMPUPDATE', 'E',
  'DELETE', 'D', 'TRUNCATE', 'T', '?'),
 gg_RSN = @GETENV('ORATRANSACTION','SCN'),
 BEFORE_AFTER = @GETENV ('GGHEADER', 'BEFOREAFTERINDICATOR')
       );
```

### Run GGSCI and register&start replicat params file:
> **GGSCI (a3abfded7bc7) 2>** ADD REPLICAT putExt, EXTTRAIL ./dirdat/in, BEGIN NOW, CHECKPOINTTABLE gg_replicat.oggchkpt  
> **GGSCI (a3abfded7bc7) 2>** START REPLICAT putExt  

# 4. Oracle GoldenGate - Emulate replication

## Insert - Source
```sql
insert into trans_user.test(empno, ename) 
select max(empno)+1, max(ename) from trans_user.test;
commit;
```

```console
SQL> begin
  2  for x in 1 .. 1000000 loop
  3  insert into trans_user.test(empno, ename)
  4  select max(empno)+1, max(ename) from trans_user.test;
  5  commit;
  6  end loop;
  7  end;
  8  /
```

## Update - Source
```sql
update trans_user.test
set ename='so'
where empno=1
```

## Result - Target
| EMPNO | ENAME | GG_TS               | GG_OP_TYPE | GG_SCN  | BEFOREAFTER |
|-------|-------|---------------------|------------|---------|-------------|
| 1     | ok    | 2018-10-11 11:19:15 | I          | 2064051 | A           |
| 1     | ok    | 2018-10-11 11:19:16 | S          | 2064052 | A           |
| 1     | ko    | 2018-10-11 11:19:16 | S          | 2064052 | B           |

# Advanced topics

## Case 1. Using different version of Oracle for source and target
Read about DEFGEN utility

## Case 2. Automate Target schema creation
I did'nt find any ready solution

## Case 3. Downstream configuration
Used for decreasing source performance impact
