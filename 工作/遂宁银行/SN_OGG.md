# 开启数据库归档
```bash
shutdown immediate;

startup mount;

alter database archivelog;
archive log list;

alter database open;

Alter database add supplemental log data;
alter system switch logfile;
alter database force logging;

alter system set enable_goldengate_replication=true;

create tablespace ogg_tbs datafile '/data/app/oracle/oradata/ORCL/datafile/ogg_tbs.dbf' size 100M AUTOEXTEND on extent management local segment space management auto

create user ggadm identified by "oracle" default tablespace ogg_tbs;
ALTER USER   GGADM QUOTA UNLIMITED ON  OGG_TBS;
grant connect, resource,CREATE SESSION to ggadm;
exec dbms_goldengate_auth.grant_admin_privilege('ggadm');
exec dbms_goldengate_auth.grant_admin_privilege(grantee=>'ggadm');
grant select any dictionary to ggadm;
commit;
```

# 配置定时删除脚本
```bash
#!/bin/bash

export ORACLE_BASE=/data/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.2.0/db_1
export ORACLE_SID=orcl
export NLS_LANG=AMERICAN_AMERICA.UTF8
export ORACLE_TERM=xterm
export PATH=$ORACLE_HOME/bin:/usr/sbin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export LANG=C

$ORACLE_HOME/bin/rman target / << EOF
crosscheck archivelog all;
delete noprompt archivelog all completed before 'sysdata-1';
EOF>>
```


# ggs源端安装
```bash
unzip 191001_fbo_ggs_Linux_x64_shiphome.zip

alter credentialstore add user ggadm password oracle alias oggsourceadm


vi ogg.env
```

```bash
# ogg.env
PATH=$PATH:$HOME/.local/bin:$HOME/bin
export PATH
export ORACLE_BASE=/data/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.2.0/db_1
export ORACLE_SID=orcl
export NLS_LANG=AMERICAN_AMERICA.UTF8
export ORACLE_TERM=xterm
export PATH=$ORACLE_HOME/bin:/usr/sbin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export LANG=C
export OGG_HOME=/home/ggs/ogg
export PATH=$OGG_HOME:$PATH
export LD_LIBRARY_PATH=$OGG_HOME:$LD_LIBRARY_PATH
```


### add tran data
```bash
ADD TRANDATA ENS_CBANK.MB_PROD_TYPE
ADD TRANDATA ENS_CBANK.CIF_CLIENT
ADD TRANDATA ENS_CBANK.FM_USER
ADD TRANDATA ENS_CBANK.FM_BRANCH
ADD TRANDATA ENS_CBANK.FM_TRAN_INFO
ADD TRANDATA ENS_CBANK.MB_ACCT_STATS
ADD TRANDATA ENS_CBANK.MB_INTERNAL_ACCT_MAPPING
ADD TRANDATA ENS_CBANK.MB_ACCT_SCHEDULE_AMEND_HIST
ADD TRANDATA ENS_CBANK.MB_ACCT_TRAN_STATS
ADD TRANDATA ENS_CBANK.CIF_CLIENT_USED_INFO
ADD TRANDATA ENS_CBANK.TB_CASH_EXCHANGE
ADD TRANDATA ENS_CBANK.TB_TRAN_HIST
ADD TRANDATA ENS_CBANK.MB_ACCT_COMPOSITE_SCHEDULE
ADD TRANDATA ENS_CBANK.MB_CHEQUE_DUD
ADD TRANDATA ENS_CBANK.LM_CLIENT_TRAN_LIMIT
ADD TRANDATA ENS_CBANK.CIF_CLIENT_RATE
ADD TRANDATA ENS_CBANK.CD_CARD_PROD_CHANGE
ADD TRANDATA ENS_CBANK.CD_CARD_CHG
ADD TRANDATA ENS_CBANK.MB_CONTACT_LIST
ADD TRANDATA ENS_CBANK.MB_TRAN_DEF
ADD TRANDATA ENS_CBANK.TB_VOUCHER_INFO
ADD TRANDATA ENS_CBANK.MB_ACCT
ADD TRANDATA ENS_CBANK.AC_SUBJECT
ADD TRANDATA ENS_CBANK.MB_ACCT_SETTLE_INFO
ADD TRANDATA ENS_CBANK.MB_GUAR_PLEDGE_INFO
ADD TRANDATA ENS_CBANK.CIF_SETTLE_DEFAULT
ADD TRANDATA ENS_CBANK.MB_OPEN_CLOSE_REG
ADD TRANDATA ENS_CBANK.MB_ACCT_CLIENT_CHANGE
ADD TRANDATA ENS_CBANK.MB_ACCT_LIMIT
ADD TRANDATA ENS_CBANK.MB_ACCT_ATTACH
ADD TRANDATA ENS_CBANK.MB_ACCT_SCHEDULE
ADD TRANDATA ENS_CBANK.MB_ACCT_SCHEDULE_HIST
ADD TRANDATA ENS_CBANK.MB_ACCT_SCHEDULE_DETAIL
ADD TRANDATA ENS_CBANK.MB_ACCT_INT_MATRIX
ADD TRANDATA ENS_CBANK.MB_ACCT_INT_DETAIL_SPLIT
ADD TRANDATA ENS_CBANK.MB_ACCT_LINKMAN_TYPE
ADD TRANDATA ENS_CBANK.MB_ACCT_SEC_BALANCE
ADD TRANDATA ENS_CBANK.MB_ACCT_BALANCE
ADD TRANDATA ENS_CBANK.MB_MONTH_LIMIT
ADD TRANDATA ENS_CBANK.MB_ACCT_WITHDRAW_TYPE
ADD TRANDATA ENS_CBANK.CD_CARD_BIN
ADD TRANDATA ENS_CBANK.CD_IC_PROD_DEF
ADD TRANDATA ENS_CBANK.CD_CANCEL_REG
ADD TRANDATA ENS_CBANK.CARD_ACCT_EXCHANGE
ADD TRANDATA ENS_CBANK.CD_TRAN_DESC
ADD TRANDATA ENS_CBANK.CD_CARD_JOURNAL
ADD TRANDATA ENS_CBANK.CD_CARD_ARCH
ADD TRANDATA ENS_CBANK.MB_CARD_BOOK_EXCHG
ADD TRANDATA ENS_CBANK.MB_OPEN_CLOSE_REG
ADD TRANDATA ENS_CBANK.GL_PROD_ACCOUNTING
ADD TRANDATA ENS_CBANK.GL_PROD_CODE_MAPPING
ADD TRANDATA ENS_CBANK.AC_SUBJECT
ADD TRANDATA ENS_CBANK.DC_BRANCH_AMT
ADD TRANDATA ENS_CBANK.FM_BANK
ADD TRANDATA ENS_CBANK.FM_EXTERNAL_BRANCH
ADD TRANDATA ENS_CBANK.IRL_PROD_TYPE
ADD TRANDATA ENS_CBANK.MB_PROD_CHARGE
ADD TRANDATA ENS_CBANK.IRL_PROD_INT
ADD TRANDATA ENS_CBANK.MB_CHANGE_PROD
ADD TRANDATA ENS_CBANK.MB_RENEWAL_TYPE
ADD TRANDATA ENS_CBANK.CIF_SIGN_PROD
ADD TRANDATA ENS_CBANK.FM_CHANNEL
ADD TRANDATA ENS_CBANK.DC_CHANNEL_AMT
ADD TRANDATA ENS_CBANK.MB_CHANNEL_RESTRAINTS
ADD TRANDATA ENS_CBANK.MB_ACCT_SETTLE_INFO
ADD TRANDATA ENS_CBANK.MB_MANUAL_DETAIL
ADD TRANDATA ENS_CBANK.MB_TDA_CHANGE
ADD TRANDATA ENS_CBANK.PT_PAYMENT_TRAN_HIST
ADD TRANDATA ENS_CBANK.FX_MB_QR_TRAN_HIST
ADD TRANDATA ENS_CBANK.IRL_ACCR_INFO
ADD TRANDATA ENS_CBANK.IRL_CAPT_MATRIX
ADD TRANDATA ENS_CBANK.IRL_CAPT_RATE_INFO
ADD TRANDATA ENS_CBANK.IRL_CAPT_INFO
ADD TRANDATA ENS_CBANK.MB_CHANNEL_TRAN_HIST
ADD TRANDATA ENS_CBANK.FX_MB_PAYMENT_TRAN_HIST
ADD TRANDATA ENS_CBANK.MB_TRAN_HIST
ADD TRANDATA ENS_CBANK.MB_MISC_HIST
ADD TRANDATA ENS_CBANK.MB_AD_REGISTER_HIST
ADD TRANDATA ENS_CBANK.MB_AC_HIST
ADD TRANDATA ENS_CBANK.CIF_CLIENT_CORP
ADD TRANDATA ENS_CBANK.CIF_CLIENT_INDVL
ADD TRANDATA ENS_CBANK.CIF_CLIENT_BLOCK
ADD TRANDATA ENS_CBANK.FM_ACCT_EXEC
ADD TRANDATA ENS_CBANK.CIF_CLIENT_COMMISSION
ADD TRANDATA ENS_CBANK.CIF_CLIENT_TYPE
ADD TRANDATA ENS_CBANK.CIF_CATEGORY_TYPE
ADD TRANDATA ENS_CBANK.CIF_CLIENT_CONTACT_ADDRESS
ADD TRANDATA ENS_CBANK.CIF_CLIENT_CONTACT_TBL
ADD TRANDATA ENS_CBANK.CIF_CLIENT_DOCUMENT
ADD TRANDATA ENS_CBANK.AMEND_BRANCH
ADD TRANDATA ENS_CBANK.TB_ATTR_INFO
ADD TRANDATA ENS_CBANK.TB_TRAILBOX
ADD TRANDATA ENS_CBANK.TB_TRAILBOX_PLAN
ADD TRANDATA ENS_CBANK.TB_TRAILBOX_PLAN_DETAIL
ADD TRANDATA ENS_CBANK.TB_TRAILBOX_OVERDRAW
ADD TRANDATA ENS_CBANK.TB_CASH_JOURNAL
ADD TRANDATA ENS_CBANK.TB_CASH_BALANCE
ADD TRANDATA ENS_CBANK.TB_CASH_BALANCE_DETAIL
ADD TRANDATA ENS_CBANK.FM_DEFAULT_RATE_TYPE
ADD TRANDATA ENS_CBANK.IRL_EXCHANGE_TYPE
ADD TRANDATA ENS_CBANK.IRL_CCY_RATE
ADD TRANDATA ENS_CBANK.IRL_FWD_CCY_RATE
ADD TRANDATA ENS_CBANK.IRL_ACCT_MATRIX
ADD TRANDATA ENS_CBANK.IRL_INT_BASIS
ADD TRANDATA ENS_CBANK.IRL_BASIS_RATE
ADD TRANDATA ENS_CBANK.IRL_ROLL_INFO
ADD TRANDATA ENS_CBANK.IRL_INT_ROLL
ADD TRANDATA ENS_CBANK.IRL_INT_MATRIX
ADD TRANDATA ENS_CBANK.IRL_INT_TYPE
ADD TRANDATA ENS_CBANK.IRL_INT_RATE
ADD TRANDATA ENS_CBANK.IRL_INT_RATE_HIST
ADD TRANDATA ENS_CBANK.DATE_RATE_ALTER_TABLE
ADD TRANDATA ENS_CBANK.MB_ACCT_INT_MATRIX
ADD TRANDATA ENS_CBANK.MB_AGREEMENT_ASSEMBLE
ADD TRANDATA ENS_CBANK.MB_AGREEMENT_ASS_DETAIL
ADD TRANDATA ENS_CBANK.MB_AGREEMENT_AGENCY
ADD TRANDATA ENS_CBANK.MB_AGREEMENT_LOAN
ADD TRANDATA ENS_CBANK.MB_AGREEMENT_SMS
ADD TRANDATA ENS_CBANK.MB_AGREEMENT_PACKAGE
ADD TRANDATA ENS_CBANK.MB_AGRE_RELATION
ADD TRANDATA ENS_CBANK.MB_AGREEMENT_FUND
ADD TRANDATA ENS_CBANK.FM_ORG_AGRE_CONSTRACT
ADD TRANDATA ENS_CBANK.MB_AGREEMENT_RISING
ADD TRANDATA ENS_CBANK.MB_FINANCIAL_ACCT
ADD TRANDATA ENS_CBANK.MB_DRA_AMOUNTLIMIT
ADD TRANDATA ENS_CBANK.MB_AGREEMENT_OVERDRAFT
ADD TRANDATA ENS_CBANK.MB_AGREEMENT_LOPLOAN
ADD TRANDATA ENS_CBANK.MB_AGREEMENT_TRAN_CHARGE
ADD TRANDATA ENS_CBANK.MB_AGREEMENT_AGENCY
ADD TRANDATA ENS_CBANK.MB_AGENCY_INFO
ADD TRANDATA ENS_CBANK.MB_AGENCY_DETAILS
ADD TRANDATA ENS_CBANK.MB_PN_PAYMENT
ADD TRANDATA ENS_CBANK.MB_PN_LOST
ADD TRANDATA ENS_CBANK.MB_PN_TRAN_DETAIL
ADD TRANDATA ENS_CBANK.MB_PN_REGISTER
ADD TRANDATA ENS_CBANK.MB_ACCT_DISCOUNT
ADD TRANDATA ENS_CBANK.NX_OC_VOUCHER_REGISTER
ADD TRANDATA ENS_CBANK.NX_OC_INTERNAL_ACCT_MAPPING
ADD TRANDATA ENS_CBANK.OC_VOUCHER_REGISTER
ADD TRANDATA ENS_CBANK.OC_TRAN_HIST
ADD TRANDATA ENS_CBANK.MB_AD_REGISTER
ADD TRANDATA ENS_CBANK.MB_RECEIPT
ADD TRANDATA ENS_CBANK.MB_RECEIPT_DETAIL
ADD TRANDATA ENS_CBANK.RC_DOCUMENT_MAPPING
ADD TRANDATA ENS_CBANK.GL_AMOUNT_TYPE
ADD TRANDATA ENS_CBANK.FM_REF_CODE
ADD TRANDATA ENS_CBANK.MB_ATTR_VALUE
ADD TRANDATA ENS_CBANK.MB_EVENT_ATTR
ADD TRANDATA ENS_CBANK.MB_EVENT_TYPE
ADD TRANDATA ENS_CBANK.TB_VOUCHER_DEF
ADD TRANDATA ENS_CBANK.CIF_DOCUMENT_TYPE
ADD TRANDATA ENS_CBANK.CIF_RELATION_TYPE
ADD TRANDATA ENS_CBANK.CIF_RESIDENT_TYPE
ADD TRANDATA ENS_CBANK.GL_EVENT_TYPE
ADD TRANDATA ENS_CBANK.IRL_FEE_TYPE
ADD TRANDATA ENS_CBANK.MB_AMT_CALC_TYPE
ADD TRANDATA ENS_CBANK.MB_ACCT_INT_DETAIL
ADD TRANDATA ENS_CBANK.CIF_CROSS_RELATIONS
ADD TRANDATA ENS_CBANK.CIF_CLIENT_STATUS
ADD TRANDATA ENS_CBANK.CIF_INDUSTRY
ADD TRANDATA ENS_CBANK.FM_COUNTRY
ADD TRANDATA ENS_CBANK.FM_CURRENCY
ADD TRANDATA ENS_CBANK.FM_STATE
ADD TRANDATA ENS_CBANK.FM_CITY
ADD TRANDATA ENS_CBANK.FM_DIST_CODE
ADD TRANDATA ENS_CBANK.CIF_EDUCATION
ADD TRANDATA ENS_CBANK.CIF_OCCUPATION
ADD TRANDATA ENS_CBANK.CIF_CLASS_4
ADD TRANDATA ENS_CBANK.CIF_CLASS_5
ADD TRANDATA ENS_CBANK.CIF_BUSINESS
ADD TRANDATA ENS_CBANK.MB_INVOICE
ADD TRANDATA ENS_CBANK.MB_ACCOUNTING_STATUS
ADD TRANDATA ENS_CBANK.MB_AGREEMENT_SXC
ADD TRANDATA ENS_CBANK.MB_SXC_TRAN_HIST
ADD TRANDATA ENS_CBANK.MB_XFC_FIN
ADD TRANDATA ENS_CBANK.MB_XFC_INT_MATRIX
ADD TRANDATA ENS_CBANK.MB_XFC_STAGE_DEFINE
ADD TRANDATA ENS_CBANK.MB_FIN_DETAIL
ADD TRANDATA ENS_CBANK.MB_FIN_INFO
ADD TRANDATA ENS_CBANK.MB_PRE_BALANCE_HIST
ADD TRANDATA ENS_CBANK.MB_STAGE_CLIENT_INFO
ADD TRANDATA ENS_CBANK.MB_STAGE_DEFINE
ADD TRANDATA ENS_CBANK.MB_STAGE_MATRIX
ADD TRANDATA ENS_CBANK.DC_CHANGE
ADD TRANDATA ENS_CBANK.DC_CHANGE_INFO
ADD TRANDATA ENS_CBANK.DC_PRECONTRACT
ADD TRANDATA ENS_CBANK.DC_REDEMPTION_INFO
ADD TRANDATA ENS_CBANK.DC_STAGE_DEFINE
ADD TRANDATA ENS_CBANK.DC_STAGE_INFO
ADD TRANDATA ENS_CBANK.DC_STAGE_INT
ADD TRANDATA ENS_CBANK.MB_TDA_HIST 
ADD TRANDATA ENS_CBANK.MB_AGREEMENT
ADD TRANDATA ENS_CBANK.FM_LOC_HOLIDAY
ADD TRANDATA ENS_CBANK.MB_BATCH_OPEN_DETAILS
ADD TRANDATA ENS_CBANK.MB_BATCH_OPEN_DCT_DETAILS
ADD TRANDATA ENS_CBANK.MB_DEBT_ASSET
ADD TRANDATA ENS_CBANK.MB_DEBT_ASSET_LOAN
ADD TRANDATA ENS_CBANK.MB_CERTIFICATE_TYPE
ADD TRANDATA ENS_CBANK.MB_MATERIALS_TYPE
ADD TRANDATA ENS_CBANK.MB_TRANSFER_CONTRACT
ADD TRANDATA ENS_CBANK.MB_TRANSFER_DETAIL
ADD TRANDATA ENS_CBANK.STC_ACCT
ADD TRANDATA ENS_CBANK.NX_RC_CUST_LIST_MG
```


# 表结构参数配置
```bash
[ogg@kdb01 ogg]$ pwd
/home/ogg/ogg
[ogg@kdb01 ogg]$ ./ggsci
 
GGSCI (kdb01) 1> edit params ensdefgen
USERIDALIAS oggsourceadm 
defsfile ./dirdef/ens.def, purge
table ens_cbank.mb_acct;
table ens_cbank.ac_subject;
table ens_cbank.mb_acct_settle_info;
table ens_cbank.mb_guar_pledge_info;
table ens_cbank.stc_tran_hist;
table ens_cbank.stc_acct;
table ens_cbank.stc_tax_dtl;
table ens_cbank.stc_income_dtl;
table ens_cbank.cif_settle_default;
table ens_cbank.mb_internal_acct_mapping;
table ens_cbank.mb_open_close_reg;
table ens_cbank.mb_acct_client_change;
table ens_cbank.mb_acct_limit;
table ens_cbank.mb_acct_attach;
table ens_cbank.mb_acct;
table ens_cbank.mb_acct_stats;
table ens_cbank.mb_acct_schedule_amend_hist;
table ens_cbank.mb_acct_schedule;
table ens_cbank.mb_acct_schedule_hist;
table ens_cbank.mb_acct_schedule_detail;
table ens_cbank.mb_acct_tran_stats;
table ens_cbank.mb_acct_int_matrix;
table ens_cbank.mb_acct_int_detail_split;
table ens_cbank.mb_acct_linkman_type;
table ens_cbank.mb_contact_list;
table ens_cbank.mb_acct_sec_balance;
table ens_cbank.mb_acct_balance;
table ens_cbank.mb_month_limit;
table ens_cbank.mb_acct_withdraw_type;
table ens_cbank.fm_acct_client_relation;
table ens_cbank.mb_corp_pay_card_info;
table ens_cbank.cd_ic_elec_card;
table ens_cbank.cd_card_bin;
table ens_cbank.cd_card_prod_change;
table ens_cbank.cd_ic_prod_def;
table ens_cbank.cd_cancel_reg;
table ens_cbank.card_acct_exchange;
table ens_cbank.cd_tran_desc;
table ens_cbank.cd_card_journal;
table ens_cbank.cd_card_chg;
table ens_cbank.cd_card_arch;
table ens_cbank.mb_card_book_exchg;
table ens_cbank.fx_mb_other_tran_hist;
table ens_cbank.mb_open_close_reg;
table ens_cbank.gl_prod_accounting;
table ens_cbank.gl_prod_code_mapping;
table ens_cbank.ac_subject;
table ens_cbank.dc_branch_amt;
table ens_cbank.fm_branch;
table ens_cbank.fm_bank;
table ens_cbank.fm_external_branch;
table ens_cbank.fm_user;
table ens_cbank.irl_prod_type;
table ens_cbank.mb_prod_type;
table ens_cbank.mb_prod_charge;
table ens_cbank.irl_prod_int;
table ens_cbank.mb_change_prod;
table ens_cbank.mb_renewal_type;
table ens_cbank.cif_sign_prod;
table ens_cbank.fm_channel;
table ens_cbank.dc_channel_amt;
table ens_cbank.mb_channel_restraints;
table ens_cbank.mb_acct_settle_info;
table ens_cbank.mb_manual_detail;
table ens_cbank.mb_tda_change;
table ens_cbank.pt_payment_tran_hist;
table ens_cbank.fx_mb_qr_tran_hist;
table ens_cbank.irl_accr_info;
table ens_cbank.irl_capt_matrix;
table ens_cbank.irl_capt_rate_info;
table ens_cbank.irl_capt_info;
table ens_cbank.tb_tran_hist;
table ens_cbank.mb_channel_tran_hist;
table ens_cbank.fx_mb_payment_tran_hist;
table ens_cbank.mb_tran_hist;
table ens_cbank.fm_tran_info;
table ens_cbank.fm_tran_info_hist;
table ens_cbank.mb_misc_hist;
table ens_cbank.mb_ad_register_hist;
table ens_cbank.mb_ac_hist;
table ens_cbank.cif_client_corp;
table ens_cbank.cif_client_indvl;
table ens_cbank.cif_client_used_info;
table ens_cbank.cif_client_block;
table ens_cbank.lm_client_tran_limit;
table ens_cbank.fm_acct_exec;
table ens_cbank.cif_client_commission;
table ens_cbank.cif_client_type;
table ens_cbank.cif_category_type;
table ens_cbank.cif_client_contact_address;
table ens_cbank.cif_client_contact_tbl;
table ens_cbank.cif_client_rate;
table ens_cbank.cif_client;
table ens_cbank.cif_client_document;
table ens_cbank.amend_branch;
table ens_cbank.fm_acct_client_relation;
table ens_cbank.tb_attr_info;
table ens_cbank.tb_trailbox;
table ens_cbank.tb_cash_exchange;
table ens_cbank.tb_trailbox_plan;
table ens_cbank.tb_trailbox_plan_detail;
table ens_cbank.tb_trailbox_overdraw;
table ens_cbank.tb_cash_journal;
table ens_cbank.tb_cash_balance;
table ens_cbank.tb_cash_balance_detail;
table ens_cbank.fm_default_rate_type;
table ens_cbank.irl_exchange_type;
table ens_cbank.irl_ccy_rate;
table ens_cbank.irl_fwd_ccy_rate;
table ens_cbank.irl_after_acct_matrix_hist;
table ens_cbank.irl_after_acct_matrix;
table ens_cbank.irl_acct_matrix;
table ens_cbank.irl_int_basis;
table ens_cbank.irl_basis_rate;
table ens_cbank.irl_roll_info;
table ens_cbank.irl_int_roll;
table ens_cbank.irl_int_matrix;
table ens_cbank.irl_int_type;
table ens_cbank.irl_int_rate;
table ens_cbank.irl_int_rate_hist;
table ens_cbank.date_rate_alter_table;
table ens_cbank.mb_acct_int_matrix;
table ens_cbank.mb_agreement_assemble;
table ens_cbank.mb_agreement_ass_detail;
table ens_cbank.mb_agreement_agency;
table ens_cbank.mb_agreement_loan;
table ens_cbank.mb_agreement_sms;
table ens_cbank.mb_agreement_package;
table ens_cbank.mb_agre_relation;
table ens_cbank.mb_agreement_fund;
table ens_cbank.fm_org_agre_constract;
table ens_cbank.mb_agreement_rising;
table ens_cbank.mb_financial_acct;
table ens_cbank.mb_dra_amountlimit;
table ens_cbank.mb_agreement_overdraft;
table ens_cbank.mb_agreement_loploan;
table ens_cbank.mb_agreement_tran_charge;
table ens_cbank.mb_agreement_agency;
table ens_cbank.mb_agency_info;
table ens_cbank.mb_agency_details;
table ens_cbank.mb_acct_composite_schedule;
table ens_cbank.mb_pn_payment;
table ens_cbank.mb_pn_lost;
table ens_cbank.mb_pn_tran_detail;
table ens_cbank.mb_pn_register;
table ens_cbank.mb_cheque_dud;
table ens_cbank.mb_acct_discount;
table ens_cbank.nx_oc_voucher_register;
table ens_cbank.nx_oc_internal_acct_mapping;
table ens_cbank.oc_voucher_register;
table ens_cbank.oc_tran_hist;
table ens_cbank.mb_ad_register;
table ens_cbank.mb_receipt;
table ens_cbank.mb_receipt_detail;
table ens_cbank.rc_document_mapping;
 
GGSCI (kdb01) 2> exit
```

# 生成表结构文件
```bash
./defgen  paramfile ./dirprm/ensdefgen.prm
scp ./dirdef/ens.def  ggs@160.161.12.43:/home/ogg/dirdef/
```

# manager 配置
```bash
edit param mgr

PORT 7809
DYNAMICPORTLIST 7810-7820
AUTORESTART ER *,RETRIES 3,WAITMINUTES 3, RESETMINUTES 60
PURGEOLDEXTRACTS ./dirdat/*,usecheckpoints, minkeepdays 3
LAGREPORTHOURS 1
LAGINFOMINUTES 30
LAGCRITICALMINUTES 45
```

# extract 初始化配置
```bash
vi ./dirprm/initens.prm

sourceistable
SETENV(ORACLE_SID=orcl)
USERIDALIAS oggsourceadm
rmthost 160.161.12.43, mgrport 7809
rmtfile /home/ogg/dirdat/i2, megabytes 200, PURGE

table ens_cbank.mb_acct;
table ens_cbank.ac_subject;
table ens_cbank.mb_acct_settle_info;
table ens_cbank.mb_guar_pledge_info;
table ens_cbank.stc_tran_hist;
table ens_cbank.stc_acct;
table ens_cbank.stc_tax_dtl;
table ens_cbank.stc_income_dtl;
table ens_cbank.cif_settle_default;
table ens_cbank.mb_internal_acct_mapping;
table ens_cbank.mb_open_close_reg;
table ens_cbank.mb_acct_client_change;
table ens_cbank.mb_acct_limit;
table ens_cbank.mb_acct_attach;
table ens_cbank.mb_acct;
table ens_cbank.mb_acct_stats;
table ens_cbank.mb_acct_schedule_amend_hist;
table ens_cbank.mb_acct_schedule;
table ens_cbank.mb_acct_schedule_hist;
table ens_cbank.mb_acct_schedule_detail;
table ens_cbank.mb_acct_tran_stats;
table ens_cbank.mb_acct_int_matrix;
table ens_cbank.mb_acct_int_detail_split;
table ens_cbank.mb_acct_linkman_type;
table ens_cbank.mb_contact_list;
table ens_cbank.mb_acct_sec_balance;
table ens_cbank.mb_acct_balance;
table ens_cbank.mb_month_limit;
table ens_cbank.mb_acct_withdraw_type;
table ens_cbank.fm_acct_client_relation;
table ens_cbank.mb_corp_pay_card_info;
table ens_cbank.cd_ic_elec_card;
table ens_cbank.cd_card_bin;
table ens_cbank.cd_card_prod_change;
table ens_cbank.cd_ic_prod_def;
table ens_cbank.cd_cancel_reg;
table ens_cbank.card_acct_exchange;
table ens_cbank.cd_tran_desc;
table ens_cbank.cd_card_journal;
table ens_cbank.cd_card_chg;
table ens_cbank.cd_card_arch;
table ens_cbank.mb_card_book_exchg;
table ens_cbank.fx_mb_other_tran_hist;
table ens_cbank.mb_open_close_reg;
table ens_cbank.gl_prod_accounting;
table ens_cbank.gl_prod_code_mapping;
table ens_cbank.ac_subject;
table ens_cbank.dc_branch_amt;
table ens_cbank.fm_branch;
table ens_cbank.fm_bank;
table ens_cbank.fm_external_branch;
table ens_cbank.fm_user;
table ens_cbank.irl_prod_type;
table ens_cbank.mb_prod_type;
table ens_cbank.mb_prod_charge;
table ens_cbank.irl_prod_int;
table ens_cbank.mb_change_prod;
table ens_cbank.mb_renewal_type;
table ens_cbank.cif_sign_prod;
table ens_cbank.fm_channel;
table ens_cbank.dc_channel_amt;
table ens_cbank.mb_channel_restraints;
table ens_cbank.mb_acct_settle_info;
table ens_cbank.mb_manual_detail;
table ens_cbank.mb_tda_change;
table ens_cbank.pt_payment_tran_hist;
table ens_cbank.fx_mb_qr_tran_hist;
table ens_cbank.irl_accr_info;
table ens_cbank.irl_capt_matrix;
table ens_cbank.irl_capt_rate_info;
table ens_cbank.irl_capt_info;
table ens_cbank.tb_tran_hist;
table ens_cbank.mb_channel_tran_hist;
table ens_cbank.fx_mb_payment_tran_hist;
table ens_cbank.mb_tran_hist;
table ens_cbank.fm_tran_info;
table ens_cbank.fm_tran_info_hist;
table ens_cbank.mb_misc_hist;
table ens_cbank.mb_ad_register_hist;
table ens_cbank.mb_ac_hist;
table ens_cbank.cif_client_corp;
table ens_cbank.cif_client_indvl;
table ens_cbank.cif_client_used_info;
table ens_cbank.cif_client_block;
table ens_cbank.lm_client_tran_limit;
table ens_cbank.fm_acct_exec;
table ens_cbank.cif_client_commission;
table ens_cbank.cif_client_type;
table ens_cbank.cif_category_type;
table ens_cbank.cif_client_contact_address;
table ens_cbank.cif_client_contact_tbl;
table ens_cbank.cif_client_rate;
table ens_cbank.cif_client;
table ens_cbank.cif_client_document;
table ens_cbank.amend_branch;
table ens_cbank.fm_acct_client_relation;
table ens_cbank.tb_attr_info;
table ens_cbank.tb_trailbox;
table ens_cbank.tb_cash_exchange;
table ens_cbank.tb_trailbox_plan;
table ens_cbank.tb_trailbox_plan_detail;
table ens_cbank.tb_trailbox_overdraw;
table ens_cbank.tb_cash_journal;
table ens_cbank.tb_cash_balance;
table ens_cbank.tb_cash_balance_detail;
table ens_cbank.fm_default_rate_type;
table ens_cbank.irl_exchange_type;
table ens_cbank.irl_ccy_rate;
table ens_cbank.irl_fwd_ccy_rate;
table ens_cbank.irl_after_acct_matrix_hist;
table ens_cbank.irl_after_acct_matrix;
table ens_cbank.irl_acct_matrix;
table ens_cbank.irl_int_basis;
table ens_cbank.irl_basis_rate;
table ens_cbank.irl_roll_info;
table ens_cbank.irl_int_roll;
table ens_cbank.irl_int_matrix;
table ens_cbank.irl_int_type;
table ens_cbank.irl_int_rate;
table ens_cbank.irl_int_rate_hist;
table ens_cbank.date_rate_alter_table;
table ens_cbank.mb_acct_int_matrix;
table ens_cbank.mb_agreement_assemble;
table ens_cbank.mb_agreement_ass_detail;
table ens_cbank.mb_agreement_agency;
table ens_cbank.mb_agreement_loan;
table ens_cbank.mb_agreement_sms;
table ens_cbank.mb_agreement_package;
table ens_cbank.mb_agre_relation;
table ens_cbank.mb_agreement_fund;
table ens_cbank.fm_org_agre_constract;
table ens_cbank.mb_agreement_rising;
table ens_cbank.mb_financial_acct;
table ens_cbank.mb_dra_amountlimit;
table ens_cbank.mb_agreement_overdraft;
table ens_cbank.mb_agreement_loploan;
table ens_cbank.mb_agreement_tran_charge;
table ens_cbank.mb_agreement_agency;
table ens_cbank.mb_agency_info;
table ens_cbank.mb_agency_details;
table ens_cbank.mb_acct_composite_schedule;
table ens_cbank.mb_pn_payment;
table ens_cbank.mb_pn_lost;
table ens_cbank.mb_pn_tran_detail;
table ens_cbank.mb_pn_register;
table ens_cbank.mb_cheque_dud;
table ens_cbank.mb_acct_discount;
table ens_cbank.nx_oc_voucher_register;
table ens_cbank.nx_oc_internal_acct_mapping;
table ens_cbank.oc_voucher_register;
table ens_cbank.oc_tran_hist;
table ens_cbank.mb_ad_register;
table ens_cbank.mb_receipt;
table ens_cbank.mb_receipt_detail;
table ens_cbank.rc_document_mapping;
```

# 添加extract
```bash
add extract extens, tranlog, begin now, threads 1
add exttrail ./dirdat/de, extract extens, megabytes 100
```

# 编辑extract参数
```bash
edit params extens


extract extens
setenv (NLS_LANG=AMERICAN_AMERICA.UTF8)
USERIDALIAS oggsourceadm
-- RAC use
-- TRANLOGOPTIONS DBLOGREADER
TRANLOGOPTIONS MINEFROMACTIVEDG
FETCHOPTIONS NOUSESNAPSHOT
GETTRUNCATES
EXTTRAIL ./dirdat/de
DISCARDFILE ./dirrpt/extract.dsc, APPEND, MEGABYTES 4000
WARNLONGTRANS 1H, CHECKINTERVAL 5M
CACHEMGR CACHESIZE 1024MB, CACHEDIRECTORY ./dirtmp
 
LOGALLSUPCOLS
NOCOMPRESSUPDATES
UPDATERECORDFORMAT FULL
 
REPORTCOUNT EVERY 60 SECONDS, RATE

table ens_cbank.mb_acct;
table ens_cbank.ac_subject;
table ens_cbank.mb_acct_settle_info;
table ens_cbank.mb_guar_pledge_info;
table ens_cbank.stc_tran_hist;
table ens_cbank.stc_acct;
table ens_cbank.stc_tax_dtl;
table ens_cbank.stc_income_dtl;
table ens_cbank.cif_settle_default;
table ens_cbank.mb_internal_acct_mapping;
table ens_cbank.mb_open_close_reg;
table ens_cbank.mb_acct_client_change;
table ens_cbank.mb_acct_limit;
table ens_cbank.mb_acct_attach;
table ens_cbank.mb_acct;
table ens_cbank.mb_acct_stats;
table ens_cbank.mb_acct_schedule_amend_hist;
table ens_cbank.mb_acct_schedule;
table ens_cbank.mb_acct_schedule_hist;
table ens_cbank.mb_acct_schedule_detail;
table ens_cbank.mb_acct_tran_stats;
table ens_cbank.mb_acct_int_matrix;
table ens_cbank.mb_acct_int_detail_split;
table ens_cbank.mb_acct_linkman_type;
table ens_cbank.mb_contact_list;
table ens_cbank.mb_acct_sec_balance;
table ens_cbank.mb_acct_balance;
table ens_cbank.mb_month_limit;
table ens_cbank.mb_acct_withdraw_type;
table ens_cbank.fm_acct_client_relation;
table ens_cbank.mb_corp_pay_card_info;
table ens_cbank.cd_ic_elec_card;
table ens_cbank.cd_card_bin;
table ens_cbank.cd_card_prod_change;
table ens_cbank.cd_ic_prod_def;
table ens_cbank.cd_cancel_reg;
table ens_cbank.card_acct_exchange;
table ens_cbank.cd_tran_desc;
table ens_cbank.cd_card_journal;
table ens_cbank.cd_card_chg;
table ens_cbank.cd_card_arch;
table ens_cbank.mb_card_book_exchg;
table ens_cbank.fx_mb_other_tran_hist;
table ens_cbank.mb_open_close_reg;
table ens_cbank.gl_prod_accounting;
table ens_cbank.gl_prod_code_mapping;
table ens_cbank.ac_subject;
table ens_cbank.dc_branch_amt;
table ens_cbank.fm_branch;
table ens_cbank.fm_bank;
table ens_cbank.fm_external_branch;
table ens_cbank.fm_user;
table ens_cbank.irl_prod_type;
table ens_cbank.mb_prod_type;
table ens_cbank.mb_prod_charge;
table ens_cbank.irl_prod_int;
table ens_cbank.mb_change_prod;
table ens_cbank.mb_renewal_type;
table ens_cbank.cif_sign_prod;
table ens_cbank.fm_channel;
table ens_cbank.dc_channel_amt;
table ens_cbank.mb_channel_restraints;
table ens_cbank.mb_acct_settle_info;
table ens_cbank.mb_manual_detail;
table ens_cbank.mb_tda_change;
table ens_cbank.pt_payment_tran_hist;
table ens_cbank.fx_mb_qr_tran_hist;
table ens_cbank.irl_accr_info;
table ens_cbank.irl_capt_matrix;
table ens_cbank.irl_capt_rate_info;
table ens_cbank.irl_capt_info;
table ens_cbank.tb_tran_hist;
table ens_cbank.mb_channel_tran_hist;
table ens_cbank.fx_mb_payment_tran_hist;
table ens_cbank.mb_tran_hist;
table ens_cbank.fm_tran_info;
table ens_cbank.fm_tran_info_hist;
table ens_cbank.mb_misc_hist;
table ens_cbank.mb_ad_register_hist;
table ens_cbank.mb_ac_hist;
table ens_cbank.cif_client_corp;
table ens_cbank.cif_client_indvl;
table ens_cbank.cif_client_used_info;
table ens_cbank.cif_client_block;
table ens_cbank.lm_client_tran_limit;
table ens_cbank.fm_acct_exec;
table ens_cbank.cif_client_commission;
table ens_cbank.cif_client_type;
table ens_cbank.cif_category_type;
table ens_cbank.cif_client_contact_address;
table ens_cbank.cif_client_contact_tbl;
table ens_cbank.cif_client_rate;
table ens_cbank.cif_client;
table ens_cbank.cif_client_document;
table ens_cbank.amend_branch;
table ens_cbank.fm_acct_client_relation;
table ens_cbank.tb_attr_info;
table ens_cbank.tb_trailbox;
table ens_cbank.tb_cash_exchange;
table ens_cbank.tb_trailbox_plan;
table ens_cbank.tb_trailbox_plan_detail;
table ens_cbank.tb_trailbox_overdraw;
table ens_cbank.tb_cash_journal;
table ens_cbank.tb_cash_balance;
table ens_cbank.tb_cash_balance_detail;
table ens_cbank.fm_default_rate_type;
table ens_cbank.irl_exchange_type;
table ens_cbank.irl_ccy_rate;
table ens_cbank.irl_fwd_ccy_rate;
table ens_cbank.irl_after_acct_matrix_hist;
table ens_cbank.irl_after_acct_matrix;
table ens_cbank.irl_acct_matrix;
table ens_cbank.irl_int_basis;
table ens_cbank.irl_basis_rate;
table ens_cbank.irl_roll_info;
table ens_cbank.irl_int_roll;
table ens_cbank.irl_int_matrix;
table ens_cbank.irl_int_type;
table ens_cbank.irl_int_rate;
table ens_cbank.irl_int_rate_hist;
table ens_cbank.date_rate_alter_table;
table ens_cbank.mb_acct_int_matrix;
table ens_cbank.mb_agreement_assemble;
table ens_cbank.mb_agreement_ass_detail;
table ens_cbank.mb_agreement_agency;
table ens_cbank.mb_agreement_loan;
table ens_cbank.mb_agreement_sms;
table ens_cbank.mb_agreement_package;
table ens_cbank.mb_agre_relation;
table ens_cbank.mb_agreement_fund;
table ens_cbank.fm_org_agre_constract;
table ens_cbank.mb_agreement_rising;
table ens_cbank.mb_financial_acct;
table ens_cbank.mb_dra_amountlimit;
table ens_cbank.mb_agreement_overdraft;
table ens_cbank.mb_agreement_loploan;
table ens_cbank.mb_agreement_tran_charge;
table ens_cbank.mb_agreement_agency;
table ens_cbank.mb_agency_info;
table ens_cbank.mb_agency_details;
table ens_cbank.mb_acct_composite_schedule;
table ens_cbank.mb_pn_payment;
table ens_cbank.mb_pn_lost;
table ens_cbank.mb_pn_tran_detail;
table ens_cbank.mb_pn_register;
table ens_cbank.mb_cheque_dud;
table ens_cbank.mb_acct_discount;
table ens_cbank.nx_oc_voucher_register;
table ens_cbank.nx_oc_internal_acct_mapping;
table ens_cbank.oc_voucher_register;
table ens_cbank.oc_tran_hist;
table ens_cbank.mb_ad_register;
table ens_cbank.mb_receipt;
table ens_cbank.mb_receipt_detail;
table ens_cbank.rc_document_mapping;
```

# 添加pump
```bash
add extract pumpens, exttrailsource ./dirdat/de
add rmttrail /home/ogg/dirdat/de, extract pumpens
edit params pumpens
```

```bash
EXTRACT pumpens
setenv (NLS_LANG=AMERICAN_AMERICA.UTF8)
USERIDALIAS oggsourceadm
discardfile ./dirrpt/pump.dsc,append,megabytes 4000
rmthost 160.161.12.43 mgrport 7809
rmttrail /home/ogg/dirdat/de
 
PASSTHRU
table ens_cbank.mb_acct;
table ens_cbank.ac_subject;
table ens_cbank.mb_acct_settle_info;
table ens_cbank.mb_guar_pledge_info;
table ens_cbank.stc_tran_hist;
table ens_cbank.stc_acct;
table ens_cbank.stc_tax_dtl;
table ens_cbank.stc_income_dtl;
table ens_cbank.cif_settle_default;
table ens_cbank.mb_internal_acct_mapping;
table ens_cbank.mb_open_close_reg;
table ens_cbank.mb_acct_client_change;
table ens_cbank.mb_acct_limit;
table ens_cbank.mb_acct_attach;
table ens_cbank.mb_acct;
table ens_cbank.mb_acct_stats;
table ens_cbank.mb_acct_schedule_amend_hist;
table ens_cbank.mb_acct_schedule;
table ens_cbank.mb_acct_schedule_hist;
table ens_cbank.mb_acct_schedule_detail;
table ens_cbank.mb_acct_tran_stats;
table ens_cbank.mb_acct_int_matrix;
table ens_cbank.mb_acct_int_detail_split;
table ens_cbank.mb_acct_linkman_type;
table ens_cbank.mb_contact_list;
table ens_cbank.mb_acct_sec_balance;
table ens_cbank.mb_acct_balance;
table ens_cbank.mb_month_limit;
table ens_cbank.mb_acct_withdraw_type;
table ens_cbank.fm_acct_client_relation;
table ens_cbank.mb_corp_pay_card_info;
table ens_cbank.cd_ic_elec_card;
table ens_cbank.cd_card_bin;
table ens_cbank.cd_card_prod_change;
table ens_cbank.cd_ic_prod_def;
table ens_cbank.cd_cancel_reg;
table ens_cbank.card_acct_exchange;
table ens_cbank.cd_tran_desc;
table ens_cbank.cd_card_journal;
table ens_cbank.cd_card_chg;
table ens_cbank.cd_card_arch;
table ens_cbank.mb_card_book_exchg;
table ens_cbank.fx_mb_other_tran_hist;
table ens_cbank.mb_open_close_reg;
table ens_cbank.gl_prod_accounting;
table ens_cbank.gl_prod_code_mapping;
table ens_cbank.ac_subject;
table ens_cbank.dc_branch_amt;
table ens_cbank.fm_branch;
table ens_cbank.fm_bank;
table ens_cbank.fm_external_branch;
table ens_cbank.fm_user;
table ens_cbank.irl_prod_type;
table ens_cbank.mb_prod_type;
table ens_cbank.mb_prod_charge;
table ens_cbank.irl_prod_int;
table ens_cbank.mb_change_prod;
table ens_cbank.mb_renewal_type;
table ens_cbank.cif_sign_prod;
table ens_cbank.fm_channel;
table ens_cbank.dc_channel_amt;
table ens_cbank.mb_channel_restraints;
table ens_cbank.mb_acct_settle_info;
table ens_cbank.mb_manual_detail;
table ens_cbank.mb_tda_change;
table ens_cbank.pt_payment_tran_hist;
table ens_cbank.fx_mb_qr_tran_hist;
table ens_cbank.irl_accr_info;
table ens_cbank.irl_capt_matrix;
table ens_cbank.irl_capt_rate_info;
table ens_cbank.irl_capt_info;
table ens_cbank.tb_tran_hist;
table ens_cbank.mb_channel_tran_hist;
table ens_cbank.fx_mb_payment_tran_hist;
table ens_cbank.mb_tran_hist;
table ens_cbank.fm_tran_info;
table ens_cbank.fm_tran_info_hist;
table ens_cbank.mb_misc_hist;
table ens_cbank.mb_ad_register_hist;
table ens_cbank.mb_ac_hist;
table ens_cbank.cif_client_corp;
table ens_cbank.cif_client_indvl;
table ens_cbank.cif_client_used_info;
table ens_cbank.cif_client_block;
table ens_cbank.lm_client_tran_limit;
table ens_cbank.fm_acct_exec;
table ens_cbank.cif_client_commission;
table ens_cbank.cif_client_type;
table ens_cbank.cif_category_type;
table ens_cbank.cif_client_contact_address;
table ens_cbank.cif_client_contact_tbl;
table ens_cbank.cif_client_rate;
table ens_cbank.cif_client;
table ens_cbank.cif_client_document;
table ens_cbank.amend_branch;
table ens_cbank.fm_acct_client_relation;
table ens_cbank.tb_attr_info;
table ens_cbank.tb_trailbox;
table ens_cbank.tb_cash_exchange;
table ens_cbank.tb_trailbox_plan;
table ens_cbank.tb_trailbox_plan_detail;
table ens_cbank.tb_trailbox_overdraw;
table ens_cbank.tb_cash_journal;
table ens_cbank.tb_cash_balance;
table ens_cbank.tb_cash_balance_detail;
table ens_cbank.fm_default_rate_type;
table ens_cbank.irl_exchange_type;
table ens_cbank.irl_ccy_rate;
table ens_cbank.irl_fwd_ccy_rate;
table ens_cbank.irl_after_acct_matrix_hist;
table ens_cbank.irl_after_acct_matrix;
table ens_cbank.irl_acct_matrix;
table ens_cbank.irl_int_basis;
table ens_cbank.irl_basis_rate;
table ens_cbank.irl_roll_info;
table ens_cbank.irl_int_roll;
table ens_cbank.irl_int_matrix;
table ens_cbank.irl_int_type;
table ens_cbank.irl_int_rate;
table ens_cbank.irl_int_rate_hist;
table ens_cbank.date_rate_alter_table;
table ens_cbank.mb_acct_int_matrix;
table ens_cbank.mb_agreement_assemble;
table ens_cbank.mb_agreement_ass_detail;
table ens_cbank.mb_agreement_agency;
table ens_cbank.mb_agreement_loan;
table ens_cbank.mb_agreement_sms;
table ens_cbank.mb_agreement_package;
table ens_cbank.mb_agre_relation;
table ens_cbank.mb_agreement_fund;
table ens_cbank.fm_org_agre_constract;
table ens_cbank.mb_agreement_rising;
table ens_cbank.mb_financial_acct;
table ens_cbank.mb_dra_amountlimit;
table ens_cbank.mb_agreement_overdraft;
table ens_cbank.mb_agreement_loploan;
table ens_cbank.mb_agreement_tran_charge;
table ens_cbank.mb_agreement_agency;
table ens_cbank.mb_agency_info;
table ens_cbank.mb_agency_details;
table ens_cbank.mb_acct_composite_schedule;
table ens_cbank.mb_pn_payment;
table ens_cbank.mb_pn_lost;
table ens_cbank.mb_pn_tran_detail;
table ens_cbank.mb_pn_register;
table ens_cbank.mb_cheque_dud;
table ens_cbank.mb_acct_discount;
table ens_cbank.nx_oc_voucher_register;
table ens_cbank.nx_oc_internal_acct_mapping;
table ens_cbank.oc_voucher_register;
table ens_cbank.oc_tran_hist;
table ens_cbank.mb_ad_register;
table ens_cbank.mb_receipt;
table ens_cbank.mb_receipt_detail;
table ens_cbank.rc_document_mapping;
```


# 配置目标端manager
```bash
edit params mgr

PORT 7809
DYNAMICPORTLIST 7810-7909
AUTORESTART EXTRACT *,RETRIES 5,WAITMINUTES 3
PURGEOLDEXTRACTS ./dirdat/*,usecheckpoints, minkeepdays 30
```

# 配置checkpoint
```bash
edit params ./GLOBALS

CHECKPOINTTABLE  ogg.checkpoint
```

# 配置replicate
```bash
add replicat reens, exttrail ./dirdat/de, checkpointtable ogg.checkpoint
edit param reens
```

```bash
REPLICAT reens
TARGETDB LIBFILE libggjava.so SET property=dirprm/kafka.props
REPORTCOUNT EVERY 1 MINUTES, RATE
GROUPTRANSOPS 10000
SOURCEDEFS ./dirdef/reens.def

MAP ens_cbank.mb_acct, TARGET ens_cbank.mb_acct;
MAP ens_cbank.ac_subject, TARGET ens_cbank.ac_subject;
MAP ens_cbank.mb_acct_settle_info, TARGET ens_cbank.mb_acct_settle_info;
MAP ens_cbank.mb_guar_pledge_info, TARGET ens_cbank.mb_guar_pledge_info;
MAP ens_cbank.stc_tran_hist, TARGET ens_cbank.stc_tran_hist;
MAP ens_cbank.stc_acct, TARGET ens_cbank.stc_acct;
MAP ens_cbank.stc_tax_dtl, TARGET ens_cbank.stc_tax_dtl;
MAP ens_cbank.stc_income_dtl, TARGET ens_cbank.stc_income_dtl;
MAP ens_cbank.cif_settle_default, TARGET ens_cbank.cif_settle_default;
MAP ens_cbank.mb_internal_acct_mapping, TARGET ens_cbank.mb_internal_acct_mapping;
MAP ens_cbank.mb_open_close_reg, TARGET ens_cbank.mb_open_close_reg;
MAP ens_cbank.mb_acct_client_change, TARGET ens_cbank.mb_acct_client_change;
MAP ens_cbank.mb_acct_limit, TARGET ens_cbank.mb_acct_limit;
MAP ens_cbank.mb_acct_attach, TARGET ens_cbank.mb_acct_attach;
MAP ens_cbank.mb_acct, TARGET ens_cbank.mb_acct;
MAP ens_cbank.mb_acct_stats, TARGET ens_cbank.mb_acct_stats;
MAP ens_cbank.mb_acct_schedule_amend_hist, TARGET ens_cbank.mb_acct_schedule_amend_hist;
MAP ens_cbank.mb_acct_schedule, TARGET ens_cbank.mb_acct_schedule;
MAP ens_cbank.mb_acct_schedule_hist, TARGET ens_cbank.mb_acct_schedule_hist;
MAP ens_cbank.mb_acct_schedule_detail, TARGET ens_cbank.mb_acct_schedule_detail;
MAP ens_cbank.mb_acct_tran_stats, TARGET ens_cbank.mb_acct_tran_stats;
MAP ens_cbank.mb_acct_int_matrix, TARGET ens_cbank.mb_acct_int_matrix;
MAP ens_cbank.mb_acct_int_detail_split, TARGET ens_cbank.mb_acct_int_detail_split;
MAP ens_cbank.mb_acct_linkman_type, TARGET ens_cbank.mb_acct_linkman_type;
MAP ens_cbank.mb_contact_list, TARGET ens_cbank.mb_contact_list;
MAP ens_cbank.mb_acct_sec_balance, TARGET ens_cbank.mb_acct_sec_balance;
MAP ens_cbank.mb_acct_balance, TARGET ens_cbank.mb_acct_balance;
MAP ens_cbank.mb_month_limit, TARGET ens_cbank.mb_month_limit;
MAP ens_cbank.mb_acct_withdraw_type, TARGET ens_cbank.mb_acct_withdraw_type;
MAP ens_cbank.fm_acct_client_relation, TARGET ens_cbank.fm_acct_client_relation;
MAP ens_cbank.mb_corp_pay_card_info, TARGET ens_cbank.mb_corp_pay_card_info;
MAP ens_cbank.cd_ic_elec_card, TARGET ens_cbank.cd_ic_elec_card;
MAP ens_cbank.cd_card_bin, TARGET ens_cbank.cd_card_bin;
MAP ens_cbank.cd_card_prod_change, TARGET ens_cbank.cd_card_prod_change;
MAP ens_cbank.cd_ic_prod_def, TARGET ens_cbank.cd_ic_prod_def;
MAP ens_cbank.cd_cancel_reg, TARGET ens_cbank.cd_cancel_reg;
MAP ens_cbank.card_acct_exchange, TARGET ens_cbank.card_acct_exchange;
MAP ens_cbank.cd_tran_desc, TARGET ens_cbank.cd_tran_desc;
MAP ens_cbank.cd_card_journal, TARGET ens_cbank.cd_card_journal;
MAP ens_cbank.cd_card_chg, TARGET ens_cbank.cd_card_chg;
MAP ens_cbank.cd_card_arch, TARGET ens_cbank.cd_card_arch;
MAP ens_cbank.mb_card_book_exchg, TARGET ens_cbank.mb_card_book_exchg;
MAP ens_cbank.fx_mb_other_tran_hist, TARGET ens_cbank.fx_mb_other_tran_hist;
MAP ens_cbank.mb_open_close_reg, TARGET ens_cbank.mb_open_close_reg;
MAP ens_cbank.gl_prod_accounting, TARGET ens_cbank.gl_prod_accounting;
MAP ens_cbank.gl_prod_code_mapping, TARGET ens_cbank.gl_prod_code_mapping;
MAP ens_cbank.ac_subject, TARGET ens_cbank.ac_subject;
MAP ens_cbank.dc_branch_amt, TARGET ens_cbank.dc_branch_amt;
MAP ens_cbank.fm_branch, TARGET ens_cbank.fm_branch;
MAP ens_cbank.fm_bank, TARGET ens_cbank.fm_bank;
MAP ens_cbank.fm_external_branch, TARGET ens_cbank.fm_external_branch;
MAP ens_cbank.fm_user, TARGET ens_cbank.fm_user;
MAP ens_cbank.irl_prod_type, TARGET ens_cbank.irl_prod_type;
MAP ens_cbank.mb_prod_type, TARGET ens_cbank.mb_prod_type;
MAP ens_cbank.mb_prod_charge, TARGET ens_cbank.mb_prod_charge;
MAP ens_cbank.irl_prod_int, TARGET ens_cbank.irl_prod_int;
MAP ens_cbank.mb_change_prod, TARGET ens_cbank.mb_change_prod;
MAP ens_cbank.mb_renewal_type, TARGET ens_cbank.mb_renewal_type;
MAP ens_cbank.cif_sign_prod, TARGET ens_cbank.cif_sign_prod;
MAP ens_cbank.fm_channel, TARGET ens_cbank.fm_channel;
MAP ens_cbank.dc_channel_amt, TARGET ens_cbank.dc_channel_amt;
MAP ens_cbank.mb_channel_restraints, TARGET ens_cbank.mb_channel_restraints;
MAP ens_cbank.mb_acct_settle_info, TARGET ens_cbank.mb_acct_settle_info;
MAP ens_cbank.mb_manual_detail, TARGET ens_cbank.mb_manual_detail;
MAP ens_cbank.mb_tda_change, TARGET ens_cbank.mb_tda_change;
MAP ens_cbank.pt_payment_tran_hist, TARGET ens_cbank.pt_payment_tran_hist;
MAP ens_cbank.fx_mb_qr_tran_hist, TARGET ens_cbank.fx_mb_qr_tran_hist;
MAP ens_cbank.irl_accr_info, TARGET ens_cbank.irl_accr_info;
MAP ens_cbank.irl_capt_matrix, TARGET ens_cbank.irl_capt_matrix;
MAP ens_cbank.irl_capt_rate_info, TARGET ens_cbank.irl_capt_rate_info;
MAP ens_cbank.irl_capt_info, TARGET ens_cbank.irl_capt_info;
MAP ens_cbank.tb_tran_hist, TARGET ens_cbank.tb_tran_hist;
MAP ens_cbank.mb_channel_tran_hist, TARGET ens_cbank.mb_channel_tran_hist;
MAP ens_cbank.fx_mb_payment_tran_hist, TARGET ens_cbank.fx_mb_payment_tran_hist;
MAP ens_cbank.mb_tran_hist, TARGET ens_cbank.mb_tran_hist;
MAP ens_cbank.fm_tran_info, TARGET ens_cbank.fm_tran_info;
MAP ens_cbank.fm_tran_info_hist, TARGET ens_cbank.fm_tran_info_hist;
MAP ens_cbank.mb_misc_hist, TARGET ens_cbank.mb_misc_hist;
MAP ens_cbank.mb_ad_register_hist, TARGET ens_cbank.mb_ad_register_hist;
MAP ens_cbank.mb_ac_hist, TARGET ens_cbank.mb_ac_hist;
MAP ens_cbank.cif_client_corp, TARGET ens_cbank.cif_client_corp;
MAP ens_cbank.cif_client_indvl, TARGET ens_cbank.cif_client_indvl;
MAP ens_cbank.cif_client_used_info, TARGET ens_cbank.cif_client_used_info;
MAP ens_cbank.cif_client_block, TARGET ens_cbank.cif_client_block;
MAP ens_cbank.lm_client_tran_limit, TARGET ens_cbank.lm_client_tran_limit;
MAP ens_cbank.fm_acct_exec, TARGET ens_cbank.fm_acct_exec;
MAP ens_cbank.cif_client_commission, TARGET ens_cbank.cif_client_commission;
MAP ens_cbank.cif_client_type, TARGET ens_cbank.cif_client_type;
MAP ens_cbank.cif_category_type, TARGET ens_cbank.cif_category_type;
MAP ens_cbank.cif_client_contact_address, TARGET ens_cbank.cif_client_contact_address;
MAP ens_cbank.cif_client_contact_tbl, TARGET ens_cbank.cif_client_contact_tbl;
MAP ens_cbank.cif_client_rate, TARGET ens_cbank.cif_client_rate;
MAP ens_cbank.cif_client, TARGET ens_cbank.cif_client;
MAP ens_cbank.cif_client_document, TARGET ens_cbank.cif_client_document;
MAP ens_cbank.amend_branch, TARGET ens_cbank.amend_branch;
MAP ens_cbank.fm_acct_client_relation, TARGET ens_cbank.fm_acct_client_relation;
MAP ens_cbank.tb_attr_info, TARGET ens_cbank.tb_attr_info;
MAP ens_cbank.tb_trailbox, TARGET ens_cbank.tb_trailbox;
MAP ens_cbank.tb_cash_exchange, TARGET ens_cbank.tb_cash_exchange;
MAP ens_cbank.tb_trailbox_plan, TARGET ens_cbank.tb_trailbox_plan;
MAP ens_cbank.tb_trailbox_plan_detail, TARGET ens_cbank.tb_trailbox_plan_detail;
MAP ens_cbank.tb_trailbox_overdraw, TARGET ens_cbank.tb_trailbox_overdraw;
MAP ens_cbank.tb_cash_journal, TARGET ens_cbank.tb_cash_journal;
MAP ens_cbank.tb_cash_balance, TARGET ens_cbank.tb_cash_balance;
MAP ens_cbank.tb_cash_balance_detail, TARGET ens_cbank.tb_cash_balance_detail;
MAP ens_cbank.fm_default_rate_type, TARGET ens_cbank.fm_default_rate_type;
MAP ens_cbank.irl_exchange_type, TARGET ens_cbank.irl_exchange_type;
MAP ens_cbank.irl_ccy_rate, TARGET ens_cbank.irl_ccy_rate;
MAP ens_cbank.irl_fwd_ccy_rate, TARGET ens_cbank.irl_fwd_ccy_rate;
MAP ens_cbank.irl_after_acct_matrix_hist, TARGET ens_cbank.irl_after_acct_matrix_hist;
MAP ens_cbank.irl_after_acct_matrix, TARGET ens_cbank.irl_after_acct_matrix;
MAP ens_cbank.irl_acct_matrix, TARGET ens_cbank.irl_acct_matrix;
MAP ens_cbank.irl_int_basis, TARGET ens_cbank.irl_int_basis;
MAP ens_cbank.irl_basis_rate, TARGET ens_cbank.irl_basis_rate;
MAP ens_cbank.irl_roll_info, TARGET ens_cbank.irl_roll_info;
MAP ens_cbank.irl_int_roll, TARGET ens_cbank.irl_int_roll;
MAP ens_cbank.irl_int_matrix, TARGET ens_cbank.irl_int_matrix;
MAP ens_cbank.irl_int_type, TARGET ens_cbank.irl_int_type;
MAP ens_cbank.irl_int_rate, TARGET ens_cbank.irl_int_rate;
MAP ens_cbank.irl_int_rate_hist, TARGET ens_cbank.irl_int_rate_hist;
MAP ens_cbank.date_rate_alter_table, TARGET ens_cbank.date_rate_alter_table;
MAP ens_cbank.mb_acct_int_matrix, TARGET ens_cbank.mb_acct_int_matrix;
MAP ens_cbank.mb_agreement_assemble, TARGET ens_cbank.mb_agreement_assemble;
MAP ens_cbank.mb_agreement_ass_detail, TARGET ens_cbank.mb_agreement_ass_detail;
MAP ens_cbank.mb_agreement_agency, TARGET ens_cbank.mb_agreement_agency;
MAP ens_cbank.mb_agreement_loan, TARGET ens_cbank.mb_agreement_loan;
MAP ens_cbank.mb_agreement_sms, TARGET ens_cbank.mb_agreement_sms;
MAP ens_cbank.mb_agreement_package, TARGET ens_cbank.mb_agreement_package;
MAP ens_cbank.mb_agre_relation, TARGET ens_cbank.mb_agre_relation;
MAP ens_cbank.mb_agreement_fund, TARGET ens_cbank.mb_agreement_fund;
MAP ens_cbank.fm_org_agre_constract, TARGET ens_cbank.fm_org_agre_constract;
MAP ens_cbank.mb_agreement_rising, TARGET ens_cbank.mb_agreement_rising;
MAP ens_cbank.mb_financial_acct, TARGET ens_cbank.mb_financial_acct;
MAP ens_cbank.mb_dra_amountlimit, TARGET ens_cbank.mb_dra_amountlimit;
MAP ens_cbank.mb_agreement_overdraft, TARGET ens_cbank.mb_agreement_overdraft;
MAP ens_cbank.mb_agreement_loploan, TARGET ens_cbank.mb_agreement_loploan;
MAP ens_cbank.mb_agreement_tran_charge, TARGET ens_cbank.mb_agreement_tran_charge;
MAP ens_cbank.mb_agreement_agency, TARGET ens_cbank.mb_agreement_agency;
MAP ens_cbank.mb_agency_info, TARGET ens_cbank.mb_agency_info;
MAP ens_cbank.mb_agency_details, TARGET ens_cbank.mb_agency_details;
MAP ens_cbank.mb_acct_composite_schedule, TARGET ens_cbank.mb_acct_composite_schedule;
MAP ens_cbank.mb_pn_payment, TARGET ens_cbank.mb_pn_payment;
MAP ens_cbank.mb_pn_lost, TARGET ens_cbank.mb_pn_lost;
MAP ens_cbank.mb_pn_tran_detail, TARGET ens_cbank.mb_pn_tran_detail;
MAP ens_cbank.mb_pn_register, TARGET ens_cbank.mb_pn_register;
MAP ens_cbank.mb_cheque_dud, TARGET ens_cbank.mb_cheque_dud;
MAP ens_cbank.mb_acct_discount, TARGET ens_cbank.mb_acct_discount;
MAP ens_cbank.nx_oc_voucher_register, TARGET ens_cbank.nx_oc_voucher_register;
MAP ens_cbank.nx_oc_internal_acct_mapping, TARGET ens_cbank.nx_oc_internal_acct_mapping;
MAP ens_cbank.oc_voucher_register, TARGET ens_cbank.oc_voucher_register;
MAP ens_cbank.oc_tran_hist, TARGET ens_cbank.oc_tran_hist;
MAP ens_cbank.mb_ad_register, TARGET ens_cbank.mb_ad_register;
MAP ens_cbank.mb_receipt, TARGET ens_cbank.mb_receipt;
MAP ens_cbank.mb_receipt_detail, TARGET ens_cbank.mb_receipt_detail;
MAP ens_cbank.rc_document_mapping, TARGET ens_cbank.rc_document_mapping;
```

# 添加初始化replicate
```bash
vi ./dirprm/initens.prm

specialrun
end runtime
extfile ./dirdat/init_ens
TARGETDB LIBFILE libggjava.so SET property=dirprm/fnsonld.props 
SOURCEDEFS ./dirdef/reens.def

MAP ens_cbank.mb_acct, TARGET ens_cbank.mb_acct;
MAP ens_cbank.ac_subject, TARGET ens_cbank.ac_subject;
MAP ens_cbank.mb_acct_settle_info, TARGET ens_cbank.mb_acct_settle_info;
MAP ens_cbank.mb_guar_pledge_info, TARGET ens_cbank.mb_guar_pledge_info;
MAP ens_cbank.stc_tran_hist, TARGET ens_cbank.stc_tran_hist;
MAP ens_cbank.stc_acct, TARGET ens_cbank.stc_acct;
MAP ens_cbank.stc_tax_dtl, TARGET ens_cbank.stc_tax_dtl;
MAP ens_cbank.stc_income_dtl, TARGET ens_cbank.stc_income_dtl;
MAP ens_cbank.cif_settle_default, TARGET ens_cbank.cif_settle_default;
MAP ens_cbank.mb_internal_acct_mapping, TARGET ens_cbank.mb_internal_acct_mapping;
MAP ens_cbank.mb_open_close_reg, TARGET ens_cbank.mb_open_close_reg;
MAP ens_cbank.mb_acct_client_change, TARGET ens_cbank.mb_acct_client_change;
MAP ens_cbank.mb_acct_limit, TARGET ens_cbank.mb_acct_limit;
MAP ens_cbank.mb_acct_attach, TARGET ens_cbank.mb_acct_attach;
MAP ens_cbank.mb_acct, TARGET ens_cbank.mb_acct;
MAP ens_cbank.mb_acct_stats, TARGET ens_cbank.mb_acct_stats;
MAP ens_cbank.mb_acct_schedule_amend_hist, TARGET ens_cbank.mb_acct_schedule_amend_hist;
MAP ens_cbank.mb_acct_schedule, TARGET ens_cbank.mb_acct_schedule;
MAP ens_cbank.mb_acct_schedule_hist, TARGET ens_cbank.mb_acct_schedule_hist;
MAP ens_cbank.mb_acct_schedule_detail, TARGET ens_cbank.mb_acct_schedule_detail;
MAP ens_cbank.mb_acct_tran_stats, TARGET ens_cbank.mb_acct_tran_stats;
MAP ens_cbank.mb_acct_int_matrix, TARGET ens_cbank.mb_acct_int_matrix;
MAP ens_cbank.mb_acct_int_detail_split, TARGET ens_cbank.mb_acct_int_detail_split;
MAP ens_cbank.mb_acct_linkman_type, TARGET ens_cbank.mb_acct_linkman_type;
MAP ens_cbank.mb_contact_list, TARGET ens_cbank.mb_contact_list;
MAP ens_cbank.mb_acct_sec_balance, TARGET ens_cbank.mb_acct_sec_balance;
MAP ens_cbank.mb_acct_balance, TARGET ens_cbank.mb_acct_balance;
MAP ens_cbank.mb_month_limit, TARGET ens_cbank.mb_month_limit;
MAP ens_cbank.mb_acct_withdraw_type, TARGET ens_cbank.mb_acct_withdraw_type;
MAP ens_cbank.fm_acct_client_relation, TARGET ens_cbank.fm_acct_client_relation;
MAP ens_cbank.mb_corp_pay_card_info, TARGET ens_cbank.mb_corp_pay_card_info;
MAP ens_cbank.cd_ic_elec_card, TARGET ens_cbank.cd_ic_elec_card;
MAP ens_cbank.cd_card_bin, TARGET ens_cbank.cd_card_bin;
MAP ens_cbank.cd_card_prod_change, TARGET ens_cbank.cd_card_prod_change;
MAP ens_cbank.cd_ic_prod_def, TARGET ens_cbank.cd_ic_prod_def;
MAP ens_cbank.cd_cancel_reg, TARGET ens_cbank.cd_cancel_reg;
MAP ens_cbank.card_acct_exchange, TARGET ens_cbank.card_acct_exchange;
MAP ens_cbank.cd_tran_desc, TARGET ens_cbank.cd_tran_desc;
MAP ens_cbank.cd_card_journal, TARGET ens_cbank.cd_card_journal;
MAP ens_cbank.cd_card_chg, TARGET ens_cbank.cd_card_chg;
MAP ens_cbank.cd_card_arch, TARGET ens_cbank.cd_card_arch;
MAP ens_cbank.mb_card_book_exchg, TARGET ens_cbank.mb_card_book_exchg;
MAP ens_cbank.fx_mb_other_tran_hist, TARGET ens_cbank.fx_mb_other_tran_hist;
MAP ens_cbank.mb_open_close_reg, TARGET ens_cbank.mb_open_close_reg;
MAP ens_cbank.gl_prod_accounting, TARGET ens_cbank.gl_prod_accounting;
MAP ens_cbank.gl_prod_code_mapping, TARGET ens_cbank.gl_prod_code_mapping;
MAP ens_cbank.ac_subject, TARGET ens_cbank.ac_subject;
MAP ens_cbank.dc_branch_amt, TARGET ens_cbank.dc_branch_amt;
MAP ens_cbank.fm_branch, TARGET ens_cbank.fm_branch;
MAP ens_cbank.fm_bank, TARGET ens_cbank.fm_bank;
MAP ens_cbank.fm_external_branch, TARGET ens_cbank.fm_external_branch;
MAP ens_cbank.fm_user, TARGET ens_cbank.fm_user;
MAP ens_cbank.irl_prod_type, TARGET ens_cbank.irl_prod_type;
MAP ens_cbank.mb_prod_type, TARGET ens_cbank.mb_prod_type;
MAP ens_cbank.mb_prod_charge, TARGET ens_cbank.mb_prod_charge;
MAP ens_cbank.irl_prod_int, TARGET ens_cbank.irl_prod_int;
MAP ens_cbank.mb_change_prod, TARGET ens_cbank.mb_change_prod;
MAP ens_cbank.mb_renewal_type, TARGET ens_cbank.mb_renewal_type;
MAP ens_cbank.cif_sign_prod, TARGET ens_cbank.cif_sign_prod;
MAP ens_cbank.fm_channel, TARGET ens_cbank.fm_channel;
MAP ens_cbank.dc_channel_amt, TARGET ens_cbank.dc_channel_amt;
MAP ens_cbank.mb_channel_restraints, TARGET ens_cbank.mb_channel_restraints;
MAP ens_cbank.mb_acct_settle_info, TARGET ens_cbank.mb_acct_settle_info;
MAP ens_cbank.mb_manual_detail, TARGET ens_cbank.mb_manual_detail;
MAP ens_cbank.mb_tda_change, TARGET ens_cbank.mb_tda_change;
MAP ens_cbank.pt_payment_tran_hist, TARGET ens_cbank.pt_payment_tran_hist;
MAP ens_cbank.fx_mb_qr_tran_hist, TARGET ens_cbank.fx_mb_qr_tran_hist;
MAP ens_cbank.irl_accr_info, TARGET ens_cbank.irl_accr_info;
MAP ens_cbank.irl_capt_matrix, TARGET ens_cbank.irl_capt_matrix;
MAP ens_cbank.irl_capt_rate_info, TARGET ens_cbank.irl_capt_rate_info;
MAP ens_cbank.irl_capt_info, TARGET ens_cbank.irl_capt_info;
MAP ens_cbank.tb_tran_hist, TARGET ens_cbank.tb_tran_hist;
MAP ens_cbank.mb_channel_tran_hist, TARGET ens_cbank.mb_channel_tran_hist;
MAP ens_cbank.fx_mb_payment_tran_hist, TARGET ens_cbank.fx_mb_payment_tran_hist;
MAP ens_cbank.mb_tran_hist, TARGET ens_cbank.mb_tran_hist;
MAP ens_cbank.fm_tran_info, TARGET ens_cbank.fm_tran_info;
MAP ens_cbank.fm_tran_info_hist, TARGET ens_cbank.fm_tran_info_hist;
MAP ens_cbank.mb_misc_hist, TARGET ens_cbank.mb_misc_hist;
MAP ens_cbank.mb_ad_register_hist, TARGET ens_cbank.mb_ad_register_hist;
MAP ens_cbank.mb_ac_hist, TARGET ens_cbank.mb_ac_hist;
MAP ens_cbank.cif_client_corp, TARGET ens_cbank.cif_client_corp;
MAP ens_cbank.cif_client_indvl, TARGET ens_cbank.cif_client_indvl;
MAP ens_cbank.cif_client_used_info, TARGET ens_cbank.cif_client_used_info;
MAP ens_cbank.cif_client_block, TARGET ens_cbank.cif_client_block;
MAP ens_cbank.lm_client_tran_limit, TARGET ens_cbank.lm_client_tran_limit;
MAP ens_cbank.fm_acct_exec, TARGET ens_cbank.fm_acct_exec;
MAP ens_cbank.cif_client_commission, TARGET ens_cbank.cif_client_commission;
MAP ens_cbank.cif_client_type, TARGET ens_cbank.cif_client_type;
MAP ens_cbank.cif_category_type, TARGET ens_cbank.cif_category_type;
MAP ens_cbank.cif_client_contact_address, TARGET ens_cbank.cif_client_contact_address;
MAP ens_cbank.cif_client_contact_tbl, TARGET ens_cbank.cif_client_contact_tbl;
MAP ens_cbank.cif_client_rate, TARGET ens_cbank.cif_client_rate;
MAP ens_cbank.cif_client, TARGET ens_cbank.cif_client;
MAP ens_cbank.cif_client_document, TARGET ens_cbank.cif_client_document;
MAP ens_cbank.amend_branch, TARGET ens_cbank.amend_branch;
MAP ens_cbank.fm_acct_client_relation, TARGET ens_cbank.fm_acct_client_relation;
MAP ens_cbank.tb_attr_info, TARGET ens_cbank.tb_attr_info;
MAP ens_cbank.tb_trailbox, TARGET ens_cbank.tb_trailbox;
MAP ens_cbank.tb_cash_exchange, TARGET ens_cbank.tb_cash_exchange;
MAP ens_cbank.tb_trailbox_plan, TARGET ens_cbank.tb_trailbox_plan;
MAP ens_cbank.tb_trailbox_plan_detail, TARGET ens_cbank.tb_trailbox_plan_detail;
MAP ens_cbank.tb_trailbox_overdraw, TARGET ens_cbank.tb_trailbox_overdraw;
MAP ens_cbank.tb_cash_journal, TARGET ens_cbank.tb_cash_journal;
MAP ens_cbank.tb_cash_balance, TARGET ens_cbank.tb_cash_balance;
MAP ens_cbank.tb_cash_balance_detail, TARGET ens_cbank.tb_cash_balance_detail;
MAP ens_cbank.fm_default_rate_type, TARGET ens_cbank.fm_default_rate_type;
MAP ens_cbank.irl_exchange_type, TARGET ens_cbank.irl_exchange_type;
MAP ens_cbank.irl_ccy_rate, TARGET ens_cbank.irl_ccy_rate;
MAP ens_cbank.irl_fwd_ccy_rate, TARGET ens_cbank.irl_fwd_ccy_rate;
MAP ens_cbank.irl_after_acct_matrix_hist, TARGET ens_cbank.irl_after_acct_matrix_hist;
MAP ens_cbank.irl_after_acct_matrix, TARGET ens_cbank.irl_after_acct_matrix;
MAP ens_cbank.irl_acct_matrix, TARGET ens_cbank.irl_acct_matrix;
MAP ens_cbank.irl_int_basis, TARGET ens_cbank.irl_int_basis;
MAP ens_cbank.irl_basis_rate, TARGET ens_cbank.irl_basis_rate;
MAP ens_cbank.irl_roll_info, TARGET ens_cbank.irl_roll_info;
MAP ens_cbank.irl_int_roll, TARGET ens_cbank.irl_int_roll;
MAP ens_cbank.irl_int_matrix, TARGET ens_cbank.irl_int_matrix;
MAP ens_cbank.irl_int_type, TARGET ens_cbank.irl_int_type;
MAP ens_cbank.irl_int_rate, TARGET ens_cbank.irl_int_rate;
MAP ens_cbank.irl_int_rate_hist, TARGET ens_cbank.irl_int_rate_hist;
MAP ens_cbank.date_rate_alter_table, TARGET ens_cbank.date_rate_alter_table;
MAP ens_cbank.mb_acct_int_matrix, TARGET ens_cbank.mb_acct_int_matrix;
MAP ens_cbank.mb_agreement_assemble, TARGET ens_cbank.mb_agreement_assemble;
MAP ens_cbank.mb_agreement_ass_detail, TARGET ens_cbank.mb_agreement_ass_detail;
MAP ens_cbank.mb_agreement_agency, TARGET ens_cbank.mb_agreement_agency;
MAP ens_cbank.mb_agreement_loan, TARGET ens_cbank.mb_agreement_loan;
MAP ens_cbank.mb_agreement_sms, TARGET ens_cbank.mb_agreement_sms;
MAP ens_cbank.mb_agreement_package, TARGET ens_cbank.mb_agreement_package;
MAP ens_cbank.mb_agre_relation, TARGET ens_cbank.mb_agre_relation;
MAP ens_cbank.mb_agreement_fund, TARGET ens_cbank.mb_agreement_fund;
MAP ens_cbank.fm_org_agre_constract, TARGET ens_cbank.fm_org_agre_constract;
MAP ens_cbank.mb_agreement_rising, TARGET ens_cbank.mb_agreement_rising;
MAP ens_cbank.mb_financial_acct, TARGET ens_cbank.mb_financial_acct;
MAP ens_cbank.mb_dra_amountlimit, TARGET ens_cbank.mb_dra_amountlimit;
MAP ens_cbank.mb_agreement_overdraft, TARGET ens_cbank.mb_agreement_overdraft;
MAP ens_cbank.mb_agreement_loploan, TARGET ens_cbank.mb_agreement_loploan;
MAP ens_cbank.mb_agreement_tran_charge, TARGET ens_cbank.mb_agreement_tran_charge;
MAP ens_cbank.mb_agreement_agency, TARGET ens_cbank.mb_agreement_agency;
MAP ens_cbank.mb_agency_info, TARGET ens_cbank.mb_agency_info;
MAP ens_cbank.mb_agency_details, TARGET ens_cbank.mb_agency_details;
MAP ens_cbank.mb_acct_composite_schedule, TARGET ens_cbank.mb_acct_composite_schedule;
MAP ens_cbank.mb_pn_payment, TARGET ens_cbank.mb_pn_payment;
MAP ens_cbank.mb_pn_lost, TARGET ens_cbank.mb_pn_lost;
MAP ens_cbank.mb_pn_tran_detail, TARGET ens_cbank.mb_pn_tran_detail;
MAP ens_cbank.mb_pn_register, TARGET ens_cbank.mb_pn_register;
MAP ens_cbank.mb_cheque_dud, TARGET ens_cbank.mb_cheque_dud;
MAP ens_cbank.mb_acct_discount, TARGET ens_cbank.mb_acct_discount;
MAP ens_cbank.nx_oc_voucher_register, TARGET ens_cbank.nx_oc_voucher_register;
MAP ens_cbank.nx_oc_internal_acct_mapping, TARGET ens_cbank.nx_oc_internal_acct_mapping;
MAP ens_cbank.oc_voucher_register, TARGET ens_cbank.oc_voucher_register;
MAP ens_cbank.oc_tran_hist, TARGET ens_cbank.oc_tran_hist;
MAP ens_cbank.mb_ad_register, TARGET ens_cbank.mb_ad_register;
MAP ens_cbank.mb_receipt, TARGET ens_cbank.mb_receipt;
MAP ens_cbank.mb_receipt_detail, TARGET ens_cbank.mb_receipt_detail;
MAP ens_cbank.rc_document_mapping, TARGET ens_cbank.rc_document_mapping;
```

# 初始化数据

## 源端
```bash
./extract paramfile ./dirprm/initens.prm
```

## 目标端
```bash
./replicat paramfile ./dirprm/initens.prm
```
