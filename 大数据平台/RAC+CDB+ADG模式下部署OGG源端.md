> CDB：container database数据库容器，12c新特征，Multitenant Environment多租户环境
>
> PDB：Pluggable Database，即可插拔数据库

# 1. 源数据库

## 1.1 创建用户

ogg官方文档中有如下描述：`Extract must connect to the root container (cdb$root) as a common user in order to interact with the logmining server`

因此，此用户务必在root容器下创建

```sql
[oracle@hxdb37 ~]$ sqlplus / as sysdba

-- 确保当前是在root容器
SQL> show con_name

CON_NAME
------------------------------
CDB$ROOT

SQL> create user C##GGADMIN identified by ggadmin;
SQL> exec dbms_goldengate_auth.grant_admin_privilege('C##GGADMIN',container=>'ALL');
SQL> grant connect, resource, dba to C##ggadmin container=all;
```

## 1.2 添加日志支持

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

SUPPLEMENTAL_LOG
----------------
FORCE_LOGGING
------------------------------------------------------------------------------
YES
YES
```

## 1.3 开启pdb日志

```sql
-- 查看所有pdb
SQL> select name,open_mode from v$pdbs;

-- hexindb的pdb日志
SQL> alter session set container=hexindb;

Session altered.

SQL> alter database add supplemental log data;

Database altered.

SQL> select supplemental_log_data_min, force_logging from v$database;

SUPPLEMENTAL_LOG
----------------
FORCE_LOGGING
------------------------------------------------------------------------------
YES
YES

-- guimiandb的pdb日志
SQL> alter session set container=guimiandb;

Session altered.

SQL> alter database add supplemental log data;

Database altered.

SQL> select supplemental_log_data_min, force_logging from v$database;

SUPPLEMENTAL_LOG
----------------
FORCE_LOGGING
------------------------------------------------------------------------------
YES
YES
```

## 1.4 创建pdb用户

```sql
SQL> create tablespace ogg_tbs datafile '+DATA' size 5G AUTOEXTEND on extent management local segment space management auto;

Tablespace created.

SQL> create user ggadm identified by "oracle" default tablespace ogg_tbs;

User created.

SQL> ALTER USER   GGADM QUOTA UNLIMITED ON  OGG_TBS;

User altered.

SQL> grant connect,resource,dba to ggadm;
```

# 2. ogg源端安装

## 2.1 新增系统用户

```bash
# 创建系统用户
[root@hxdbdg ~]# useradd ggs -g oinstall
[root@hxdbdg ~]# cd /home/
[root@hxdbdg home]# ls
ggs  oracle  zz
[root@hxdbdg home]#
```

## 2.2 安装包准备

```bash
[ggs@hxdbdg ~]$ cd ggs/
[ggs@hxdbdg ggs]$ ls
191004_fbo_ggs_Linux_x64_shiphome.zip  fbo_ggs_Linux_x64_shiphome  OGG-19.1.0.0-README.txt  OGG_WinUnix_Rel_Notes_19.1.0.0.4.pdf
[ggs@hxdbdg ggs]$ pwd
/home/ggs/ggs
[ggs@hxdbdg ggs]$ unzip 191004_fbo_ggs_Linux_x64_shiphome.zip
```

## 2.3 修改响应文件

```bash
[ggs@hxdbdg ggs]$ cd fbo_ggs_Linux_x64_shiphome/Disk1/response/
[ggs@hxdbdg response]$ ls
oggcore.rsp
[ggs@hxdbdg response]$ pwd
/home/ggs/ggs/fbo_ggs_Linux_x64_shiphome/Disk1/response
[ggs@hxdbdg response]$ vim oggcore.rsp

# 修改如下内容
INSTALL_OPTION=ORA19c
SOFTWARE_LOCATION=/home/ggs/ogg
START_MANAGER=false
```

## 2.4 安装

**安装路径务必保证是空目录**

```bash
[ggs@hxdbdg response]$ mkdir /home/ggs/ogg
[ggs@hxdbdg response]$ cd /home/ggs/ggs/fbo_ggs_Linux_x64_shiphome/Disk1/
[ggs@hxdbdg Disk1]$ ls
install  response  runInstaller  stage
[ggs@hxdbdg Disk1]$ pwd
/home/ggs/ggs/fbo_ggs_Linux_x64_shiphome/Disk1

[ggs@hxdbdg Disk1]$ ./runInstaller -silent -responseFile /home/ggs/ggs/fbo_ggs_Linux_x64_shiphome/Disk1/response/oggcore.rsp
正在启动 Oracle Universal Installer...

检查临时空间: 必须大于 120 MB。   实际为 16213 MB    通过
检查交换空间: 必须大于 150 MB。   实际为 7987 MB    通过
准备从以下地址启动 Oracle Universal Installer /tmp/OraInstall2021-11-30_09-25-02AM. 请稍候...[ggs@hxdbdg Disk1]$ 可以在以下位置找到本次安装会话的日志:
 /data/app/oraInventory/logs/installActions2021-11-30_09-25-02AM.log
Successfully Setup Software.
Oracle GoldenGate Core 的 安装 已成功。
请查看 '/data/app/oraInventory/logs/silentInstall2021-11-30_09-25-02AM.log' 以获取详细资料。
```

## 2.5 配置环境变量

```bash
[ggs@hxdbdg ~]$ vim ogg.env

# ORACLE_BASE、ORACLE_HOME、ORACLE_SID需要根据实际部署进行调整
# 否则后续的adg远程登录rac主库会无法登录
PATH=$PATH:$HOME/.local/bin:$HOME/bin
export PATH
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.3.0/dbhome_1
export ORACLE_SID=hxdb
export NLS_LANG=AMERICAN_AMERICA.UTF8
export ORACLE_TERM=xterm
export PATH=$ORACLE_HOME/bin:/usr/sbin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
# export LANG=C
export OGG_HOME=/home/ggs/ogg
export PATH=$OGG_HOME:$PATH
export LD_LIBRARY_PATH=$OGG_HOME:$LD_LIBRARY_PATH

[ggs@hxdbdg ~]$ vim .bashrc
# 添加
source ogg.env

[ggs@hxdbdg ~]$ source .bashrc
```

## 2.6 生成ogg目录

```bash
[ggs@hxdbdg ~]$ ggsci

Oracle GoldenGate Command Interpreter for Oracle
Version 19.1.0.0.4 OGGCORE_19.1.0.0.0_PLATFORMS_191017.1054_FBO
Linux, x64, 64bit (optimized), Oracle 19c on Oct 17 2019 21:16:29
Operating system character set identified as UTF-8.

Copyright (C) 1995, 2019, Oracle and/or its affiliates. All rights reserved.



GGSCI (hxdbdg) 1> create subdirs

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

## 2.6 CredentialStore认证

```bash
GGSCI (hxdbdg) 1> add credentialstore

Credential store created.

# 此处的C##GGADMIN@hxdb， hxdb是根据监听文件/u01/app/oracle/product/19.3.0/dbhome_1/network/admin/tnsnames.ora中配置的对主库的监听确定的
GGSCI (hxdbdg) 3> alter credentialstore add user C##GGADMIN@hxdb password ggadmin alias oggmain

Credential store altered.

GGSCI (hxdbdg) 2> dblogin useridalias oggmain
Successfully logged into database CDB$ROOT.
```

## 2.7 添加附加日志

```bash
# 添加附加日志 1、登录到root容器时；2、必须增加容器名，即 容器名+数据库+数据表
GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 332> 

add trandata hexindb.ens_cbank.ac_subject
add trandata hexindb.ens_cbank.cd_card_arch
add trandata hexindb.ens_cbank.cd_card_chg
add trandata hexindb.ens_cbank.cif_business
add trandata hexindb.ens_cbank.cif_category_type
add trandata hexindb.ens_cbank.cif_class_4
add trandata hexindb.ens_cbank.cif_class_5
add trandata hexindb.ens_cbank.cif_client
add trandata hexindb.ens_cbank.cif_client_contact_address
add trandata hexindb.ens_cbank.cif_client_contact_tbl
add trandata hexindb.ens_cbank.cif_client_corp
add trandata hexindb.ens_cbank.cif_client_document
add trandata hexindb.ens_cbank.cif_client_indvl
add trandata hexindb.ens_cbank.cif_client_status
add trandata hexindb.ens_cbank.cif_client_type
add trandata hexindb.ens_cbank.cif_cross_relations
add trandata hexindb.ens_cbank.cif_document_type
add trandata hexindb.ens_cbank.cif_education
add trandata hexindb.ens_cbank.cif_industry
add trandata hexindb.ens_cbank.cif_occupation
add trandata hexindb.ens_cbank.cif_relation_type
add trandata hexindb.ens_cbank.cif_resident_type
add trandata hexindb.ens_cbank.dc_precontract
add trandata hexindb.ens_cbank.dc_stage_define
add trandata hexindb.ens_cbank.fm_branch
add trandata hexindb.ens_cbank.fm_channel
add trandata hexindb.ens_cbank.fm_city
add trandata hexindb.ens_cbank.fm_country
add trandata hexindb.ens_cbank.fm_currency
add trandata hexindb.ens_cbank.fm_dist_code
add trandata hexindb.ens_cbank.fm_loc_holiday
add trandata hexindb.ens_cbank.fm_period_freq
add trandata hexindb.ens_cbank.fm_ref_code
add trandata hexindb.ens_cbank.fm_restraint_type
add trandata hexindb.ens_cbank.fm_state
add trandata hexindb.ens_cbank.fm_tran_info
add trandata hexindb.ens_cbank.fm_user
add trandata hexindb.ens_cbank.gl_prod_accounting
add trandata hexindb.ens_cbank.irl_ccy_rate
add trandata hexindb.ens_cbank.irl_exchange_type
add trandata hexindb.ens_cbank.irl_int_basis
add trandata hexindb.ens_cbank.irl_int_matrix
add trandata hexindb.ens_cbank.irl_int_rate
add trandata hexindb.ens_cbank.irl_int_type
add trandata hexindb.ens_cbank.irl_prod_int
add trandata hexindb.ens_cbank.irl_prod_type
add trandata hexindb.ens_cbank.mb_accounting_status
add trandata hexindb.ens_cbank.mb_acct
add trandata hexindb.ens_cbank.mb_acct_attach
add trandata hexindb.ens_cbank.mb_acct_balance
add trandata hexindb.ens_cbank.mb_acct_int_detail
add trandata hexindb.ens_cbank.mb_acct_schedule_detail
add trandata hexindb.ens_cbank.mb_ad_register
add trandata hexindb.ens_cbank.mb_agreement
add trandata hexindb.ens_cbank.mb_agreement_accord
add trandata hexindb.ens_cbank.mb_agreement_sms
add trandata hexindb.ens_cbank.mb_agreement_sxc
add trandata hexindb.ens_cbank.mb_attr_value
add trandata hexindb.ens_cbank.mb_batch_open_details
add trandata hexindb.ens_cbank.mb_cdt_info
add trandata hexindb.ens_cbank.mb_certificate_type
add trandata hexindb.ENS_CBANK.mb_commission_register
add trandata hexindb.ens_cbank.mb_debt_asset
add trandata hexindb.ens_cbank.mb_debt_asset_loan
add trandata hexindb.ens_cbank.mb_debt_asset_hist
add trandata hexindb.ens_cbank.mb_fin_detail
add trandata hexindb.ENS_CBANK.mb_impound_info
add trandata hexindb.ens_cbank.mb_invoice
add trandata hexindb.ens_cbank.mb_materials_type
add trandata hexindb.ens_cbank.mb_open_close_reg
add trandata hexindb.ens_cbank.mb_prod_type
add trandata hexindb.ens_cbank.mb_prod_define
add trandata hexindb.ens_cbank.mb_receipt
add trandata hexindb.ens_cbank.mb_receipt_detail
add trandata hexindb.ens_cbank.mb_restraints
add trandata hexindb.ens_cbank.mb_stage_define
add trandata hexindb.ens_cbank.mb_stage_matrix
add trandata hexindb.ens_cbank.mb_tda_hist
add trandata hexindb.ens_cbank.mb_tran_def
add trandata hexindb.ens_cbank.mb_tran_hist
add trandata hexindb.ens_cbank.mb_transfer_contract
add trandata hexindb.ens_cbank.mb_transfer_detail
add trandata hexindb.ens_cbank.mb_xfc_fin
add trandata hexindb.ens_cbank.mb_xfc_stage_define
add trandata hexindb.ens_cbank.mm_interbank_busi_reg
add trandata hexindb.ens_cbank.mm_interbank_repay_detail
add trandata hexindb.ens_cbank.nx_rc_cust_list_mg
add trandata hexindb.ens_cbank.stc_acct
add trandata hexindb.ens_cbank.tb_cash_balance
add trandata hexindb.ens_cbank.tb_cash_journal
add trandata hexindb.ens_cbank.tb_cash_move
add trandata hexindb.ens_cbank.tb_cash_move_detail
add trandata hexindb.ens_cbank.tb_trailbox
add trandata hexindb.ens_cbank.tb_voucher_def
add trandata hexindb.ens_cbank.tb_voucher_info
add trandata hexindb.ens_cbank.fm_system
add trandata hexindb.ens_cbank.rc_all_list
add trandata hexindb.ens_cbank.fm_settlement_type
add trandata hexindb.ens_cbank.mb_event_type
add trandata hexindb.ens_cbank.gl_event_type
add trandata hexindb.ens_cbank.rc_list_type
add trandata hexindb.ens_cbank.rc_list_category
add trandata hexindb.ens_cbank.fm_system_id
add trandata hexindb.ens_cbank.mb_event_part
add trandata hexindb.ens_cbank.cd_card_journal
add trandata hexindb.ens_cbank.mb_ac_hist
add trandata hexindb.ens_cbank.mb_batch_tran
add trandata hexindb.ens_cbank.mb_batch_transfer_details
add trandata hexindb.ens_cbank.cif_client_verification
add trandata hexindb.ens_cbank.fx_mb_exchange_tran_hist
add trandata hexindb.ens_cbank.gl_acct_type
add trandata hexindb.ens_cbank.mb_acct_discount
add trandata hexindb.ens_cbank.mb_seal_relation
add trandata hexindb.ens_cbank.mb_agreement_yht
add trandata hexindb.ens_cbank.mb_payinfo
add trandata hexindb.ens_cbank.irl_capt_info
add trandata hexindb.ens_cbank.irl_accr_info_main
add trandata hexindb.ens_cbank.irl_int_split
add trandata hexindb.ens_cbank.gl_amount_type
add trandata hexindb.ens_cbank.mb_acct_settle
add trandata hexindb.ens_cbank.mb_acct_nature_def
add trandata hexindb.ens_cbank.mb_launch
add trandata hexindb.ens_cbank.mb_launch_detail
add trandata hexindb.ens_cbank.mb_reason_code
add trandata hexindb.ens_cbank.cif_qualification
add trandata hexindb.ens_cbank.mb_acct_hist
add trandata hexindb.ens_cbank.mb_nocard_file
add trandata hexindb.ens_cbank.cif_document_read
add trandata hexindb.ens_cbank.lm_limit_cumulative
add trandata hexindb.ens_cbank.lm_tran_limit_def
add trandata hexindb.ens_cbank.stria_flow
add trandata hexindb.ens_cbank.mb_sxc_tran_hist
add trandata guimiandb.teller.sso_rolebasic
add trandata guimiandb.teller.sso_userrole
add trandata guimiandb.teller.sso_roleaccessresource
add trandata guimiandb.teller.sso_resourcebasic
add trandata guimiandb.teller.tl9_resource_mapping
add trandata guimiandb.teller.tl9_business_journala_extends
add trandata guimiandb.teller.tl9_business_journala
add trandata guimiandb.teller.tl9_unprint_tran
```

## 2.8 添加检查点

```bash
GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 334> add checkpointtable hexindb.ggadm.checkpoint

Successfully created checkpoint table hexindb.ggadm.checkpoint.
```

## 2.9 生成表结构定义文件

```bash
# 第一部分 核心主库
GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 335> edit param ensdef_hx


USERIDALIAS oggmain
defsfile ./dirdef/ens_hx.def PURGE

table hexindb.ens_cbank.ac_subject;
table hexindb.ens_cbank.cd_card_arch;
table hexindb.ens_cbank.cd_card_chg;
table hexindb.ens_cbank.cif_business;
table hexindb.ens_cbank.cif_category_type;
table hexindb.ens_cbank.cif_class_4;
table hexindb.ens_cbank.cif_class_5;
table hexindb.ens_cbank.cif_client;
table hexindb.ens_cbank.cif_client_contact_address;
table hexindb.ens_cbank.cif_client_contact_tbl;
table hexindb.ens_cbank.cif_client_corp;
table hexindb.ens_cbank.cif_client_document;
table hexindb.ens_cbank.cif_client_indvl;
table hexindb.ens_cbank.cif_client_status;
table hexindb.ens_cbank.cif_client_type;
table hexindb.ens_cbank.cif_cross_relations;
table hexindb.ens_cbank.cif_document_type;
table hexindb.ens_cbank.cif_education;
table hexindb.ens_cbank.cif_industry;
table hexindb.ens_cbank.cif_occupation;
table hexindb.ens_cbank.cif_relation_type;
table hexindb.ens_cbank.cif_resident_type;
table hexindb.ens_cbank.dc_precontract;
table hexindb.ens_cbank.dc_stage_define;
table hexindb.ens_cbank.fm_branch;
table hexindb.ens_cbank.fm_channel;
table hexindb.ens_cbank.fm_city;
table hexindb.ens_cbank.fm_country;
table hexindb.ens_cbank.fm_currency;
table hexindb.ens_cbank.fm_dist_code;
table hexindb.ens_cbank.fm_loc_holiday;
table hexindb.ens_cbank.fm_period_freq;
table hexindb.ens_cbank.fm_ref_code;
table hexindb.ens_cbank.fm_restraint_type;
table hexindb.ens_cbank.fm_state;
table hexindb.ens_cbank.fm_tran_info;
table hexindb.ens_cbank.fm_user;
table hexindb.ens_cbank.gl_prod_accounting;
table hexindb.ens_cbank.irl_ccy_rate;
table hexindb.ens_cbank.irl_exchange_type;
table hexindb.ens_cbank.irl_int_basis;
table hexindb.ens_cbank.irl_int_matrix;
table hexindb.ens_cbank.irl_int_rate;
table hexindb.ens_cbank.irl_int_type;
table hexindb.ens_cbank.irl_prod_int;
table hexindb.ens_cbank.irl_prod_type;
table hexindb.ens_cbank.mb_accounting_status;
table hexindb.ens_cbank.mb_acct;
table hexindb.ens_cbank.mb_acct_attach;
table hexindb.ens_cbank.mb_acct_balance;
table hexindb.ens_cbank.mb_acct_int_detail;
table hexindb.ens_cbank.mb_acct_schedule_detail;
table hexindb.ens_cbank.mb_ad_register;
table hexindb.ens_cbank.mb_agreement;
table hexindb.ens_cbank.mb_agreement_accord;
table hexindb.ens_cbank.mb_agreement_sms;
table hexindb.ens_cbank.mb_agreement_sxc;
table hexindb.ens_cbank.mb_attr_value;
table hexindb.ens_cbank.mb_batch_open_details;
table hexindb.ens_cbank.mb_cdt_info;
table hexindb.ens_cbank.mb_certificate_type;
table hexindb.ens_cbank.mb_commission_register;
table hexindb.ens_cbank.mb_debt_asset;
table hexindb.ens_cbank.mb_debt_asset_loan;
table hexindb.ens_cbank.mb_debt_asset_hist;
table hexindb.ens_cbank.mb_fin_detail;
table hexindb.ens_cbank.mb_impound_info;
table hexindb.ens_cbank.mb_invoice;
table hexindb.ens_cbank.mb_materials_type;
table hexindb.ens_cbank.mb_open_close_reg;
table hexindb.ens_cbank.mb_prod_type;
table hexindb.ens_cbank.mb_receipt;
table hexindb.ens_cbank.mb_receipt_detail;
table hexindb.ens_cbank.mb_restraints;
table hexindb.ens_cbank.mb_stage_define;
table hexindb.ens_cbank.mb_stage_matrix;
table hexindb.ens_cbank.mb_tda_hist;
table hexindb.ens_cbank.mb_tran_def;
table hexindb.ens_cbank.mb_tran_hist;
table hexindb.ens_cbank.mb_transfer_contract;
table hexindb.ens_cbank.mb_transfer_detail;
table hexindb.ens_cbank.mb_xfc_fin;
table hexindb.ens_cbank.mb_xfc_stage_define;
table hexindb.ens_cbank.mm_interbank_busi_reg;
table hexindb.ens_cbank.mm_interbank_repay_detail;
table hexindb.ens_cbank.stc_acct;
table hexindb.ens_cbank.tb_cash_balance;
table hexindb.ens_cbank.tb_cash_journal;
table hexindb.ens_cbank.tb_cash_move;
table hexindb.ens_cbank.tb_cash_move_detail;
table hexindb.ens_cbank.tb_trailbox;
table hexindb.ens_cbank.tb_voucher_def;
table hexindb.ens_cbank.tb_voucher_info;

table hexindb.ens_cbank.fm_system;

table hexindb.ens_cbank.rc_all_list;
table hexindb.ens_cbank.fm_settlement_type;
table hexindb.ens_cbank.mb_event_type;
table hexindb.ens_cbank.gl_event_type;
table hexindb.ens_cbank.rc_list_type;
table hexindb.ens_cbank.rc_list_category;
table hexindb.ens_cbank.fm_system_id;
table hexindb.ens_cbank.mb_event_part;
table hexindb.ens_cbank.cd_card_journal;
table hexindb.ens_cbank.mb_ac_hist;

table hexindb.ens_cbank.mb_batch_tran;
table hexindb.ens_cbank.mb_batch_transfer_details;
table hexindb.ens_cbank.cif_client_verification;
table hexindb.ens_cbank.fx_mb_exchange_tran_hist;
table hexindb.ens_cbank.gl_acct_type;
table hexindb.ens_cbank.mb_acct_discount;
table hexindb.ens_cbank.mb_seal_relation;

table hexindb.ens_cbank.mb_agreement_yht;
table hexindb.ens_cbank.mb_prod_define;
table hexindb.ens_cbank.mb_payinfo;
table hexindb.ens_cbank.irl_capt_info;
table hexindb.ens_cbank.irl_accr_info_main;
table hexindb.ens_cbank.irl_int_split;
table hexindb.ens_cbank.gl_amount_type;
table hexindb.ens_cbank.mb_acct_settle;
table hexindb.ens_cbank.mb_acct_nature_def;
table hexindb.ens_cbank.mb_launch;
table hexindb.ens_cbank.mb_launch_detail;
table hexindb.ens_cbank.mb_reason_code;
table hexindb.ens_cbank.cif_qualification;
table hexindb.ens_cbank.mb_acct_hist;
table hexindb.ens_cbank.mb_nocard_file;
table hexindb.ens_cbank.cif_document_read;
table hexindb.ens_cbank.lm_limit_cumulative;
table hexindb.ens_cbank.lm_tran_limit_def;
table hexindb.ens_cbank.stria_flow;
table hexindb.ens_cbank.mb_sxc_tran_hist;


# 生成表结构
[ggs@hxdbdg ogg]$ pwd
/home/ggs/ogg
[ggs@hxdbdg ogg]$ ./defgen paramfile ./dirprm/ensdef_hx.prm NOEXTATTR

# 将表结构传送到目标服务器
[ggs@hxdbdg ogg]$ scp dirdef/ens_hx.def ogg12@160.161.12.43:~/dirdef/


# 第二部分：核心核算库
GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 280> edit param ensdef_gm

USERIDALIAS oggmain
defsfile ./dirdef/ens_gm.def PURGE

table guimiandb.teller.sso_rolebasic;
table guimiandb.teller.sso_userrole;
table guimiandb.teller.sso_roleaccessresource;
table guimiandb.teller.sso_resourcebasic;
table guimiandb.teller.tl9_resource_mapping;
table guimiandb.teller.tl9_business_journala_extends;
table guimiandb.teller.tl9_business_journala;
table guimiandb.teller.tl9_unprint_tran;

# 生成表结构
[ggs@hxdbdg ogg]$ pwd
/home/ggs/ogg
[ggs@hxdbdg ogg]$ ./defgen paramfile ./dirprm/ensdef_gm.prm NOEXTATTR

# 将表结构传送到目标服务器
[ggs@hxdbdg ogg]$ scp dirdef/ens_gm.def ogg12@160.161.12.43:~/dirdef/
```

## 2.10 manager配置

```bash
# 启动manger需要在OGG_HOME目录，否则会报错
[ggs@hxdbdg ogg]$ cd $OGG_HOME
[ggs@hxdbdg ogg]$ ggsci

GGSCI (hxdbdg) 1> edit param mgr

PORT 7809
DYNAMICPORTLIST 7810-7820
AUTORESTART ER *,RETRIES 3,WAITMINUTES 3, RESETMINUTES 60
PURGEOLDEXTRACTS ./dirdat/*,usecheckpoints, minkeepdays 3
LAGREPORTHOURS 1
LAGINFOMINUTES 30
LAGCRITICALMINUTES 45
```

## 2.11 extract配置

```bash
# 第一部分：核心库hexindb
GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 338> add extract exthx, INTEGRATED tranlog, begin now
EXTRACT added.


GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 339> add exttrail ./dirdat/hx, extract exthx, megabytes 200
EXTTRAIL added.

GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 342> edit params exthx

extract exthx
setenv (NLS_LANG=AMERICAN_AMERICA.UTF8)
USERIDALIAS oggmain
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

table hexindb.ens_cbank.ac_subject;
table hexindb.ens_cbank.cd_card_arch;
table hexindb.ens_cbank.cd_card_chg;
table hexindb.ens_cbank.cif_business;
table hexindb.ens_cbank.cif_category_type;
table hexindb.ens_cbank.cif_class_4;
table hexindb.ens_cbank.cif_class_5;
table hexindb.ens_cbank.cif_client;
table hexindb.ens_cbank.cif_client_contact_address;
table hexindb.ens_cbank.cif_client_contact_tbl;
table hexindb.ens_cbank.cif_client_corp;
table hexindb.ens_cbank.cif_client_document;
table hexindb.ens_cbank.cif_client_indvl;
table hexindb.ens_cbank.cif_client_status;
table hexindb.ens_cbank.cif_client_type;
table hexindb.ens_cbank.cif_cross_relations;
table hexindb.ens_cbank.cif_document_type;
table hexindb.ens_cbank.cif_education;
table hexindb.ens_cbank.cif_industry;
table hexindb.ens_cbank.cif_occupation;
table hexindb.ens_cbank.cif_relation_type;
table hexindb.ens_cbank.cif_resident_type;
table hexindb.ens_cbank.dc_precontract;
table hexindb.ens_cbank.dc_stage_define;
table hexindb.ens_cbank.fm_branch;
table hexindb.ens_cbank.fm_channel;
table hexindb.ens_cbank.fm_city;
table hexindb.ens_cbank.fm_country;
table hexindb.ens_cbank.fm_currency;
table hexindb.ens_cbank.fm_dist_code;
table hexindb.ens_cbank.fm_loc_holiday;
table hexindb.ens_cbank.fm_period_freq;
table hexindb.ens_cbank.fm_ref_code;
table hexindb.ens_cbank.fm_restraint_type;
table hexindb.ens_cbank.fm_state;
table hexindb.ens_cbank.fm_tran_info;
table hexindb.ens_cbank.fm_user;
table hexindb.ens_cbank.gl_prod_accounting;
table hexindb.ens_cbank.irl_ccy_rate;
table hexindb.ens_cbank.irl_exchange_type;
table hexindb.ens_cbank.irl_int_basis;
table hexindb.ens_cbank.irl_int_matrix;
table hexindb.ens_cbank.irl_int_rate;
table hexindb.ens_cbank.irl_int_type;
table hexindb.ens_cbank.irl_prod_int;
table hexindb.ens_cbank.irl_prod_type;
table hexindb.ens_cbank.mb_accounting_status;
table hexindb.ens_cbank.mb_acct;
table hexindb.ens_cbank.mb_acct_attach;
table hexindb.ens_cbank.mb_acct_balance;
table hexindb.ens_cbank.mb_acct_int_detail;
table hexindb.ens_cbank.mb_acct_schedule_detail;
table hexindb.ens_cbank.mb_ad_register;
table hexindb.ens_cbank.mb_agreement;
table hexindb.ens_cbank.mb_agreement_accord;
table hexindb.ens_cbank.mb_agreement_sms;
table hexindb.ens_cbank.mb_agreement_sxc;
table hexindb.ens_cbank.mb_attr_value;
table hexindb.ens_cbank.mb_batch_open_details;
table hexindb.ens_cbank.mb_cdt_info;
table hexindb.ens_cbank.mb_certificate_type;
table hexindb.ens_cbank.mb_commission_register;
table hexindb.ens_cbank.mb_debt_asset;
table hexindb.ens_cbank.mb_debt_asset_loan;
table hexindb.ens_cbank.mb_debt_asset_hist;
table hexindb.ens_cbank.mb_fin_detail;
table hexindb.ens_cbank.mb_impound_info;
table hexindb.ens_cbank.mb_invoice;
table hexindb.ens_cbank.mb_materials_type;
table hexindb.ens_cbank.mb_open_close_reg;
table hexindb.ens_cbank.mb_prod_type;
table hexindb.ens_cbank.mb_receipt;
table hexindb.ens_cbank.mb_receipt_detail;
table hexindb.ens_cbank.mb_restraints;
table hexindb.ens_cbank.mb_stage_define;
table hexindb.ens_cbank.mb_stage_matrix;
table hexindb.ens_cbank.mb_tda_hist;
table hexindb.ens_cbank.mb_tran_def;
table hexindb.ens_cbank.mb_tran_hist;
table hexindb.ens_cbank.mb_transfer_contract;
table hexindb.ens_cbank.mb_transfer_detail;
table hexindb.ens_cbank.mb_xfc_fin;
table hexindb.ens_cbank.mb_xfc_stage_define;
table hexindb.ens_cbank.mm_interbank_busi_reg;
table hexindb.ens_cbank.mm_interbank_repay_detail;
table hexindb.ens_cbank.stc_acct;
table hexindb.ens_cbank.tb_cash_balance;
table hexindb.ens_cbank.tb_cash_journal;
table hexindb.ens_cbank.tb_cash_move;
table hexindb.ens_cbank.tb_cash_move_detail;
table hexindb.ens_cbank.tb_trailbox;
table hexindb.ens_cbank.tb_voucher_def;
table hexindb.ens_cbank.tb_voucher_info;

table hexindb.ens_cbank.fm_system;

table hexindb.ens_cbank.rc_all_list;
table hexindb.ens_cbank.fm_settlement_type;
table hexindb.ens_cbank.mb_event_type;
table hexindb.ens_cbank.gl_event_type;
table hexindb.ens_cbank.rc_list_type;
table hexindb.ens_cbank.rc_list_category;
table hexindb.ens_cbank.fm_system_id;
table hexindb.ens_cbank.mb_event_part;
table hexindb.ens_cbank.cd_card_journal;
table hexindb.ens_cbank.mb_ac_hist;

table hexindb.ens_cbank.mb_batch_tran;
table hexindb.ens_cbank.mb_batch_transfer_details;

table hexindb.ens_cbank.cif_client_verification;
table hexindb.ens_cbank.fx_mb_exchange_tran_hist;
table hexindb.ens_cbank.gl_acct_type;
table hexindb.ens_cbank.mb_acct_discount;
table hexindb.ens_cbank.mb_seal_relation;
table hexindb.ens_cbank.mb_agreement_yht;
table hexindb.ens_cbank.mb_prod_define;
table hexindb.ens_cbank.mb_payinfo;
table hexindb.ens_cbank.irl_capt_info;
table hexindb.ens_cbank.irl_accr_info_main;
table hexindb.ens_cbank.irl_int_split;
table hexindb.ens_cbank.gl_amount_type;
table hexindb.ens_cbank.mb_acct_settle;
table hexindb.ens_cbank.mb_acct_nature_def;
table hexindb.ens_cbank.mb_launch;
table hexindb.ens_cbank.mb_launch_detail;
table hexindb.ens_cbank.mb_reason_code;
table hexindb.ens_cbank.cif_qualification;
table hexindb.ens_cbank.mb_acct_hist;
table hexindb.ens_cbank.mb_nocard_file;
table hexindb.ens_cbank.cif_document_read;
table hexindb.ens_cbank.lm_limit_cumulative;
table hexindb.ens_cbank.lm_tran_limit_def;
table hexindb.ens_cbank.stria_flow;
table hexindb.ens_cbank.mb_sxc_tran_hist;


# 第二部分：柜面库guimiandb
GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 338> add extract extgm, INTEGRATED tranlog, begin now
EXTRACT added.


GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 339> add exttrail ./dirdat/gm, extract extgm, megabytes 200
EXTTRAIL added.

GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 342> edit params extgm

extract extgm
setenv (NLS_LANG=AMERICAN_AMERICA.UTF8)
USERIDALIAS oggmain
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

table guimiandb.teller.sso_rolebasic;
table guimiandb.teller.sso_userrole;
table guimiandb.teller.sso_roleaccessresource;
table guimiandb.teller.sso_resourcebasic;
table guimiandb.teller.tl9_business_journala, COLSEXCEPT (business_data, bussview);
table guimiandb.teller.tl9_unprint_tran;
```

## 2.12 注册数据库

```bash
GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 21> register extract exthx database container(hexindb)
GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 22> register extract extgm database container(guimiandb)
```

## 2.13 pump配置

```bash
# 第一部分：核心库hexindb
GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 42> add extract pphx, exttrailsource ./dirdat/hx
EXTRACT added.


GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 43> add rmttrail /home/ogg12/dirdat/h1, extract pphx
RMTTRAIL added.

GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 44> edit params pphx

EXTRACT pphx
setenv (NLS_LANG=AMERICAN_AMERICA.UTF8)
USERIDALIAS oggmain
discardfile ./dirrpt/pphx.dsc,append,megabytes 4000
rmthost 160.161.12.43 mgrport 7809
rmttrail /home/ogg12/dirdat/h1

PASSTHRU
table hexindb.ens_cbank.*;


# 第二部分：柜面库guimiandb
GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 42> add extract ppgm, exttrailsource ./dirdat/gm
EXTRACT added.


GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 43> add rmttrail /home/ogg12/dirdat/g2, extract ppgm
RMTTRAIL added.

GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 44> edit params ppgm

EXTRACT ppgm
setenv (NLS_LANG=AMERICAN_AMERICA.UTF8)
USERIDALIAS oggmain
discardfile ./dirrpt/ppgm.dsc,append,megabytes 4000
rmthost 160.161.12.43 mgrport 7809
rmttrail /home/ogg12/dirdat/g2

PASSTHRU
table guimiandb.teller.*;
```

## 2.14 启动

```bash
# 启动
GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 60> start mgr


GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 61> start exthx
GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 61> start extgm


GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 62> start pphx
GGSCI (hxdbdg as C##GGADMIN@hxdb2/CDB$ROOT) 62> start ppgm
```

# 3. ogg目标端

## 3.1 replicat配置

```bash
# 第一部分：核心库hexindb
GGSCI (node43) 98> add replicat rehx, exttrail ./dirdat/h1, checkpointtable hexindb.ggadm.checkpoint
REPLICAT added.

GGSCI (node43) 125> edit param rehx

replicat rehx
TARGETDB LIBFILE libggjava.so SET property=dirprm/reens.props 
SOURCEDEFS ./dirdef/ens_hx.def OVERRIDE
REPORTCOUNT EVERY 1 MINUTES, RATE
GROUPTRANSOPS 10000
MAXTRANSOPS 20000
GETTRUNCATES
REPLACEBADCHAR NULL FORCECHECK

MAP hexindb.ens_cbank.ac_subject, TARGET ens_cbank.ac_subject;
MAP hexindb.ens_cbank.cd_card_arch, TARGET ens_cbank.cd_card_arch;
MAP hexindb.ens_cbank.cd_card_chg, TARGET ens_cbank.cd_card_chg;
MAP hexindb.ens_cbank.cif_business, TARGET ens_cbank.cif_business;
MAP hexindb.ens_cbank.cif_category_type, TARGET ens_cbank.cif_category_type;
MAP hexindb.ens_cbank.cif_class_4, TARGET ens_cbank.cif_class_4;
MAP hexindb.ens_cbank.cif_class_5, TARGET ens_cbank.cif_class_5;
MAP hexindb.ens_cbank.cif_client, TARGET ens_cbank.cif_client;
MAP hexindb.ens_cbank.cif_client_contact_address, TARGET ens_cbank.cif_client_contact_address;
MAP hexindb.ens_cbank.cif_client_contact_tbl, TARGET ens_cbank.cif_client_contact_tbl;
MAP hexindb.ens_cbank.cif_client_corp, TARGET ens_cbank.cif_client_corp;
MAP hexindb.ens_cbank.cif_client_document, TARGET ens_cbank.cif_client_document;
MAP hexindb.ens_cbank.cif_client_indvl, TARGET ens_cbank.cif_client_indvl;
MAP hexindb.ens_cbank.cif_client_status, TARGET ens_cbank.cif_client_status;
MAP hexindb.ens_cbank.cif_client_type, TARGET ens_cbank.cif_client_type;
MAP hexindb.ens_cbank.cif_cross_relations, TARGET ens_cbank.cif_cross_relations;
MAP hexindb.ens_cbank.cif_document_type, TARGET ens_cbank.cif_document_type;
MAP hexindb.ens_cbank.cif_education, TARGET ens_cbank.cif_education;
MAP hexindb.ens_cbank.cif_industry, TARGET ens_cbank.cif_industry;
MAP hexindb.ens_cbank.cif_occupation, TARGET ens_cbank.cif_occupation;
MAP hexindb.ens_cbank.cif_relation_type, TARGET ens_cbank.cif_relation_type;
MAP hexindb.ens_cbank.cif_resident_type, TARGET ens_cbank.cif_resident_type;
MAP hexindb.ens_cbank.dc_precontract, TARGET ens_cbank.dc_precontract;
MAP hexindb.ens_cbank.dc_stage_define, TARGET ens_cbank.dc_stage_define;
MAP hexindb.ens_cbank.fm_branch, TARGET ens_cbank.fm_branch;
MAP hexindb.ens_cbank.fm_channel, TARGET ens_cbank.fm_channel;
MAP hexindb.ens_cbank.fm_city, TARGET ens_cbank.fm_city;
MAP hexindb.ens_cbank.fm_country, TARGET ens_cbank.fm_country;
MAP hexindb.ens_cbank.fm_currency, TARGET ens_cbank.fm_currency;
MAP hexindb.ens_cbank.fm_dist_code, TARGET ens_cbank.fm_dist_code;
MAP hexindb.ens_cbank.fm_loc_holiday, TARGET ens_cbank.fm_loc_holiday;
MAP hexindb.ens_cbank.fm_period_freq, TARGET ens_cbank.fm_period_freq;
MAP hexindb.ens_cbank.fm_ref_code, TARGET ens_cbank.fm_ref_code;
MAP hexindb.ens_cbank.fm_restraint_type, TARGET ens_cbank.fm_restraint_type;
MAP hexindb.ens_cbank.fm_state, TARGET ens_cbank.fm_state;
MAP hexindb.ens_cbank.fm_tran_info, TARGET ens_cbank.fm_tran_info, KEYCOLS (TRAN_SEQ_NO);
MAP hexindb.ens_cbank.fm_user, TARGET ens_cbank.fm_user;
MAP hexindb.ens_cbank.gl_prod_accounting, TARGET ens_cbank.gl_prod_accounting;
MAP hexindb.ens_cbank.irl_ccy_rate, TARGET ens_cbank.irl_ccy_rate;
MAP hexindb.ens_cbank.irl_exchange_type, TARGET ens_cbank.irl_exchange_type;
MAP hexindb.ens_cbank.irl_int_basis, TARGET ens_cbank.irl_int_basis;
MAP hexindb.ens_cbank.irl_int_matrix, TARGET ens_cbank.irl_int_matrix;
MAP hexindb.ens_cbank.irl_int_rate, TARGET ens_cbank.irl_int_rate;
MAP hexindb.ens_cbank.irl_int_type, TARGET ens_cbank.irl_int_type;
MAP hexindb.ens_cbank.irl_prod_int, TARGET ens_cbank.irl_prod_int;
MAP hexindb.ens_cbank.irl_prod_type, TARGET ens_cbank.irl_prod_type;
MAP hexindb.ens_cbank.mb_accounting_status, TARGET ens_cbank.mb_accounting_status;
MAP hexindb.ens_cbank.mb_acct, TARGET ens_cbank.mb_acct;
MAP hexindb.ens_cbank.mb_acct_attach, TARGET ens_cbank.mb_acct_attach;
MAP hexindb.ens_cbank.mb_acct_balance, TARGET ens_cbank.mb_acct_balance;
MAP hexindb.ens_cbank.mb_acct_int_detail, TARGET ens_cbank.mb_acct_int_detail;
MAP hexindb.ens_cbank.mb_acct_schedule_detail, TARGET ens_cbank.mb_acct_schedule_detail;
MAP hexindb.ens_cbank.mb_ad_register, TARGET ens_cbank.mb_ad_register;
MAP hexindb.ens_cbank.mb_agreement, TARGET ens_cbank.mb_agreement;
MAP hexindb.ens_cbank.mb_agreement_accord, TARGET ens_cbank.mb_agreement_accord;
MAP hexindb.ens_cbank.mb_agreement_sms, TARGET ens_cbank.mb_agreement_sms;
MAP hexindb.ens_cbank.mb_agreement_sxc, TARGET ens_cbank.mb_agreement_sxc;
MAP hexindb.ens_cbank.mb_attr_value, TARGET ens_cbank.mb_attr_value;
MAP hexindb.ens_cbank.mb_batch_open_details, TARGET ens_cbank.mb_batch_open_details;
MAP hexindb.ens_cbank.mb_cdt_info, TARGET ens_cbank.mb_cdt_info;
MAP hexindb.ens_cbank.mb_certificate_type, TARGET ens_cbank.mb_certificate_type;
MAP hexindb.ENS_CBANK.mb_commission_register, TARGET ENS_CBANK.mb_commission_register;
MAP hexindb.ens_cbank.mb_debt_asset, TARGET ens_cbank.mb_debt_asset;
MAP hexindb.ens_cbank.mb_debt_asset_loan, TARGET ens_cbank.mb_debt_asset_loan;
MAP hexindb.ens_cbank.mb_debt_asset_hist, TARGET ens_cbank.mb_debt_asset_hist;
MAP hexindb.ens_cbank.mb_fin_detail, TARGET ens_cbank.mb_fin_detail;
MAP hexindb.ENS_CBANK.mb_impound_info, TARGET ENS_CBANK.mb_impound_info, KEYCOLS (reference, tran_date, internal_key);
MAP hexindb.ens_cbank.mb_invoice, TARGET ens_cbank.mb_invoice;
MAP hexindb.ens_cbank.mb_materials_type, TARGET ens_cbank.mb_materials_type;
MAP hexindb.ens_cbank.mb_open_close_reg, TARGET ens_cbank.mb_open_close_reg;
MAP hexindb.ens_cbank.mb_prod_type, TARGET ens_cbank.mb_prod_type;
MAP hexindb.ens_cbank.mb_receipt, TARGET ens_cbank.mb_receipt;
MAP hexindb.ens_cbank.mb_receipt_detail, TARGET ens_cbank.mb_receipt_detail;
MAP hexindb.ens_cbank.mb_restraints, TARGET ens_cbank.mb_restraints;
MAP hexindb.ens_cbank.mb_stage_define, TARGET ens_cbank.mb_stage_define;
MAP hexindb.ens_cbank.mb_stage_matrix, TARGET ens_cbank.mb_stage_matrix;
MAP hexindb.ens_cbank.mb_tda_hist, TARGET ens_cbank.mb_tda_hist;
MAP hexindb.ens_cbank.mb_tran_def, TARGET ens_cbank.mb_tran_def;
MAP hexindb.ens_cbank.mb_tran_hist, TARGET ens_cbank.mb_tran_hist;
MAP hexindb.ens_cbank.mb_transfer_contract, TARGET ens_cbank.mb_transfer_contract;
MAP hexindb.ens_cbank.mb_transfer_detail, TARGET ens_cbank.mb_transfer_detail;
MAP hexindb.ens_cbank.mb_xfc_fin, TARGET ens_cbank.mb_xfc_fin;
MAP hexindb.ens_cbank.mb_xfc_stage_define, TARGET ens_cbank.mb_xfc_stage_define;
MAP hexindb.ens_cbank.mm_interbank_busi_reg, TARGET ens_cbank.mm_interbank_busi_reg;
MAP hexindb.ens_cbank.mm_interbank_repay_detail, TARGET ens_cbank.mm_interbank_repay_detail;
MAP hexindb.ens_cbank.nx_rc_cust_list_mg, TARGET ens_cbank.nx_rc_cust_list_mg;
MAP hexindb.ens_cbank.stc_acct, TARGET ens_cbank.stc_acct;
MAP hexindb.ens_cbank.tb_cash_balance, TARGET ens_cbank.tb_cash_balance;
MAP hexindb.ens_cbank.tb_cash_journal, TARGET ens_cbank.tb_cash_journal;
MAP hexindb.ens_cbank.tb_cash_move, TARGET ens_cbank.tb_cash_move;
MAP hexindb.ens_cbank.tb_cash_move_detail, TARGET ens_cbank.tb_cash_move_detail;
MAP hexindb.ens_cbank.tb_trailbox, TARGET ens_cbank.tb_trailbox;
MAP hexindb.ens_cbank.tb_voucher_def, TARGET ens_cbank.tb_voucher_def;
MAP hexindb.ens_cbank.tb_voucher_info, TARGET ens_cbank.tb_voucher_info;

-- 日终任务参数表
-- MAP hexindb.ens_cbank.fm_system, TARGET ens_cbank.fm_system, FILTER(@STREQ (process_split_ind, 'N')), EVENTACTIONS(SHELL "/home/ogg12/t.sh", LOG INFO, STOP);
MAP hexindb.ens_cbank.fm_system, TARGET ens_cbank.fm_system;

MAP hexindb.ens_cbank.rc_all_list, TARGET ens_cbank.rc_all_list;
MAP hexindb.ens_cbank.fm_settlement_type, TARGET ens_cbank.fm_settlement_type;
MAP hexindb.ens_cbank.mb_event_type, TARGET ens_cbank.mb_event_type;
MAP hexindb.ens_cbank.gl_event_type, TARGET ens_cbank.gl_event_type;
MAP hexindb.ens_cbank.rc_list_type, TARGET ens_cbank.rc_list_type;
MAP hexindb.ens_cbank.rc_list_category, TARGET ens_cbank.rc_list_category;
MAP hexindb.ens_cbank.fm_system_id, TARGET ens_cbank.fm_system_id;
MAP hexindb.ens_cbank.mb_event_part, TARGET ens_cbank.mb_event_part;
MAP hexindb.ens_cbank.cd_card_journal, TARGET ens_cbank.cd_card_journal;
MAP hexindb.ens_cbank.mb_ac_hist, TARGET ens_cbank.mb_ac_hist;

MAP hexindb.ens_cbank.mb_batch_tran, TARGET ens_cbank.mb_batch_tran;
MAP hexindb.ens_cbank.mb_batch_transfer_details, TARGET ens_cbank.mb_batch_transfer_details;
MAP hexindb.ens_cbank.cif_client_verification, TARGET ens_cbank.cif_client_verification;
MAP hexindb.ens_cbank.fx_mb_exchange_tran_hist, TARGET ens_cbank.fx_mb_exchange_tran_hist;
MAP hexindb.ens_cbank.gl_acct_type, TARGET ens_cbank.gl_acct_type;
MAP hexindb.ens_cbank.mb_acct_discount, TARGET ens_cbank.mb_acct_discount;
MAP hexindb.ens_cbank.mb_seal_relation, TARGET ens_cbank.mb_seal_relation;

MAP hexindb.ens_cbank.mb_agreement_yht; TARGET ens_cbank.mb_agreement_yht;
MAP hexindb.ens_cbank.mb_prod_define; TARGET ens_cbank.mb_prod_define;
MAP hexindb.ens_cbank.irl_capt_info; TARGET ens_cbank.irl_capt_info;
MAP hexindb.ens_cbank.irl_accr_info_main; TARGET ens_cbank.irl_accr_info_main;
MAP hexindb.ens_cbank.irl_int_split; TARGET ens_cbank.irl_int_split;
MAP hexindb.ens_cbank.gl_amount_type; TARGET ens_cbank.gl_amount_type;
MAP hexindb.ens_cbank.mb_acct_settle; TARGET ens_cbank.mb_acct_settle;
MAP hexindb.ens_cbank.mb_acct_nature_def; TARGET ens_cbank.mb_acct_nature_def;
MAP hexindb.ens_cbank.mb_launch; TARGET ens_cbank.mb_launch;
MAP hexindb.ens_cbank.mb_launch_detail; TARGET ens_cbank.mb_launch_detail;
MAP hexindb.ens_cbank.mb_reason_code; TARGET ens_cbank.mb_reason_code;
MAP hexindb.ens_cbank.cif_qualification; TARGET ens_cbank.cif_qualification;
MAP hexindb.ens_cbank.mb_acct_hist; TARGET ens_cbank.mb_acct_hist;
MAP hexindb.ens_cbank.mb_nocard_file; TARGET ens_cbank.mb_nocard_file;
MAP hexindb.ens_cbank.cif_document_read; TARGET ens_cbank.cif_document_read;
MAP hexindb.ens_cbank.lm_limit_cumulative; TARGET ens_cbank.lm_limit_cumulative;
MAP hexindb.ens_cbank.lm_tran_limit_def; TARGET ens_cbank.lm_tran_limit_def;
MAP hexindb.ens_cbank.stria_flow; TARGET ens_cbank.stria_flow;
MAP hexindb.ens_cbank.mb_sxc_tran_hist; TARGET ens_cbank.mb_sxc_tran_hist;


# 第二部分：柜面库guimiandb
GGSCI (node43) 98> add replicat regm, exttrail ./dirdat/g2, checkpointtable hexindb.ggadm.checkpoint
REPLICAT added.

GGSCI (node43) 125> edit param regm

replicat regm
TARGETDB LIBFILE libggjava.so SET property=dirprm/reens.props 
SOURCEDEFS ./dirdef/ens_gm.def OVERRIDE
REPORTCOUNT EVERY 1 MINUTES, RATE
GROUPTRANSOPS 10000
MAXTRANSOPS 20000
GETTRUNCATES
REPLACEBADCHAR NULL FORCECHECK

MAP guimiandb.teller.sso_rolebasic, TARGET ens_cbank.sso_rolebasic;
MAP guimiandb.teller.sso_userrole, TARGET ens_cbank.sso_userrole;
MAP guimiandb.teller.sso_roleaccessresource, TARGET ens_cbank.sso_roleaccessresource;
MAP guimiandb.teller.sso_resourcebasic, TARGET ens_cbank.sso_resourcebasic;
MAP guimiandb.teller.tl9_resource_mapping, TARGET ens_cbank.tl9_resource_mapping, KEYCOLS (tran_code_old,tran_code_new);
MAP guimiandb.teller.tl9_business_journala_extends, TARGET ens_cbank.tl9_business_journala_extends;
MAP guimiandb.teller.tl9_business_journala, TARGET ens_cbank.tl9_business_journala;
MAP guimiandb.teller.tl9_unprint_tran, TARGET ens_cbank.tl9_unprint_tran;

```