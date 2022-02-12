```bash
 python pump.py --db-source oracle+cx_oracle://sn:sn@leson:1521/orcl --db-sink "mysql+pymysql://sn:sn@localhost/sndb?charset=utf8" --database sn --schema sndb --fiscal-date 2021-08-15 --overwrite true
```

# 问题

MySQL中数据类型为`NUMERIC`，Oracle中为`NUMBER`

需要增加visit_NUMBER方法

```bash
vim .env/lib/python2.7/site-packages/sqlalchemy/dialects/mysql/base.py
```

```python
# NUMBER -> NUMRIC
def visit_NUMBER(self, type_, **kw):
    # if column is pk or have foreign_keys, it cat not be NUMERIC type
    is_pk = kw['type_expression'].primary_key
    foreign_keys = kw['type_expression'].foreign_keys
    if is_pk or foreign_keys:
        type_ = BIGINT
        return self._extend_numeric(type_, "BIGINT")
    if type_.precision is None:
        return self._extend_numeric(type_, "NUMERIC")
    elif type_.scale is None:
        return self._extend_numeric(type_,
                                    "NUMERIC(%(precision)s)" %
                                    {'precision': type_.precision})
    else:
        return self._extend_numeric(type_,
                                    "NUMERIC(%(precision)s, %(scale)s)" %
                                    {'precision': type_.precision,
                                     'scale': type_.scale})
      
# CLOB -> TEXT
def visit_CLOB(self, type_, **kw):
    return self._extend_string(TEXT, {}, "TEXT")
```

