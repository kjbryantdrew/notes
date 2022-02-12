# 修改

```bash
(kanas)[kanas@node43 kudu]$ pwd
/home/kanas/kanas/patch/kudu1.8/kudu-python-1.8.0/kudu
(kanas)[kanas@node43 kudu]$ vim client.pyx
(kanas)[kanas@node43 kudu]$
```

```python
# 增加长整型处理
# client.pyx
elif t == KUDU_STRING:
    if isinstance(value, unicode):
		    value = value.encode('utf8')
    elif isinstance(value, int):
        value = str(value)
    elif isinstance(value, long):
        value = str(value)

# schema.pyx
cdef class KuduValue:

    def __cinit__(self, KuduType type_, value):
        cdef:
            Slice slc

        if (type_.name[:3] == 'int'):
            self._value = C_KuduValue.FromInt(value)
        elif (type_.name in ['string', 'binary']):
            if isinstance(value, unicode):
                value = value.encode('utf8')
            # 增加整形 -> str
            elif isinstance(value, int):
                value = str(value)
            elif isinstance(value, long):
                value = str(value)

            slc = Slice(<char*> value, len(value))
            self._value = C_KuduValue.CopyString(slc)
```

# 打包

```bash
(kanas)[kanas@node43 kudu]$ pwd
/home/kanas/kanas/patch/kudu1.8/kudu-python-1.8.0/kudu
(kanas)[kanas@node43 kudu]$ ls
client.cpp  client.pyx  compat.py   config.pxi.in  errors.pxd  errors.so     __init__.py         schema.cpp  schema.pyx  tests    version.py
client.pxd  client.so   config.pxi  errors.cpp     errors.pyx  __init__.pxd  libkudu_client.pxd  schema.pxd  schema.so   util.py
(kanas)[kanas@node43 kudu]$ cd ..
(kanas)[kanas@node43 kudu-python-1.8.0]$ ls
build  dist  kudu  kudu_python.egg-info  MANIFEST.in  PKG-INFO  README.md  setup.cfg  setup.py
(kanas)[kanas@node43 kudu-python-1.8.0]$ pwd
/home/kanas/kanas/patch/kudu1.8/kudu-python-1.8.0
(kanas)[kanas@node43 kudu-python-1.8.0]$ python setup.py bdist_wheel

# 打包过后的原始文件路径
(kanas)[kanas@node43 lib.linux-x86_64-2.7]$ ls
kudu
(kanas)[kanas@node43 lib.linux-x86_64-2.7]$ pwd
/home/kanas/kanas/patch/kudu1.8/kudu-python-1.8.0/build/lib.linux-x86_64-2.7
```

# 补丁

