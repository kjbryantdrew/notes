```sql
[oracle@cdlab01 ~]$ sqlplus / as sysdba
 
SQL*Plus: Release 11.2.0.1.0 Production on Mon Jun 25 10:23:41 2018
 
Copyright (c) 1982, 2009, Oracle.  All rights reserved.
 
 
Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.1.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SQL> select * from nls_database_parameters where parameter ='NLS_CHARACTERSET';

PARAMETER
--------------------------------------------------------------------------------
VALUE
--------------------------------------------------------------------------------
NLS_CHARACTERSET
ZHS16GBK

SQL> shutdown immediate;
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup mount;
ORACLE instance started.
 
Total System Global Area 5.3982E+10 bytes
Fixed Size                  2218032 bytes
Variable Size            2.6575E+10 bytes
Database Buffers         2.7380E+10 bytes
Redo Buffers               24133632 bytes
Database mounted.
SQL> alter system enable restricted session;
 
System altered.
SQL> alter system set job_queue_processes=0;
 
System altered.
 
SQL> alter system set aq_tm_processes=0;
 
System altered.
 
SQL> alter database open;
 
Database altered.
 
SQL> alter database character set internal_use utf8;
 
Database altered.
 
SQL> shutdown immediate;
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup;
ORACLE instance started.
```
