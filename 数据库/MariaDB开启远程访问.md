```sql
MariaDB [(none)]> select User, host from mysql.user;
+-----------+-----------+
| User      | host      |
+-----------+-----------+
| root      | 127.0.0.1 |
| root      | ::1       |
| root      | leson     |
| root      | localhost |
| wordpress | localhost |
+-----------+-----------+
5 rows in set (0.01 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'lx1003' WITH GRANT OPTION;
Query OK, 0 rows affected (0.01 sec)

MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> select User, host from mysql.user;
+-----------+-----------+
| User      | host      |
+-----------+-----------+
| root      | %         |
| root      | 127.0.0.1 |
| root      | ::1       |
| root      | leson     |
| root      | localhost |
| wordpress | localhost |
+-----------+-----------+
6 rows in set (0.00 sec)
```