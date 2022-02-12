版本情况



```
Centos: 7.5 1804
CDH: 6.1.0
```



服务器



```
whbdp01: none
whbdp02: none
whbdp03: haproxy
whbdp04: none
```



在 whbdp03 上安装 haproxy



```
yum install haproxy
```



配置



```
cat /etc/haproxy/haproxy.cfg
#
# This sets up the admin page for HA Proxy at port 25002.
#
listen stats :25002
    balance
    mode http
    stats enable
    stats auth username:password
# This is the setup for Impala. Impala client connect to load_balancer_host:25003.
# HAProxy will balance connections among the list of servers listed below.
# The list of Impalad is listening at port 21000 for beeswax (impala-shell) or original ODBC driver.
# For JDBC or ODBC version 2.x driver, use port 21050 instead of 21000.
listen impala :21001
    mode tcp
    option tcplog
    balance leastconn
    server impalad_client_whbdp01 whbdp01:21000 check
    server impalad_client_whbdp02 whbdp02:21000 check
    server impalad_client_whbdp03 whbdp03:21000 check
    server impalad_client_whbdp04 whbdp04:21000 check
# Setup for Hue or other JDBC-enabled applications.
# In particular, Hue requires sticky sessions.
# The application connects to load_balancer_host:21051, and HAProxy balances
# connections to the associated hosts, where Impala listens for JDBC
# requests on port 21050.
listen impalajdbc :21051
    mode tcp
    option tcplog
    balance source
    server impalad_jdbc_whbdp01 whbdp01:21050
    server impalad_jdbc_whbdp02 whbdp02:21050
    server impalad_jdbc_whbdp03 whbdp03:21050
    server impalad_jdbc_whbdp04 whbdp04:21050
```



启动



```
/usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg
```