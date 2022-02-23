> 新核心UAT3环境统一采用单生产节点，UAT4环境统一采用RAC+CDB+ADG架构，而正式的生产环境将交易库和管理库（主库中有需要同步的柜面库）做了架构拆分，交易库采用单主节点+独立的adg节点的模式，管理库采用cdb+独立adg节点的模式。在该模式下，预设部署方案为：将ogg源端程序部署在交易库adg节点，同时在adg的Oracle下添加对管理库的adg节点的监听，远程抽取管理库日志。

# 1. 源交易库

## 1.1 开启数据库日志

**开启归档日志需要在主库中操作**

```sql
-- 登录主库oracle
[core01:oracle]:/home/oracle>sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Wed Feb 23 09:52:01 2022
Version 19.9.0.0.0

Copyright (c) 1982, 2020, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.9.0.0.0

-- 查看归档日志状态
SQL> archive log list
Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     1106
Next log sequence to archive   1109
Current log sequence           1109

-- 如果归档日志没有打开，需要打开归档日志
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup mount
ORACLE instance started.
 
SQL> alter database archivelog;
 
Database altered.
-- 打开数据库
SQL> alter database open;

-- 打开force logging
SQL> alter database add supplemental log data;
SQL> alter system switch logfile;
SQL> alter database force logging;
SQL> select log_mode,force_logging from v$database;

LOG_MODE     FORCE_LOGGING
------------ ---------------------------------------
ARCHIVELOG   YES
```

## 1.2 添加oracle golden gate支持

```sql
SQL> show parameter enable_goldengate_replication;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
enable_goldengate_replication        boolean     FALSE
-- 打开ogg支持
SQL> alter system set enable_goldengate_replication=true scope=both sid='*';

System altered.
-- 再次确认是否开启成功
SQL> show parameter enable_goldengate_replication;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
enable_goldengate_replication        boolean     TRUE
```

## 1.3 添加ogg用户

```sql
-- 创建表空间
SQL>create tablespace ogg_tbs datafile '+DATA/COREDB/DATAFILE/ogg_tbs.dbf' size 5G AUTOEXTEND on extent management local segment space management auto

-- 创建用户
SQL> create user ggadm identified by "oracle" default tablespace ogg_tbs;

User created.

SQL> ALTER USER GGADM QUOTA UNLIMITED ON  OGG_TBS;

User altered.

SQL> grant dba,connect,resource,CREATE SESSION to ggadm;

Grant succeeded.

SQL> exec dbms_goldengate_auth.grant_admin_privilege('ggadm');

PL/SQL procedure successfully completed.

SQL> exec dbms_goldengate_auth.grant_admin_privilege(grantee=>'ggadm');

PL/SQL procedure successfully completed.

SQL> grant select any dictionary to ggadm;

Grant succeeded.

SQL> commit;

Commit complete.
```

# 2. 源交易库ogg安装

**ogg安装在adg节点**

## 2.1 新增用户

```bash
# 在root用户下创建ggs用户，并和oracle用户在同一个用户组
useradd ggs -g oinstall
```

## 2.2 ogg安装

### 2.2.1 安装包准备

```bash
# 将对应的ogg版本上传到服务器ggs用户根目录下并解压
[ggs@core_dg ~]$ unzip 191004_fbo_ggs_Linux_s390x_shiphome.zip
[ggs@core_dg ~]$ ls
191004_fbo_ggs_Linux_s390x_shiphome.zip  fbo_ggs_Linux_s390x_shiphome  OGG-19.1.0.0-README.txt  OGG_WinUnix_Rel_Notes_19.1.0.0.4.pdf
```

### 2.2.2 修改配置文件

```bash
# 修改相应文件
[ggs@core_dg ~]$ cd fbo_ggs_Linux_s390x_shiphome/Disk1/response/
[ggs@core_dg response]$ ls
oggcore.rsp
[ggs@core_dg response]$ vim oggcore.rsp

# 打开文件后修改下述几个配置
INSTALL_OPTION=ORA19c
SOFTWARE_LOCATION=/home/ggs/ogg
START_MANAGER=false
```

### 2.2.3 安装

```bash
# 创建空目录，路径与前面修改的oggcore.rsp文件中的SOFTWARE_LOCATION一致
[ggs@core_dg ~]$ mkdir -p /home/ggs/ogg

# 执行安装程序
[ggs@core_dg ~]$ cd fbo_ggs_Linux_s390x_shiphome/Disk1/
[ggs@core_dg Disk1]$ ./runInstaller -silent -responseFile /home/ggs/fbo_ggs_Linux_s390x_shiphome/Disk1/response/oggcore.rsp
正在启动 Oracle Universal Installer...

检查临时空间: 必须大于 120 MB。   实际为 113882 MB    通过
检查交换空间: 必须大于 150 MB。   实际为 32721 MB    通过
准备从以下地址启动 Oracle Universal Installer /tmp/OraInstall2022-02-23_10-54-33AM. 请稍候...[ggs@core_dg Disk1]$ 可以在以下位置找到本次安装会话的日志:
 /u01/app/oraInventory/logs/installActions2022-02-23_10-54-33AM.log
Successfully Setup Software.
Oracle GoldenGate Core 的 安装 已成功。
请查看 '/u01/app/oraInventory/logs/silentInstall2022-02-23_10-54-33AM.log' 以获取详细资料。
# 有如上所示的“安装已成功”的字样表示安装成功
```

### 2.2.4 配置系统环境变量

```bash
[ggs@core_dg ~]$ vim ogg.env

# 在文件中增加一下内容(ORACLE_BASE、ORACLE_HOME、ORACLE_SID需要根据具体oracle配置调整)

PATH=$PATH:$HOME/.local/bin:$HOME/bin
export PATH
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.3.0/dbhome_1
export ORACLE_SID=coredb
export NLS_LANG=AMERICAN_AMERICA.UTF8
export ORACLE_TERM=xterm
export PATH=$ORACLE_HOME/bin:/usr/sbin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export LANG=C
export OGG_HOME=/home/ggs/ogg
export PATH=$OGG_HOME:$PATH
export LD_LIBRARY_PATH=$OGG_HOME:$LD_LIBRARY_PATH

# 使环境变量生效
[ggs@core_dg ~]$ vim .bashrc

# 添加如下内容
source ogg.env
# 保存退出

[ggs@core_dg ~]$ source .bashrc
```

# 3. 源交易库ogg配置

## 3.1 ogg子文件夹

```bash
[ggs@core_dg ~]$ ggsci

Oracle GoldenGate Command Interpreter for Oracle
Version 19.1.0.0.4 OGGCORE_19.1.0.0.0_PLATFORMS_191017.1054_FBO
Linux, s390x, 64bit (optimized), Oracle 19c on Oct 19 2019 09:41:48
Operating system character set identified as US-ASCII.

Copyright (C) 1995, 2019, Oracle and/or its affiliates. All rights reserved.



GGSCI (core_dg) 1> create subdirs

Creating subdirectories under current directory /home/ggs

Parameter file                 /home/ggs/ogg/dirprm: created.
Report file                    /home/ggs/ogg/dirrpt: created.
Checkpoint file                /home/ggs/ogg/dirchk: created.
Process status files           /home/ggs/ogg/dirpcs: created.
SQL script files               /home/ggs/ogg/dirsql: created.
Database definitions files     /home/ggs/ogg/dirdef: created.
Extract data files             /home/ggs/ogg/dirdat: created.
Temporary files                /home/ggs/ogg/dirtmp: created.
Credential store files         /home/ggs/ogg/dircrd: created.
Masterkey wallet files         /home/ggs/ogg/dirwlt: created.
Dump files                     /home/ggs/ogg/dirdmp: created.
```

## 3.2 CredentialStore认证

用证书可以免密登录Oracle

```bash
GGSCI (core_dg) 1> add credentialstore

Credential store created.

GGSCI (core_dg) 2> alter credentialstore add user ggadm@coredb password oracle alias ogg_hx
# ggadm/oracle为1.3章节中在主库上创建的ogg用户名和密码
# 特别注意：“@coredb”中，coredb为dg节点的Oracle监听程序中配置的对主库的监听服务名，从$ORACLE_HOME/network/admin/tnsnames.ora中获取到的
Credential store altered.

GGSCI (core_dg) 3> dblogin useridalias ogg_hx
Successfully logged into database.
```

## 3.3 添加表的附加日志

```shell
# 先登录
GGSCI (core_dg) 2> dblogin useridalias ogg_hx
Successfully logged into database.

# 批量添加
add trandata ens_cbank.ac_subject
add trandata ens_cbank.cd_card_arch
add trandata ens_cbank.cd_card_chg
add trandata ens_cbank.cif_business
add trandata ens_cbank.cif_category_type
add trandata ens_cbank.cif_class_4
add trandata ens_cbank.cif_class_5
add trandata ens_cbank.cif_client
add trandata ens_cbank.cif_client_contact_address
add trandata ens_cbank.cif_client_contact_tbl
add trandata ens_cbank.cif_client_corp
add trandata ens_cbank.cif_client_document
add trandata ens_cbank.cif_client_indvl
add trandata ens_cbank.cif_client_status
add trandata ens_cbank.cif_client_type
add trandata ens_cbank.cif_cross_relations
add trandata ens_cbank.cif_document_type
add trandata ens_cbank.cif_education
add trandata ens_cbank.cif_industry
add trandata ens_cbank.cif_occupation
add trandata ens_cbank.cif_relation_type
add trandata ens_cbank.cif_resident_type
add trandata ens_cbank.dc_precontract
add trandata ens_cbank.dc_stage_define
add trandata ens_cbank.fm_branch
add trandata ens_cbank.fm_channel
add trandata ens_cbank.fm_city
add trandata ens_cbank.fm_country
add trandata ens_cbank.fm_currency
add trandata ens_cbank.fm_dist_code
add trandata ens_cbank.fm_loc_holiday
add trandata ens_cbank.fm_period_freq
add trandata ens_cbank.fm_ref_code
add trandata ens_cbank.fm_restraint_type
add trandata ens_cbank.fm_state
add trandata ens_cbank.fm_tran_info
add trandata ens_cbank.fm_user
add trandata ens_cbank.gl_prod_accounting
add trandata ens_cbank.irl_ccy_rate
add trandata ens_cbank.irl_exchange_type
add trandata ens_cbank.irl_int_basis
add trandata ens_cbank.irl_int_matrix
add trandata ens_cbank.irl_int_rate
add trandata ens_cbank.irl_int_type
add trandata ens_cbank.irl_prod_int
add trandata ens_cbank.irl_prod_type
add trandata ens_cbank.mb_accounting_status
add trandata ens_cbank.mb_acct
add trandata ens_cbank.mb_acct_attach
add trandata ens_cbank.mb_acct_balance
add trandata ens_cbank.mb_acct_int_detail
add trandata ens_cbank.mb_acct_schedule_detail
add trandata ens_cbank.mb_ad_register
add trandata ens_cbank.mb_agreement
add trandata ens_cbank.mb_agreement_accord
add trandata ens_cbank.mb_agreement_sms
add trandata ens_cbank.mb_agreement_sxc
add trandata ens_cbank.mb_attr_value
add trandata ens_cbank.mb_batch_open_details
add trandata ens_cbank.mb_cdt_info
add trandata ens_cbank.mb_certificate_type
add trandata ENS_CBANK.mb_commission_register
add trandata ens_cbank.mb_debt_asset
add trandata ens_cbank.mb_debt_asset_loan
add trandata ens_cbank.mb_debt_asset_hist
add trandata ens_cbank.mb_fin_detail
add trandata ENS_CBANK.mb_impound_info
add trandata ens_cbank.mb_invoice
add trandata ens_cbank.mb_materials_type
add trandata ens_cbank.mb_open_close_reg
add trandata ens_cbank.mb_prod_type
add trandata ens_cbank.mb_prod_define
add trandata ens_cbank.mb_receipt
add trandata ens_cbank.mb_receipt_detail
add trandata ens_cbank.mb_restraints
add trandata ens_cbank.mb_stage_define
add trandata ens_cbank.mb_stage_matrix
add trandata ens_cbank.mb_tda_hist
add trandata ens_cbank.mb_tran_def
add trandata ens_cbank.mb_tran_hist
add trandata ens_cbank.mb_transfer_contract
add trandata ens_cbank.mb_transfer_detail
add trandata ens_cbank.mb_xfc_fin
add trandata ens_cbank.mb_xfc_stage_define
add trandata ens_cbank.mm_interbank_busi_reg
add trandata ens_cbank.mm_interbank_repay_detail
add trandata ens_cbank.nx_rc_cust_list_mg
add trandata ens_cbank.stc_acct
add trandata ens_cbank.tb_cash_balance
add trandata ens_cbank.tb_cash_journal
add trandata ens_cbank.tb_cash_move
add trandata ens_cbank.tb_cash_move_detail
add trandata ens_cbank.tb_trailbox
add trandata ens_cbank.tb_voucher_def
add trandata ens_cbank.tb_voucher_info
add trandata ens_cbank.fm_system
add trandata ens_cbank.rc_all_list
add trandata ens_cbank.fm_settlement_type
add trandata ens_cbank.mb_event_type
add trandata ens_cbank.gl_event_type
add trandata ens_cbank.rc_list_type
add trandata ens_cbank.rc_list_category
add trandata ens_cbank.fm_system_id
add trandata ens_cbank.mb_event_part
add trandata ens_cbank.cd_card_journal
add trandata ens_cbank.mb_ac_hist
add trandata ens_cbank.mb_batch_tran
add trandata ens_cbank.mb_batch_transfer_details
add trandata ens_cbank.cif_client_verification
add trandata ens_cbank.fx_mb_exchange_tran_hist
add trandata ens_cbank.gl_acct_type
add trandata ens_cbank.mb_acct_discount
add trandata ens_cbank.mb_seal_relation
add trandata ens_cbank.mb_agreement_yht
add trandata ens_cbank.mb_payinfo
add trandata ens_cbank.irl_capt_info
add trandata ens_cbank.irl_accr_info_main
add trandata ens_cbank.irl_int_split
add trandata ens_cbank.gl_amount_type
add trandata ens_cbank.mb_acct_settle
add trandata ens_cbank.mb_acct_nature_def
add trandata ens_cbank.mb_launch
add trandata ens_cbank.mb_launch_detail
add trandata ens_cbank.mb_reason_code
add trandata ens_cbank.cif_qualification
add trandata ens_cbank.mb_acct_hist
add trandata ens_cbank.mb_nocard_file
add trandata ens_cbank.cif_document_read
add trandata ens_cbank.lm_limit_cumulative
add trandata ens_cbank.lm_tran_limit_def
add trandata ens_cbank.stria_flow
add trandata ens_cbank.mb_sxc_tran_hist
```

## 3.4 添加检查点

```bash
GGSCI (core_dg as ggadm@coredb2) 135> add checkpointtable ggadm.checkpoint

Successfully created checkpoint table ggadm.checkpoint.
```

## 3.5 生成表结构定义文件

### 3.5.1 编辑参数文件

```bash
GGSCI (core_dg) 1> edit param def_hx

# 添加如下内容

USERIDALIAS ogg_hx
defsfile ./dirdef/def_hx.def PURGE

table ens_cbank.ac_subject;
table ens_cbank.cd_card_arch;
table ens_cbank.cd_card_chg;
table ens_cbank.cif_business;
table ens_cbank.cif_category_type;
table ens_cbank.cif_class_4;
table ens_cbank.cif_class_5;
table ens_cbank.cif_client;
table ens_cbank.cif_client_contact_address;
table ens_cbank.cif_client_contact_tbl;
table ens_cbank.cif_client_corp;
table ens_cbank.cif_client_document;
table ens_cbank.cif_client_indvl;
table ens_cbank.cif_client_status;
table ens_cbank.cif_client_type;
table ens_cbank.cif_cross_relations;
table ens_cbank.cif_document_type;
table ens_cbank.cif_education;
table ens_cbank.cif_industry;
table ens_cbank.cif_occupation;
table ens_cbank.cif_relation_type;
table ens_cbank.cif_resident_type;
table ens_cbank.dc_precontract;
table ens_cbank.dc_stage_define;
table ens_cbank.fm_branch;
table ens_cbank.fm_channel;
table ens_cbank.fm_city;
table ens_cbank.fm_country;
table ens_cbank.fm_currency;
table ens_cbank.fm_dist_code;
table ens_cbank.fm_loc_holiday;
table ens_cbank.fm_period_freq;
table ens_cbank.fm_ref_code;
table ens_cbank.fm_restraint_type;
table ens_cbank.fm_state;
table ens_cbank.fm_tran_info;
table ens_cbank.fm_user;
table ens_cbank.gl_prod_accounting;
table ens_cbank.irl_ccy_rate;
table ens_cbank.irl_exchange_type;
table ens_cbank.irl_int_basis;
table ens_cbank.irl_int_matrix;
table ens_cbank.irl_int_rate;
table ens_cbank.irl_int_type;
table ens_cbank.irl_prod_int;
table ens_cbank.irl_prod_type;
table ens_cbank.mb_accounting_status;
table ens_cbank.mb_acct;
table ens_cbank.mb_acct_attach;
table ens_cbank.mb_acct_balance;
table ens_cbank.mb_acct_int_detail;
table ens_cbank.mb_acct_schedule_detail;
table ens_cbank.mb_ad_register;
table ens_cbank.mb_agreement;
table ens_cbank.mb_agreement_accord;
table ens_cbank.mb_agreement_sms;
table ens_cbank.mb_agreement_sxc;
table ens_cbank.mb_attr_value;
table ens_cbank.mb_batch_open_details;
table ens_cbank.mb_cdt_info;
table ens_cbank.mb_certificate_type;
table ens_cbank.mb_commission_register;
table ens_cbank.mb_debt_asset;
table ens_cbank.mb_debt_asset_loan;
table ens_cbank.mb_debt_asset_hist;
table ens_cbank.mb_fin_detail;
table ens_cbank.mb_impound_info;
table ens_cbank.mb_invoice;
table ens_cbank.mb_materials_type;
table ens_cbank.mb_open_close_reg;
table ens_cbank.mb_prod_type;
table ens_cbank.mb_receipt;
table ens_cbank.mb_receipt_detail;
table ens_cbank.mb_restraints;
table ens_cbank.mb_stage_define;
table ens_cbank.mb_stage_matrix;
table ens_cbank.mb_tda_hist;
table ens_cbank.mb_tran_def;
table ens_cbank.mb_tran_hist;
table ens_cbank.mb_transfer_contract;
table ens_cbank.mb_transfer_detail;
table ens_cbank.mb_xfc_fin;
table ens_cbank.mb_xfc_stage_define;
table ens_cbank.mm_interbank_busi_reg;
table ens_cbank.mm_interbank_repay_detail;
table ens_cbank.stc_acct;
table ens_cbank.tb_cash_balance;
table ens_cbank.tb_cash_journal;
table ens_cbank.tb_cash_move;
table ens_cbank.tb_cash_move_detail;
table ens_cbank.tb_trailbox;
table ens_cbank.tb_voucher_def;
table ens_cbank.tb_voucher_info;
table ens_cbank.fm_system;
table ens_cbank.rc_all_list;
table ens_cbank.fm_settlement_type;
table ens_cbank.mb_event_type;
table ens_cbank.gl_event_type;
table ens_cbank.rc_list_type;
table ens_cbank.rc_list_category;
table ens_cbank.fm_system_id;
table ens_cbank.mb_event_part;
table ens_cbank.cd_card_journal;
table ens_cbank.mb_ac_hist;
table ens_cbank.mb_batch_tran;
table ens_cbank.mb_batch_transfer_details;
table ens_cbank.cif_client_verification;
table ens_cbank.fx_mb_exchange_tran_hist;
table ens_cbank.gl_acct_type;
table ens_cbank.mb_acct_discount;
table ens_cbank.mb_seal_relation;
table ens_cbank.mb_agreement_yht;
table ens_cbank.mb_prod_define;
table ens_cbank.mb_payinfo;
table ens_cbank.irl_capt_info;
table ens_cbank.irl_accr_info_main;
table ens_cbank.irl_int_split;
table ens_cbank.gl_amount_type;
table ens_cbank.mb_acct_settle;
table ens_cbank.mb_acct_nature_def;
table ens_cbank.mb_launch;
table ens_cbank.mb_launch_detail;
table ens_cbank.mb_reason_code;
table ens_cbank.cif_qualification;
table ens_cbank.mb_acct_hist;
table ens_cbank.mb_nocard_file;
table ens_cbank.cif_document_read;
table ens_cbank.lm_limit_cumulative;
table ens_cbank.lm_tran_limit_def;
table ens_cbank.stria_flow;
table ens_cbank.mb_sxc_tran_hist;
```

### 3.5.2 生成表结构定义文件

```bash
[ggs@core_dg ogg]$ ./defgen paramfile ./dirprm/def_hx.prm NOEXTATTR

# 将配置文件送到目标服务器
[ggs@core_dg ogg]$ scp dirdef/def_hx.def ogg12@160.161.12.43:~/dirdef/
```



## 3.6 manager配置

```bash
# 启动manger需要在OGG_HOME目录，否则会报错
[ggs@core_dg ogg]$ cd $OGG_HOME
[ggs@core_dg ogg]$ ggsci

GGSCI (core_dg) 1> edit param mgr

# 添加如下内容
PORT 7809
DYNAMICPORTLIST 7810-7820
AUTORESTART ER *,RETRIES 3,WAITMINUTES 3, RESETMINUTES 60
PURGEOLDEXTRACTS ./dirdat/*,usecheckpoints, minkeepdays 3
LAGREPORTHOURS 1
LAGINFOMINUTES 30
LAGCRITICALMINUTES 45
```



## 3.7 extract配置

```bash
GGSCI (core_dg) 2> add extract exthx, INTEGRATED tranlog, begin now
EXTRACT (Integrated) added.


GGSCI (core_dg) 3> add exttrail ./dirdat/hx, extract exthx, megabytes 200
EXTTRAIL added.

GGSCI (core_dg) 4> edit params exthx

# 添加如下内容
extract exthx
setenv (NLS_LANG=AMERICAN_AMERICA.UTF8)
USERIDALIAS ogg_hx
FETCHOPTIONS NOUSESNAPSHOT
GETTRUNCATES
EXTTRAIL ./dirdat/hx,format release 12.3
DISCARDFILE ./dirrpt/hx.dsc, APPEND, MEGABYTES 4000
WARNLONGTRANS 1H, CHECKINTERVAL 5M
CACHEMGR CACHESIZE 1024MB, CACHEDIRECTORY ./dirtmp

GETUPDATEBEFORES
LOGALLSUPCOLS
NOCOMPRESSUPDATES
UPDATERECORDFORMAT FULL

REPORTCOUNT EVERY 60 SECONDS, RATE

table ens_cbank.ac_subject;
table ens_cbank.cd_card_arch;
table ens_cbank.cd_card_chg;
table ens_cbank.cif_business;
table ens_cbank.cif_category_type;
table ens_cbank.cif_class_4;
table ens_cbank.cif_class_5;
table ens_cbank.cif_client;
table ens_cbank.cif_client_contact_address;
table ens_cbank.cif_client_contact_tbl;
table ens_cbank.cif_client_corp;
table ens_cbank.cif_client_document;
table ens_cbank.cif_client_indvl;
table ens_cbank.cif_client_status;
table ens_cbank.cif_client_type;
table ens_cbank.cif_cross_relations;
table ens_cbank.cif_document_type;
table ens_cbank.cif_education;
table ens_cbank.cif_industry;
table ens_cbank.cif_occupation;
table ens_cbank.cif_relation_type;
table ens_cbank.cif_resident_type;
table ens_cbank.dc_precontract;
table ens_cbank.dc_stage_define;
table ens_cbank.fm_branch;
table ens_cbank.fm_channel;
table ens_cbank.fm_city;
table ens_cbank.fm_country;
table ens_cbank.fm_currency;
table ens_cbank.fm_dist_code;
table ens_cbank.fm_loc_holiday;
table ens_cbank.fm_period_freq;
table ens_cbank.fm_ref_code;
table ens_cbank.fm_restraint_type;
table ens_cbank.fm_state;
table ens_cbank.fm_tran_info;
table ens_cbank.fm_user;
table ens_cbank.gl_prod_accounting;
table ens_cbank.irl_ccy_rate;
table ens_cbank.irl_exchange_type;
table ens_cbank.irl_int_basis;
table ens_cbank.irl_int_matrix;
table ens_cbank.irl_int_rate;
table ens_cbank.irl_int_type;
table ens_cbank.irl_prod_int;
table ens_cbank.irl_prod_type;
table ens_cbank.mb_accounting_status;
table ens_cbank.mb_acct;
table ens_cbank.mb_acct_attach;
table ens_cbank.mb_acct_balance;
table ens_cbank.mb_acct_int_detail;
table ens_cbank.mb_acct_schedule_detail;
table ens_cbank.mb_ad_register;
table ens_cbank.mb_agreement;
table ens_cbank.mb_agreement_accord;
table ens_cbank.mb_agreement_sms;
table ens_cbank.mb_agreement_sxc;
table ens_cbank.mb_attr_value;
table ens_cbank.mb_batch_open_details;
table ens_cbank.mb_cdt_info;
table ens_cbank.mb_certificate_type;
table ens_cbank.mb_commission_register;
table ens_cbank.mb_debt_asset;
table ens_cbank.mb_debt_asset_loan;
table ens_cbank.mb_debt_asset_hist;
table ens_cbank.mb_fin_detail;
table ens_cbank.mb_impound_info;
table ens_cbank.mb_invoice;
table ens_cbank.mb_materials_type;
table ens_cbank.mb_open_close_reg;
table ens_cbank.mb_prod_type;
table ens_cbank.mb_receipt;
table ens_cbank.mb_receipt_detail;
table ens_cbank.mb_restraints;
table ens_cbank.mb_stage_define;
table ens_cbank.mb_stage_matrix;
table ens_cbank.mb_tda_hist;
table ens_cbank.mb_tran_def;
table ens_cbank.mb_tran_hist;
table ens_cbank.mb_transfer_contract;
table ens_cbank.mb_transfer_detail;
table ens_cbank.mb_xfc_fin;
table ens_cbank.mb_xfc_stage_define;
table ens_cbank.mm_interbank_busi_reg;
table ens_cbank.mm_interbank_repay_detail;
table ens_cbank.stc_acct;
table ens_cbank.tb_cash_balance;
table ens_cbank.tb_cash_journal;
table ens_cbank.tb_cash_move;
table ens_cbank.tb_cash_move_detail;
table ens_cbank.tb_trailbox;
table ens_cbank.tb_voucher_def;
table ens_cbank.tb_voucher_info;
table ens_cbank.fm_system;
table ens_cbank.rc_all_list;
table ens_cbank.fm_settlement_type;
table ens_cbank.mb_event_type;
table ens_cbank.gl_event_type;
table ens_cbank.rc_list_type;
table ens_cbank.rc_list_category;
table ens_cbank.fm_system_id;
table ens_cbank.mb_event_part;
table ens_cbank.cd_card_journal;
table ens_cbank.mb_ac_hist;
table ens_cbank.mb_batch_tran;
table ens_cbank.mb_batch_transfer_details;
table ens_cbank.cif_client_verification;
table ens_cbank.fx_mb_exchange_tran_hist;
table ens_cbank.gl_acct_type;
table ens_cbank.mb_acct_discount;
table ens_cbank.mb_seal_relation;
table ens_cbank.mb_agreement_yht;
table ens_cbank.mb_prod_define;
table ens_cbank.mb_payinfo;
table ens_cbank.irl_capt_info;
table ens_cbank.irl_accr_info_main;
table ens_cbank.irl_int_split;
table ens_cbank.gl_amount_type;
table ens_cbank.mb_acct_settle;
table ens_cbank.mb_acct_nature_def;
table ens_cbank.mb_launch;
table ens_cbank.mb_launch_detail;
table ens_cbank.mb_reason_code;
table ens_cbank.cif_qualification;
table ens_cbank.mb_acct_hist;
table ens_cbank.mb_nocard_file;
table ens_cbank.cif_document_read;
table ens_cbank.lm_limit_cumulative;
table ens_cbank.lm_tran_limit_def;
table ens_cbank.stria_flow;
table ens_cbank.mb_sxc_tran_hist;
```

## 3.8 pump配置

```bash
GGSCI (core_dg) 7> add extract pphx, exttrailsource ./dirdat/hx
EXTRACT added.


GGSCI (core_dg) 8> add rmttrail /home/ogg12/dirdat/jy, extract pphx
RMTTRAIL added.

GGSCI (core_dg) 9> edit params pphx

# 添加如下内容
EXTRACT pphx
setenv (NLS_LANG=AMERICAN_AMERICA.UTF8)
USERIDALIAS ogg_hx
discardfile ./dirrpt/pphx.dsc,append,megabytes 4000
rmthost 160.161.12.43 mgrport 7809
rmttrail /home/ogg12/dirdat/jy

PASSTHRU
table ens_cbank.*;
```

## 3.9 注册数据库

```bash
# ogg命令行登录oracle
GGSCI (core_dg) 20> dblogin USERIDALIAS ogg_hx
Successfully logged into database.

# 注册数据库
GGSCI (core_dg as ggadm@coredb2) 21> register extract exthx database
```

## 3.10 启动

```bash
# 启动manager
GGSCI (core_dg as ggadm@coredb2) 23> start mgr

# 启动extract
GGSCI (core_dg as ggadm@coredb2) 23> start exthx

# 启动pump
GGSCI (core_dg as ggadm@coredb2) 23> start pphx
```



# 4. 源柜面库

## 4.1 创建用户

ogg官方文档中有如下描述：`Extract must connect to the root container (cdb$root) as a common user in order to interact with the logmining server`

因此，此用户务必在root容器下创建

```sql
-- 登录到柜面库主库所在的服务器
[coredb01:oracle]:/home/oracle>sqlplus / as sysdba

-- 查看当前容器，确保在root容器下
SQL> show con_name

CON_NAME
------------------------------
CDB$ROOT
-- 查看所有pdb容器
SQL> select name,open_mode from v$pdbs;

-- 创建用户并赋权
SQL> create user C##GGADMIN identified by ggadmin;

User created.

SQL> exec dbms_goldengate_auth.grant_admin_privilege('C##GGADMIN',container=>'ALL');

PL/SQL procedure successfully completed.

SQL> ALTER SYSTEM SET ENABLE_GOLDENGATE_REPLICATION = TRUE;

System altered.

SQL> grant connect, resource, dba to C##ggadmin container=all;

Grant succeeded.
```

## 4.2 添加日志

```sql
-- 在root容器下
SQL> alter database add supplemental log data;

SQL> alter database force logging;

SQL> alter system switch logfile;


SQL> select log_mode from v$database;

LOG_MODE
------------------------
ARCHIVELOG

SQL> select supplemental_log_data_min, force_logging from v$database;

SUPPLEME FORCE_LOGGING
-------- ---------------------------------------
YES      YES
```

## 4.3 开启pdb日志

```sql
-- 切换到对应的pdb容器
SQL> alter session set container=tellerdb;

Session altered.

SQL> show con_name;

CON_NAME
------------------------------
TELLERDB

SQL> alter database add supplemental log data;

Database altered.
-- 最终确认是否打开
SQL> select supplemental_log_data_min, force_logging from v$database;

SUPPLEME FORCE_LOGGING
-------- ---------------------------------------
YES      YES
```

## 4.4 创建pdb用户

```sql
SQL> create tablespace ogg_tbs datafile '+DATA' size 5G AUTOEXTEND on extent management local segment space management auto;

Tablespace created.

SQL> create user ggadm identified by "oracle" default tablespace ogg_tbs;

User created.

SQL> ALTER USER   GGADM QUOTA UNLIMITED ON  OGG_TBS;

User altered.

SQL> grant connect,resource,dba to ggadm;

Grant succeeded.
```



# 5. 源柜面库ogg安装

> 因为前文中已经在交易库的adg节点安装部署ogg源端，柜面库则使用同一个ogg源端，做远程抽取
>
> 需要添加额外的对柜面库主库的监听配置
>
> 本节中所有涉及到ogg的操作均在已安装完成的交易库adg节点的ogg源端用户下

## 5.1 配置监听

```bash
# 在已经安装了ogg源端的交易库adg节点的Oracle用户下执行
[core_dg:oracle]:/home/oracle>vim /u01/app/oracle/product/19.3.0/dbhome_1/network/admin/tnsnames.ora

# 增加如下配置
HXDB =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 160.160.16.10)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = HXDB)
    )
  )
```

## 5.2 CredentialStore认证

```bash
GGSCI (core_dg) 8> alter credentialstore add user C##GGADMIN@hxdb password ggadmin alias ogg_gm

Credential store altered.

GGSCI (core_dg) 9> dblogin useridalias ogg_gm
Successfully logged into database CDB$ROOT.
```

## 5.3 添加附加日志

```bash
# 在cdb模式下添加需要注意：1、必须登录到主库root容器；2、必须增加容器名，即 容器名+数据库+数据表

GGSCI (core_dg as C##GGADMIN@hxdb1/CDB$ROOT) 10> 
add trandata tellerdb.teller9.sso_rolebasic
add trandata tellerdb.teller9.sso_userrole
add trandata tellerdb.teller9.sso_roleaccessresource
add trandata tellerdb.teller9.sso_resourcebasic
add trandata tellerdb.teller9.tl9_resource_mapping
add trandata tellerdb.teller9.tl9_business_journala_extends
add trandata tellerdb.teller9.tl9_business_journala
add trandata tellerdb.teller9.tl9_unprint_tran
```

## 5.4 添加检查点

```bash
GGSCI (core_dg as C##GGADMIN@hxdb1/CDB$ROOT) 19> add checkpointtable tellerdb.ggadm.checkpoint

Successfully created checkpoint table tellerdb.ggadm.checkpoint.
```

## 5.5 生成表结构定义文件

### 5.5.1 编辑参数文件

```bash
GGSCI (core_dg as C##GGADMIN@hxdb1/CDB$ROOT) 20> edit param def_gm

# 添加如下配置
USERIDALIAS ogg_gm
defsfile ./dirdef/def_gm.def PURGE

table tellerdb.teller9.sso_rolebasic;
table tellerdb.teller9.sso_userrole;
table tellerdb.teller9.sso_roleaccessresource;
table tellerdb.teller9.sso_resourcebasic;
table tellerdb.teller9.tl9_resource_mapping;
table tellerdb.teller9.tl9_business_journala_extends;
table tellerdb.teller9.tl9_business_journala;
table tellerdb.teller9.tl9_unprint_tran;
```

### 5.5.2 生成表结构定义文件

```bash
[ggs@core_dg ogg]$ pwd
/home/ggs/ogg
[ggs@core_dg ogg]$ ./defgen paramfile ./dirprm/def_gm.prm NOEXTATTR
# 传送到ogg目标端服务器
[ggs@core_dg ogg]$ scp dirdef/def_gm.def ogg12@160.161.12.43:~/dirdef/
```



## 5.6 manager配置

manager已经在配置交易库的时候配好成功了，此处无需再次配置

## 5.7 extract配置

```bash
GGSCI (core_dg as C##GGADMIN@hxdb1/CDB$ROOT) 26> add extract extgm, INTEGRATED tranlog, begin now
EXTRACT (Integrated) added.


GGSCI (core_dg as C##GGADMIN@hxdb1/CDB$ROOT) 27> add exttrail ./dirdat/gm, extract extgm, megabytes 200
EXTTRAIL added.

GGSCI (core_dg as C##GGADMIN@hxdb1/CDB$ROOT) 28> edit params extgm

# 添加如下配置
extract extgm
setenv (NLS_LANG=AMERICAN_AMERICA.UTF8)
USERIDALIAS ogg_gm
FETCHOPTIONS NOUSESNAPSHOT
GETTRUNCATES
EXTTRAIL ./dirdat/gm,format release 12.3
DISCARDFILE ./dirrpt/gm.dsc, APPEND, MEGABYTES 4000
WARNLONGTRANS 1H, CHECKINTERVAL 5M
CACHEMGR CACHESIZE 1024MB, CACHEDIRECTORY ./dirtmp

GETUPDATEBEFORES
LOGALLSUPCOLS
NOCOMPRESSUPDATES
UPDATERECORDFORMAT FULL

REPORTCOUNT EVERY 60 SECONDS, RATE

table tellerdb.teller9.sso_rolebasic;
table tellerdb.teller9.sso_userrole;
table tellerdb.teller9.sso_roleaccessresource;
table tellerdb.teller9.sso_resourcebasic;
table tellerdb.teller9.tl9_business_journala, COLSEXCEPT (business_data, bussview);
table tellerdb.teller9.tl9_unprint_tran;
```

## 5.8 pump 配置

```bash
GGSCI (core_dg as C##GGADMIN@hxdb1/CDB$ROOT) 30> add extract ppgm, exttrailsource ./dirdat/gm
EXTRACT added.


GGSCI (core_dg as C##GGADMIN@hxdb1/CDB$ROOT) 31> add rmttrail /home/ogg12/dirdat/gm, extract ppgm
RMTTRAIL added.

GGSCI (core_dg as C##GGADMIN@hxdb1/CDB$ROOT) 32> edit params ppgm

# 添加如下配置
```



## 5.9 注册数据库

```bash
GGSCI (core_dg as C##GGADMIN@hxdb1/CDB$ROOT) 33> register extract extgm database container(tellerdb)

2022-02-23 19:19:50  INFO    OGG-02003  Extract EXTGM successfully registered with database at SCN 233618221.
```

## 5.10 启动

```bash
# 启动extract
GGSCI (core_dg as C##GGADMIN@hxdb1/CDB$ROOT) 40> start extgm

# 启动pump
GGSCI (core_dg as C##GGADMIN@hxdb1/CDB$ROOT) 41> start ppgm

# 查看所有进程状态
GGSCI (core_dg as C##GGADMIN@hxdb1/CDB$ROOT) 48> info all

Program     Status      Group       Lag at Chkpt  Time Since Chkpt

MANAGER     RUNNING
EXTRACT     RUNNING     EXTGM       00:00:07      00:00:01
EXTRACT     RUNNING     EXTHX       00:00:04      00:00:05
EXTRACT     RUNNING     PPGM        00:00:00      00:00:08
EXTRACT     RUNNING     PPHX        00:00:00      00:00:00
```



# 9. ogg目标端

## 9.1 kafka

1. 增加topic
2. 创建consumer并注册topic

## 9.2 replicat配置

```bash
# 核心交易库部分
GGSCI (node43) 1254> add replicat hx_pd, exttrail ./dirdat/hx, checkpointtable ggadm.checkpoint
REPLICAT added.

GGSCI (node43) 1255> edit param hx_pd

# 添加如下内容
replicat hx_pd
TARGETDB LIBFILE libggjava.so SET property=dirprm/ens_pd.props 
SOURCEDEFS ./dirdef/def_hx.def OVERRIDE
REPORTCOUNT EVERY 1 MINUTES, RATE
GROUPTRANSOPS 10000
MAXTRANSOPS 20000
GETTRUNCATES
REPLACEBADCHAR NULL FORCECHECK

MAP ens_cbank.ac_subject, TARGET ens_cbank.ac_subject;
MAP ens_cbank.cd_card_arch, TARGET ens_cbank.cd_card_arch;
MAP ens_cbank.cd_card_chg, TARGET ens_cbank.cd_card_chg;
MAP ens_cbank.cif_business, TARGET ens_cbank.cif_business;
MAP ens_cbank.cif_category_type, TARGET ens_cbank.cif_category_type;
MAP ens_cbank.cif_class_4, TARGET ens_cbank.cif_class_4;
MAP ens_cbank.cif_class_5, TARGET ens_cbank.cif_class_5;
MAP ens_cbank.cif_client, TARGET ens_cbank.cif_client;
MAP ens_cbank.cif_client_contact_address, TARGET ens_cbank.cif_client_contact_address;
MAP ens_cbank.cif_client_contact_tbl, TARGET ens_cbank.cif_client_contact_tbl;
MAP ens_cbank.cif_client_corp, TARGET ens_cbank.cif_client_corp;
MAP ens_cbank.cif_client_document, TARGET ens_cbank.cif_client_document;
MAP ens_cbank.cif_client_indvl, TARGET ens_cbank.cif_client_indvl;
MAP ens_cbank.cif_client_status, TARGET ens_cbank.cif_client_status;
MAP ens_cbank.cif_client_type, TARGET ens_cbank.cif_client_type;
MAP ens_cbank.cif_cross_relations, TARGET ens_cbank.cif_cross_relations;
MAP ens_cbank.cif_document_type, TARGET ens_cbank.cif_document_type;
MAP ens_cbank.cif_education, TARGET ens_cbank.cif_education;
MAP ens_cbank.cif_industry, TARGET ens_cbank.cif_industry;
MAP ens_cbank.cif_occupation, TARGET ens_cbank.cif_occupation;
MAP ens_cbank.cif_relation_type, TARGET ens_cbank.cif_relation_type;
MAP ens_cbank.cif_resident_type, TARGET ens_cbank.cif_resident_type;
MAP ens_cbank.dc_precontract, TARGET ens_cbank.dc_precontract;
MAP ens_cbank.dc_stage_define, TARGET ens_cbank.dc_stage_define;
MAP ens_cbank.fm_branch, TARGET ens_cbank.fm_branch;
MAP ens_cbank.fm_channel, TARGET ens_cbank.fm_channel;
MAP ens_cbank.fm_city, TARGET ens_cbank.fm_city;
MAP ens_cbank.fm_country, TARGET ens_cbank.fm_country;
MAP ens_cbank.fm_currency, TARGET ens_cbank.fm_currency;
MAP ens_cbank.fm_dist_code, TARGET ens_cbank.fm_dist_code;
MAP ens_cbank.fm_loc_holiday, TARGET ens_cbank.fm_loc_holiday;
MAP ens_cbank.fm_period_freq, TARGET ens_cbank.fm_period_freq;
MAP ens_cbank.fm_ref_code, TARGET ens_cbank.fm_ref_code;
MAP ens_cbank.fm_restraint_type, TARGET ens_cbank.fm_restraint_type;
MAP ens_cbank.fm_state, TARGET ens_cbank.fm_state;
MAP ens_cbank.fm_tran_info, TARGET ens_cbank.fm_tran_info, KEYCOLS (TRAN_SEQ_NO);
MAP ens_cbank.fm_user, TARGET ens_cbank.fm_user;
MAP ens_cbank.gl_prod_accounting, TARGET ens_cbank.gl_prod_accounting;
MAP ens_cbank.irl_ccy_rate, TARGET ens_cbank.irl_ccy_rate;
MAP ens_cbank.irl_exchange_type, TARGET ens_cbank.irl_exchange_type;
MAP ens_cbank.irl_int_basis, TARGET ens_cbank.irl_int_basis;
MAP ens_cbank.irl_int_matrix, TARGET ens_cbank.irl_int_matrix;
MAP ens_cbank.irl_int_rate, TARGET ens_cbank.irl_int_rate;
MAP ens_cbank.irl_int_type, TARGET ens_cbank.irl_int_type;
MAP ens_cbank.irl_prod_int, TARGET ens_cbank.irl_prod_int;
MAP ens_cbank.irl_prod_type, TARGET ens_cbank.irl_prod_type;
MAP ens_cbank.mb_accounting_status, TARGET ens_cbank.mb_accounting_status;
MAP ens_cbank.mb_acct, TARGET ens_cbank.mb_acct;
MAP ens_cbank.mb_acct_attach, TARGET ens_cbank.mb_acct_attach;
MAP ens_cbank.mb_acct_balance, TARGET ens_cbank.mb_acct_balance;
MAP ens_cbank.mb_acct_int_detail, TARGET ens_cbank.mb_acct_int_detail;
MAP ens_cbank.mb_acct_schedule_detail, TARGET ens_cbank.mb_acct_schedule_detail;
MAP ens_cbank.mb_ad_register, TARGET ens_cbank.mb_ad_register;
MAP ens_cbank.mb_agreement, TARGET ens_cbank.mb_agreement;
MAP ens_cbank.mb_agreement_accord, TARGET ens_cbank.mb_agreement_accord;
MAP ens_cbank.mb_agreement_sms, TARGET ens_cbank.mb_agreement_sms;
MAP ens_cbank.mb_agreement_sxc, TARGET ens_cbank.mb_agreement_sxc;
MAP ens_cbank.mb_attr_value, TARGET ens_cbank.mb_attr_value;
MAP ens_cbank.mb_batch_open_details, TARGET ens_cbank.mb_batch_open_details;
MAP ens_cbank.mb_cdt_info, TARGET ens_cbank.mb_cdt_info;
MAP ens_cbank.mb_certificate_type, TARGET ens_cbank.mb_certificate_type;
MAP ENS_CBANK.mb_commission_register, TARGET ENS_CBANK.mb_commission_register;
MAP ens_cbank.mb_debt_asset, TARGET ens_cbank.mb_debt_asset;
MAP ens_cbank.mb_debt_asset_loan, TARGET ens_cbank.mb_debt_asset_loan;
MAP ens_cbank.mb_debt_asset_hist, TARGET ens_cbank.mb_debt_asset_hist;
MAP ens_cbank.mb_fin_detail, TARGET ens_cbank.mb_fin_detail;
MAP ENS_CBANK.mb_impound_info, TARGET ENS_CBANK.mb_impound_info, KEYCOLS (reference, tran_date, internal_key);
MAP ens_cbank.mb_invoice, TARGET ens_cbank.mb_invoice;
MAP ens_cbank.mb_materials_type, TARGET ens_cbank.mb_materials_type;
MAP ens_cbank.mb_open_close_reg, TARGET ens_cbank.mb_open_close_reg;
MAP ens_cbank.mb_prod_type, TARGET ens_cbank.mb_prod_type;
MAP ens_cbank.mb_receipt, TARGET ens_cbank.mb_receipt;
MAP ens_cbank.mb_receipt_detail, TARGET ens_cbank.mb_receipt_detail;
MAP ens_cbank.mb_restraints, TARGET ens_cbank.mb_restraints;
MAP ens_cbank.mb_stage_define, TARGET ens_cbank.mb_stage_define;
MAP ens_cbank.mb_stage_matrix, TARGET ens_cbank.mb_stage_matrix;
MAP ens_cbank.mb_tda_hist, TARGET ens_cbank.mb_tda_hist;
MAP ens_cbank.mb_tran_def, TARGET ens_cbank.mb_tran_def;
MAP ens_cbank.mb_tran_hist, TARGET ens_cbank.mb_tran_hist;
MAP ens_cbank.mb_transfer_contract, TARGET ens_cbank.mb_transfer_contract;
MAP ens_cbank.mb_transfer_detail, TARGET ens_cbank.mb_transfer_detail;
MAP ens_cbank.mb_xfc_fin, TARGET ens_cbank.mb_xfc_fin;
MAP ens_cbank.mb_xfc_stage_define, TARGET ens_cbank.mb_xfc_stage_define;
MAP ens_cbank.mm_interbank_busi_reg, TARGET ens_cbank.mm_interbank_busi_reg;
MAP ens_cbank.mm_interbank_repay_detail, TARGET ens_cbank.mm_interbank_repay_detail;
MAP ens_cbank.nx_rc_cust_list_mg, TARGET ens_cbank.nx_rc_cust_list_mg;
MAP ens_cbank.stc_acct, TARGET ens_cbank.stc_acct;
MAP ens_cbank.tb_cash_balance, TARGET ens_cbank.tb_cash_balance;
MAP ens_cbank.tb_cash_journal, TARGET ens_cbank.tb_cash_journal;
MAP ens_cbank.tb_cash_move, TARGET ens_cbank.tb_cash_move;
MAP ens_cbank.tb_cash_move_detail, TARGET ens_cbank.tb_cash_move_detail;
MAP ens_cbank.tb_trailbox, TARGET ens_cbank.tb_trailbox;
MAP ens_cbank.tb_voucher_def, TARGET ens_cbank.tb_voucher_def;
MAP ens_cbank.tb_voucher_info, TARGET ens_cbank.tb_voucher_info;

-- 日终任务参数表
-- MAP ens_cbank.fm_system, TARGET ens_cbank.fm_system, FILTER(@STREQ (process_split_ind, 'N')), EVENTACTIONS(SHELL "/home/ogg12/t.sh", LOG INFO, STOP);
MAP ens_cbank.fm_system, TARGET ens_cbank.fm_system;

MAP ens_cbank.rc_all_list, TARGET ens_cbank.rc_all_list;
MAP ens_cbank.fm_settlement_type, TARGET ens_cbank.fm_settlement_type;
MAP ens_cbank.mb_event_type, TARGET ens_cbank.mb_event_type;
MAP ens_cbank.gl_event_type, TARGET ens_cbank.gl_event_type;
MAP ens_cbank.rc_list_type, TARGET ens_cbank.rc_list_type;
MAP ens_cbank.rc_list_category, TARGET ens_cbank.rc_list_category;
MAP ens_cbank.fm_system_id, TARGET ens_cbank.fm_system_id;
MAP ens_cbank.mb_event_part, TARGET ens_cbank.mb_event_part;
MAP ens_cbank.cd_card_journal, TARGET ens_cbank.cd_card_journal;
MAP ens_cbank.mb_ac_hist, TARGET ens_cbank.mb_ac_hist;
MAP ens_cbank.mb_batch_tran, TARGET ens_cbank.mb_batch_tran;
MAP ens_cbank.mb_batch_transfer_details, TARGET ens_cbank.mb_batch_transfer_details;
MAP ens_cbank.cif_client_verification, TARGET ens_cbank.cif_client_verification;
MAP ens_cbank.fx_mb_exchange_tran_hist, TARGET ens_cbank.fx_mb_exchange_tran_hist;
MAP ens_cbank.gl_acct_type, TARGET ens_cbank.gl_acct_type;
MAP ens_cbank.mb_acct_discount, TARGET ens_cbank.mb_acct_discount;
MAP ens_cbank.mb_seal_relation, TARGET ens_cbank.mb_seal_relation;
MAP ens_cbank.mb_agreement_yht; TARGET ens_cbank.mb_agreement_yht;
MAP ens_cbank.mb_prod_define; TARGET ens_cbank.mb_prod_define;
MAP ens_cbank.irl_capt_info; TARGET ens_cbank.irl_capt_info;
MAP ens_cbank.irl_accr_info_main; TARGET ens_cbank.irl_accr_info_main;
MAP ens_cbank.irl_int_split; TARGET ens_cbank.irl_int_split;
MAP ens_cbank.gl_amount_type; TARGET ens_cbank.gl_amount_type;
MAP ens_cbank.mb_acct_settle; TARGET ens_cbank.mb_acct_settle;
MAP ens_cbank.mb_acct_nature_def; TARGET ens_cbank.mb_acct_nature_def;
MAP ens_cbank.mb_launch; TARGET ens_cbank.mb_launch;
MAP ens_cbank.mb_launch_detail; TARGET ens_cbank.mb_launch_detail;
MAP ens_cbank.mb_reason_code; TARGET ens_cbank.mb_reason_code;
MAP ens_cbank.cif_qualification; TARGET ens_cbank.cif_qualification;
MAP ens_cbank.mb_acct_hist; TARGET ens_cbank.mb_acct_hist;
MAP ens_cbank.mb_nocard_file; TARGET ens_cbank.mb_nocard_file;
MAP ens_cbank.cif_document_read; TARGET ens_cbank.cif_document_read;
MAP ens_cbank.lm_limit_cumulative; TARGET ens_cbank.lm_limit_cumulative;
MAP ens_cbank.lm_tran_limit_def; TARGET ens_cbank.lm_tran_limit_def;
MAP ens_cbank.stria_flow; TARGET ens_cbank.stria_flow;
MAP ens_cbank.mb_sxc_tran_hist; TARGET ens_cbank.mb_sxc_tran_hist;


# 核心柜面库部分
GGSCI (node43) 1270> add replicat GM_PD, exttrail ./dirdat/gm, checkpointtable hexindb.ggadm.checkpoint
REPLICAT added.


GGSCI (node43) 1271> edit param gm_pd

# 添加如下配置
replicat gm_pd
TARGETDB LIBFILE libggjava.so SET property=dirprm/ens_pd.props
SOURCEDEFS ./dirdef/def_gm.def OVERRIDE
REPORTCOUNT EVERY 1 MINUTES, RATE
GROUPTRANSOPS 10000
MAXTRANSOPS 20000
GETTRUNCATES
REPLACEBADCHAR NULL FORCECHECK

MAP tellerdb.teller9.sso_rolebasic, TARGET ens_cbank.sso_rolebasic;
MAP tellerdb.teller9.sso_userrole, TARGET ens_cbank.sso_userrole;
MAP tellerdb.teller9.sso_roleaccessresource, TARGET ens_cbank.sso_roleaccessresource;
MAP tellerdb.teller9.sso_resourcebasic, TARGET ens_cbank.sso_resourcebasic;
MAP tellerdb.teller9.tl9_resource_mapping, TARGET ens_cbank.tl9_resource_mapping, KEYCOLS (tran_code_old,tran_code_new);
MAP tellerdb.teller9.tl9_business_journala_extends, TARGET ens_cbank.tl9_business_journala_extends;
MAP tellerdb.teller9.tl9_business_journala, TARGET ens_cbank.tl9_business_journala;
MAP tellerdb.teller9.tl9_unprint_tran, TARGET ens_cbank.tl9_unprint_tran;
```

## 9.3 ogg kafka配置文件

```bash
[ogg12@node43 dirprm]$ vim ens_pd.props

# 添加如下内容
gg.handlerlist=kafkaconnect

#The handler properties
gg.handler.kafkaconnect.type=kafkaconnect
gg.handler.kafkaconnect.kafkaProducerConfigFile=json.properties
gg.handler.kafkaconnect.mode=op
#The following selects the topic name based on the fully qualified table name
gg.handler.kafkaconnect.topicMappingTemplate=ens_ogg
#The following selects the message key using the concatenated primary keys
gg.handler.kafkaconnect.keyMappingTemplate=${tableName}_${primaryKeys}
gg.handler.kafkahandler.MetaHeaderTemplate=${alltokens}

#The formatter properties
gg.handler.kafkaconnect.messageFormatting=op
gg.handler.kafkaconnect.insertOpKey=I
gg.handler.kafkaconnect.updateOpKey=U
gg.handler.kafkaconnect.deleteOpKey=D
gg.handler.kafkaconnect.truncateOpKey=T
gg.handler.kafkaconnect.treatAllColumnsAsStrings=false
gg.handler.kafkaconnect.mapLargeNumbersAsStrings=true
gg.handler.kafkaconnect.iso8601Format=false
gg.handler.kafkaconnect.pkUpdateHandling=abend
gg.handler.kafkaconnect.includeTableName=true
gg.handler.kafkaconnect.includeOpType=true
gg.handler.kafkaconnect.includeOpTimestamp=true
gg.handler.kafkaconnect.includeCurrentTimestamp=true
gg.handler.kafkaconnect.includePosition=true
gg.handler.kafkaconnect.includePrimaryKeys=true
gg.handler.kafkaconnect.includeTokens=true

goldengate.userexit.writers=javawriter
javawriter.stats.display=TRUE
javawriter.stats.full=TRUE

gg.log=log4j
gg.log.level=INFO

gg.report.time=30sec

#Apache Kafka Classpath
gg.classpath=/usr/share/java/kafka/*:/usr/share/java/confluent-common/*
#Confluent IO classpath
#gg.classpath={Confluent install dir}/share/java/kafka-serde-tools/*:{Confluent install dir}/share/java/kafka/*:{Confluent install dir}/share/java/confluent-common/*

javawriter.bootoptions=-Xmx1024m -Xms128m -Djava.class.path=.:ggjava/ggjava.jar:./dirprm
```
