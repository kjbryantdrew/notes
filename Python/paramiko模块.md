```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from __future__ import unicode_literals
from __future__ import absolute_import

import paramiko
import os


# 通过密码登录执行命令
def cmd_by_passwd(host=None, user=None, passwd=None, port=None):
    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    ssh.connect("47.240.34.31", 22, "root", "Lx19961003.")
    stdin, stdout, stderr = ssh.exec_command('pwd')
    print(stdout.read().decode('utf-8'))
    ssh.close()


# 通过密码登录进行文件传输
def trans_by_passwd():

    # 文件上传
    t = paramiko.Transport(('47.240.34.31', 22))
    t.connect(username='root', password='Lx19961003.')
    sftp = paramiko.SFTPClient.from_transport(t)
    sftp.put('/tmp/a.log', '/tmp/b.log')
    print("upload success")
    t.close()

    # 文件下载
    t = paramiko.Transport(('47.240.34.31', 22))
    t.connect(username='root', password='Lx19961003.')
    sftp = paramiko.SFTPClient.from_transport(t)
    sftp.get('/tmp/b.log', '/tmp/b.log')
    print("download success")
    t.close()


# 通过密钥登录执行命令
def cmd_by_key():

    private_key_path = os.path.join(os.environ['HOME'], '.ssh/id_rsa')
    key = paramiko.RSAKey.from_private_key_file(private_key_path)

    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    ssh.connect('47.240.34.31', 22, 'root', key)

    stdin, stdout, stderr = ssh.exec_command('pwd', timeout=5)
    print(stdout.read())
    ssh.close()


# 通过密钥登录进行文件传输
def trans_by_key():

    # 上传文件
    private_key_path = os.path.join(os.environ['HOME'], '.ssh/id_rsa')
    key = paramiko.RSAKey.from_private_key_file(private_key_path)

    t = paramiko.Transport(('47.240.34.31', 22))
    t.connect(username='root', pkey=key)

    sftp = paramiko.SFTPClient.from_transport(t)
    sftp.put('/tmp/a.log', '/tmp/c.log')
    print("upload success")

    # 下载文件
    sftp = paramiko.SFTPClient.from_transport(t)
    sftp.get('/tmp/c.log', '/tmp/c.log')
    print("download success")

    t.close()


if __name__ == '__main__':
    #  cmd_by_passwd()
    #  trans_by_passwd()
    #  cmd_by_key()
    trans_by_key()

``````python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from __future__ import unicode_literals
from __future__ import absolute_import

import paramiko
import os


# 通过密码登录执行命令
def cmd_by_passwd(host=None, user=None, passwd=None, port=None):
    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    ssh.connect("47.240.34.31", 22, "root", "Lx19961003.")
    stdin, stdout, stderr = ssh.exec_command('pwd')
    print(stdout.read().decode('utf-8'))
    ssh.close()


# 通过密码登录进行文件传输
def trans_by_passwd():

    # 文件上传
    t = paramiko.Transport(('47.240.34.31', 22))
    t.connect(username='root', password='Lx19961003.')
    sftp = paramiko.SFTPClient.from_transport(t)
    sftp.put('/tmp/a.log', '/tmp/b.log')
    print("upload success")
    t.close()

    # 文件下载
    t = paramiko.Transport(('47.240.34.31', 22))
    t.connect(username='root', password='Lx19961003.')
    sftp = paramiko.SFTPClient.from_transport(t)
    sftp.get('/tmp/b.log', '/tmp/b.log')
    print("download success")
    t.close()


# 通过密钥登录执行命令
def cmd_by_key():

    private_key_path = os.path.join(os.environ['HOME'], '.ssh/id_rsa')
    key = paramiko.RSAKey.from_private_key_file(private_key_path)

    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    ssh.connect('47.240.34.31', 22, 'root', key)

    stdin, stdout, stderr = ssh.exec_command('pwd', timeout=5)
    print(stdout.read())
    ssh.close()


# 通过密钥登录进行文件传输
def trans_by_key():

    # 上传文件
    private_key_path = os.path.join(os.environ['HOME'], '.ssh/id_rsa')
    key = paramiko.RSAKey.from_private_key_file(private_key_path)

    t = paramiko.Transport(('47.240.34.31', 22))
    t.connect(username='root', pkey=key)

    sftp = paramiko.SFTPClient.from_transport(t)
    sftp.put('/tmp/a.log', '/tmp/c.log')
    print("upload success")

    # 下载文件
    sftp = paramiko.SFTPClient.from_transport(t)
    sftp.get('/tmp/c.log', '/tmp/c.log')
    print("download success")

    t.close()


if __name__ == '__main__':
    #  cmd_by_passwd()
    #  trans_by_passwd()
    #  cmd_by_key()
    trans_by_key()
```
