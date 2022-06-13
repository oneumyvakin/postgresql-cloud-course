# Работа с кластером высокой доступности
## Вариант 2: pg_auto_failover

1. Сразу нужно обозначить, что решение с `pg_auto_failover` не делает PostgreSQL-кластер высокодоступным, 
потому что компонент `monitor` не является высокодоступным и вендор не предоставляет инструментов для решения этой [проблемы](https://pg-auto-failover.readthedocs.io/en/master/faq.html#the-monitor-is-a-spof-in-pg-auto-failover-design-how-should-we-handle-that). 
2. Создадим три сервера для монитора, мастера и реплики.
3. Установим `pg_auto_failover` на сервер для монитора:
```shell
echo "deb [signed-by=/usr/share/keyrings/download-archive-keyring.gpg] http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | tee /etc/apt/sources.list.d/pgdg.list
curl https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor > /usr/share/keyrings/download-archive-keyring.gpg
apt update
apt install pg-auto-failover-cli
apt install postgresql-14-auto-failover
export PGDATA=/var/lib/postgresql/monitor
export PGPORT=5000
sudo --preserve-env=PGDATA,PGPORT -u postgres pg_autoctl create monitor --ssl-self-signed --hostname 10.54.2.4 --auth trust
pg_autoctl -q show systemd --pgdata "/var/lib/postgresql/monitor" | tee /etc/systemd/system/pgautofailover.service
systemctl daemon-reload
systemctl enable pgautofailover
systemctl start pgautofailover
```
4. Установим `pg_auto_failover` на сервер для мастера:
```shell
export PGDATA=/var/lib/postgresql/node1
export PGPORT=5000
sudo --preserve-env=PGDATA,PGPORT -u postgres pg_autoctl create postgres --hostname 10.54.2.1 --auth trust --ssl-self-signed --monitor 'postgres://autoctl_node@10.54.2.4:5000/pg_auto_failover?sslmode=require'
pg_autoctl -q show systemd --pgdata "/var/lib/postgresql/node1" | tee /etc/systemd/system/pgautofailover.service
systemctl daemon-reload
systemctl enable pgautofailover
systemctl start pgautofailover
```
5. Установим `pg_auto_failover` на сервер для реплики:
```shell
export PGDATA=/var/lib/postgresql/node2
export PGPORT=5000
sudo --preserve-env=PGDATA,PGPORT -u postgres pg_autoctl create postgres --hostname 10.54.2.2 --auth trust --ssl-self-signed --monitor 'postgres://autoctl_node@10.54.2.4:5000/pg_auto_failover?sslmode=require'
pg_autoctl -q show systemd --pgdata "/var/lib/postgresql/node1" | tee /etc/systemd/system/pgautofailover.service
systemctl daemon-reload
systemctl enable pgautofailover
systemctl start pgautofailover
```
6. Проверим статус кластера:
```shell
root@db-monitor:~# sudo -u postgres pg_autoctl show state --pgdata /var/lib/postgresql/monitor/
   Name |  Node |      Host:Port |       TLI: LSN |   Connection |      Reported State |      Assigned State
--------+-------+----------------+----------------+--------------+---------------------+--------------------
 node_1 |     1 | 10.54.2.1:5000 |   2: 0/C002288 |   read-write |             primary |             primary
node_21 |    21 | 10.54.2.2:5000 |   2: 0/C002288 |    read-only |           secondary |           secondary

root@db-monitor:~#
```
7. Подключимся к мастеру:
```shell
root@db1:~# sudo -i -u postgres psql -p 5000
psql (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# CREATE DATABASE otus;
CREATE DATABASE
postgres=#
```
8. Проверим, что данные отреплицировались на реплике и реплика находится в `read-only` режиме:
```shell
root@db2:~# sudo -i -u postgres psql -p 5000
psql (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
-----------+----------+----------+---------+---------+-----------------------
 otus      | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(4 rows)

postgres=# CREATE DATABASE otus;
ERROR:  cannot execute CREATE DATABASE in a read-only transaction
```
9. Выполним `switchover`:
```shell
root@db-monitor:~#  sudo --preserve-env=PGDATA -u postgres pg_autoctl perform switchover
17:40:14 36916 INFO  Waiting 60 secs for a notification with state "primary" in formation "default" and group 0
17:40:14 36916 INFO  Listening monitor notifications about state changes in formation "default" and group 0
17:40:14 36916 INFO  Following table displays times when notifications are received
    Time |    Name |  Node |      Host:Port |       Current State |      Assigned State
---------+---------+-------+----------------+---------------------+--------------------
17:40:14 |  node_1 |     1 | 10.54.2.1:5000 |             primary |            draining
17:40:14 | node_21 |    21 | 10.54.2.2:5000 |           secondary |   prepare_promotion
17:40:14 | node_21 |    21 | 10.54.2.2:5000 |   prepare_promotion |   prepare_promotion
17:40:14 | node_21 |    21 | 10.54.2.2:5000 |   prepare_promotion |    stop_replication
17:40:14 |  node_1 |     1 | 10.54.2.1:5000 |             primary |      demote_timeout
17:40:14 |  node_1 |     1 | 10.54.2.1:5000 |            draining |      demote_timeout
17:40:14 |  node_1 |     1 | 10.54.2.1:5000 |      demote_timeout |      demote_timeout
17:40:15 | node_21 |    21 | 10.54.2.2:5000 |    stop_replication |    stop_replication
17:40:15 | node_21 |    21 | 10.54.2.2:5000 |    stop_replication |        wait_primary
17:40:15 |  node_1 |     1 | 10.54.2.1:5000 |      demote_timeout |             demoted
17:40:15 |  node_1 |     1 | 10.54.2.1:5000 |             demoted |             demoted
17:40:15 | node_21 |    21 | 10.54.2.2:5000 |        wait_primary |        wait_primary
17:40:15 |  node_1 |     1 | 10.54.2.1:5000 |             demoted |          catchingup
17:40:23 |  node_1 |     1 | 10.54.2.1:5000 |             demoted |          catchingup
17:40:37 |  node_1 |     1 | 10.54.2.1:5000 |          catchingup |          catchingup
17:40:38 |  node_1 |     1 | 10.54.2.1:5000 |          catchingup |          catchingup
17:40:38 |  node_1 |     1 | 10.54.2.1:5000 |          catchingup |           secondary
17:40:38 |  node_1 |     1 | 10.54.2.1:5000 |           secondary |           secondary
17:40:38 | node_21 |    21 | 10.54.2.2:5000 |        wait_primary |             primary
17:40:38 | node_21 |    21 | 10.54.2.2:5000 |             primary |             primary
root@db-monitor:~# 
root@db-monitor:~# sudo -u postgres pg_autoctl show state --pgdata /var/lib/postgresql/monitor/
   Name |  Node |      Host:Port |       TLI: LSN |   Connection |      Reported State |      Assigned State
--------+-------+----------------+----------------+--------------+---------------------+--------------------
 node_1 |     1 | 10.54.2.1:5000 |   3: 0/C003CB0 |    read-only |           secondary |           secondary
node_21 |    21 | 10.54.2.2:5000 |   3: 0/C003CB0 |   read-write |             primary |             primary
```
10. Проверим, что реплика перешла в режим `read-write`:
```shell
root@db2:~# sudo -i -u postgres psql -p 5000
psql (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# CREATE DATABASE otus;
ERROR:  database "otus" already exists
```
