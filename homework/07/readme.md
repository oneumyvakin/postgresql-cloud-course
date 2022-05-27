# Кластер Patroni

1. Создадим три виртуальные машины для etcd:
```shell
apt update && apt install -y etcd && 
```
2. Зададим конфигурацию кластера в `/etc/default/etcd`:
```shell
ETCD_NAME="etcd1"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://10.54.2.5:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.54.2.5:2380"
ETCD_INITIAL_CLUSTER_TOKEN="PatroniCluster"
ETCD_INITIAL_CLUSTER="etcd1=http://10.54.2.5:2380,etcd2=http://10.54.2.6:2380,etcd3=http://10.54.2.13:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
```

```shell
ETCD_NAME="etcd2"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://10.54.2.6:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.54.2.6:2380"
ETCD_INITIAL_CLUSTER_TOKEN="PatroniCluster"
ETCD_INITIAL_CLUSTER="etcd1=http://10.54.2.5:2380,etcd2=http://10.54.2.6:2380,etcd3=http://10.54.2.13:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
```

```shell
ETCD_NAME="etcd3"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://10.54.2.13:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.54.2.13:2380"
ETCD_INITIAL_CLUSTER_TOKEN="PatroniCluster"
ETCD_INITIAL_CLUSTER="etcd1=http://10.54.2.5:2380,etcd2=http://10.54.2.6:2380,etcd3=http://10.54.2.13:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
```
3. Проверим статус кластера etcd:
```shell
root@etcd3:~# systemctl restart etcd
root@etcd3:~# etcdctl cluster-health
member 3232fca3eb720b04 is healthy: got healthy result from http://10.54.2.5:2379
member 40e60d4c1725a9ea is healthy: got healthy result from http://10.54.2.13:2379
member c65a1a3241093baa is healthy: got healthy result from http://10.54.2.6:2379
cluster is healthy
root@etcd3:~#
```
4. Создадим три виртуальные машины для клстера Patroni и установим на них PostgreSQL.
5. Установим Patroni на машины с PostgreSQL
```shell
apt install -y python3 python3-pip
pip3 install psycopg2-binary
systemctl stop postgresql@14-main
sudo -u postgres pg_dropcluster 14 main 
pip3 install patroni[etcd]
systemctl enable patroni && systemctl start patroni
```

<details>
  <summary>Конфиг Patroni /etc/patroni.yml</summary>

```
scope: patroni
#namespace: /service/
name: pgsql1

restapi:
  listen: 10.54.2.1:8008
  connect_address: 10.54.2.1:8008
#  certfile: /etc/ssl/certs/ssl-cert-snakeoil.pem
#  keyfile: /etc/ssl/private/ssl-cert-snakeoil.key
#  authentication:
#    username: username
#    password: password

# ctl:
#   insecure: false # Allow connections to SSL sites without certs
#   certfile: /etc/ssl/certs/ssl-cert-snakeoil.pem
#   cacert: /etc/ssl/certs/ssl-cacert-snakeoil.pem

etcd:
  #Provide host to do the initial discovery of the cluster topology:
  #host: 127.0.0.1:2379
  #Or use "hosts" to provide multiple endpoints
  #Could be a comma separated string:
  hosts: 10.54.2.5:2379,10.54.2.6:2379,10.54.2.13:2379
  #or an actual yaml list:
  #hosts:
  #- host1:port1
  #- host2:port2
  #Once discovery is complete Patroni will use the list of advertised clientURLs
  #It is possible to change this behavior through by setting:
  #use_proxies: true

#raft:
#  data_dir: .
#  self_addr: 127.0.0.1:2222
#  partner_addrs:
#  - 127.0.0.1:2223
#  - 127.0.0.1:2224

bootstrap:
  # this section will be written into Etcd:/<namespace>/<scope>/config after initializing new cluster
  # and all other cluster members will use it as a `global configuration`
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
#    master_start_timeout: 300
#    synchronous_mode: false
    #standby_cluster:
      #host: 127.0.0.1
      #port: 1111
      #primary_slot_name: patroni
    postgresql:
      use_pg_rewind: true
#      use_slots: true
      parameters:
        max_connections: 100
#        wal_level: hot_standby
#        hot_standby: "on"
#        max_connections: 100
#        max_worker_processes: 8
#        wal_keep_segments: 8
#        max_wal_senders: 10
#        max_replication_slots: 10
#        max_prepared_transactions: 0
#        max_locks_per_transaction: 64
#        wal_log_hints: "on"
#        track_commit_timestamp: "off"
#        archive_mode: "on"
#        archive_timeout: 1800s
#        archive_command: mkdir -p ../wal_archive && test ! -f ../wal_archive/%f && cp %p ../wal_archive/%f
#      recovery_conf:
#        restore_command: cp ../wal_archive/%f %p

  # some desired options for 'initdb'
  initdb:  # Note: It needs to be a list (some options need values, others are switches)
  - encoding: UTF8
  - data-checksums

  pg_hba:  # Add following lines to pg_hba.conf after running 'initdb'
  # For kerberos gss based connectivity (discard @.*$)
  #- host replication replicator 127.0.0.1/32 gss include_realm=0
  #- host all all 0.0.0.0/0 gss include_realm=0
  - host replication replicator 10.0.0.0/8 md5
  - host all all 10.0.0.0/8 md5
#  - hostssl all all 0.0.0.0/0 md5

  # Additional script to be launched after initial cluster creation (will be passed the connection URL as parameter)
# post_init: /usr/local/bin/setup_cluster.sh

  # Some additional users users which needs to be created after initializing new cluster
  users:
    admin:
      password: admin_321
      options:
        - createrole
        - createdb

postgresql:
  listen: 127.0.0.1, 10.54.2.1:5432
  connect_address: 10.54.2.1:5432
  data_dir: /var/lib/postgresql/14/main
  bin_dir: /usr/lib/postgresql/14/bin
#  config_dir:
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: rep-pass_321
    superuser:
      username: postgres
      password: zalando_321
    rewind:  # Has no effect on postgres 10 and lower
      username: rewind_user
      password: rewind_password_321
  # Server side kerberos spn
#  krbsrvname: postgres
  parameters:
    # Fully qualified kerberos ticket file for the running user
    # same as KRB5CCNAME used by the GSS
#   krb_server_keyfile: /var/spool/keytabs/postgres
    unix_socket_directories: '.'
  # Additional fencing script executed after acquiring the leader lock but before promoting the replica
  #pre_promote: /path/to/pre_promote.sh

#watchdog:
#  mode: automatic # Allowed values: off, automatic, required
#  device: /dev/watchdog
#  safety_margin: 5

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```
</details>

<details>
  <summary>Конфиг сервиса Patroni /etc/systemd/system/patroni.service</summary>

```
# This is an example systemd config file for Patroni
# You can copy it to "/etc/systemd/system/patroni.service",

[Unit]
Description=Runners to orchestrate a high-availability PostgreSQL
After=syslog.target network.target

[Service]
Type=simple

User=postgres
Group=postgres

# Read in configuration file if it exists, otherwise proceed
EnvironmentFile=-/etc/patroni_env.conf

# the default is the user's home directory, and if you want to change it, you must provide an absolute path.
# WorkingDirectory=/home/sameuser

# Where to send early-startup messages from the server
# This is normally controlled by the global default set by systemd
#StandardOutput=syslog

# Pre-commands to start watchdog device
# Uncomment if watchdog is part of your patroni setup
#ExecStartPre=-/usr/bin/sudo /sbin/modprobe softdog
#ExecStartPre=-/usr/bin/sudo /bin/chown postgres /dev/watchdog

# Start the patroni process
ExecStart=/usr/local/bin/patroni /etc/patroni.yml

# Send HUP to reload from patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID

# only kill the patroni process, not it's children, so it will gracefully stop postgres
KillMode=process

# Give a reasonable amount of time for the server to start up/shut down
TimeoutSec=30

# Do not restart the service if it crashes, we want to manually inspect database on failure
Restart=no

[Install]
WantedBy=multi-user.target
```
</details>

6. Проверим статус кластера Patroni:
```shell
~# patronictl -c /etc/patroni.yml list
+--------+-----------+---------+---------+----+-----------+
| Member | Host      | Role    | State   | TL | Lag in MB |
+ Cluster: patroni (7102258055171846244) +----+-----------+
| pgsql1 | 10.54.2.1 | Leader  | running |  1 |           |
| pgsql2 | 10.54.2.2 | Replica | running |  1 |         0 |
| pgsql3 | 10.54.2.4 | Replica | running |  1 |         0 |
+--------+-----------+---------+---------+----+-----------+
```
7. Установим pgbouncer:
```shell
sudo -i -u postgres psql  -h 127.0.0.1 -c "select passwd from pg_shadow where usename='postgres';"
                                                                passwd
---------------------------------------------------------------------------------------------------------------------------------------
 SCRAM-SHA-256$4096:HRInwli9HLH5iUBWzUmRFA==$fzZzetJfOf/hvxyffGrb/laoHIHAwuERDw9DQ8tSFFw=:Hh2yh0FdK5kZTivzOB9xVBE61BBXsAdMIdnyNFngCtI=
(1 row)

~# cat /etc/pgbouncer/userlist.txt
"postgres"  "SCRAM-SHA-256$4096:HRInwli9HLH5iUBWzUmRFA==$fzZzetJfOf/hvxyffGrb/laoHIHAwuERDw9DQ8tSFFw=:Hh2yh0FdK5kZTivzOB9xVBE61BBXsAdMIdnyNFngCtI="

~# cat /etc/pgbouncer/pgbouncer.ini
[databases]
* = host=127.0.0.1 port=5432
[pgbouncer]
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid
listen_addr = *
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
admin_users = postgres

~# apt install pgbouncer
```
8. Проверим подключение к PostgreSQL через pgbouncer к базе данных `postgres`:
```shell
~# export PGPASSWORD=zalando_321
~# sudo --preserve-env=PGPASSWORD -i -u postgres psql  -h 127.0.0.1 -p 6432 -c "select usename from pg_shadow;"
   usename
-------------
 postgres
 replicator
 rewind_user
 admin
(4 rows)
```
9. Установим HAProxy на отдельный сервер:
```shell
~# apt install haproxy
~# tail -n 11 /etc/haproxy/haproxy.cfg
listen postgres_write
    bind *:5432
    mode   tcp
    option httpchk
    http-check connect
    http-check send meth GET uri /master
    http-check expect status 200
    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions
    server pgsql1 10.54.2.1:6432 check port 8008
    server pgsql2 10.54.2.2:6432 check port 8008
    server pgsql3 10.54.2.4:6432 check port 8008
```
10. Проверим подключение к PostgreSQL через HAProxy:
```shell
root@haproxy:~# PGPASSWORD=zalando_321 psql -p 5432 -d postgres -h 10.54.2.55 -U postgres
psql (14.3 (Ubuntu 14.3-0ubuntu0.22.04.1))
Type "help" for help.

postgres=#
```
11. Проверим отказоустойчивость кластера PostgreSQL:
Остановим сервер с лидером:
```shell
root@db1:~# patronictl -c /etc/patroni.yml list
+--------+-----------+---------+---------+----+-----------+
| Member | Host      | Role    | State   | TL | Lag in MB |
+ Cluster: patroni (7102258055171846244) +----+-----------+
| pgsql1 | 10.54.2.1 | Leader  | running |  1 |           |
| pgsql2 | 10.54.2.2 | Replica | running |  1 |         0 |
| pgsql3 | 10.54.2.4 | Replica | running |  1 |         0 |
+--------+-----------+---------+---------+----+-----------+
root@db1:~# shutdown -h now
```
Проверим, что лидером стал другой сервер:
```shell
root@db2:~# patronictl -c /etc/patroni.yml list
+--------+-----------+---------+---------+----+-----------+
| Member | Host      | Role    | State   | TL | Lag in MB |
+ Cluster: patroni (7102258055171846244) +----+-----------+
| pgsql1 | 10.54.2.1 | Replica | stopped |    |   unknown |
| pgsql2 | 10.54.2.2 | Leader  | running |  2 |           |
| pgsql3 | 10.54.2.4 | Replica | running |  2 |         0 |
+--------+-----------+---------+---------+----+-----------+
root@db2:~#
```
Проверим, что через HAProxy мы можем подключиться к мастеру PostgreSQL:
```shell
root@haproxy:~# PGPASSWORD=zalando_321 psql -p 5432 -d postgres -h 10.54.2.55 -U postgres
psql (14.3 (Ubuntu 14.3-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# CREATE DATABASE pgbench;
CREATE DATABASE
postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
-----------+----------+----------+---------+---------+-----------------------
 pgbench   | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(4 rows)

postgres=#
```
Запустим первый сервер:
```shell
root@db1:~# patronictl -c /etc/patroni.yml list
+--------+-----------+---------+---------+----+-----------+
| Member | Host      | Role    | State   | TL | Lag in MB |
+ Cluster: patroni (7102258055171846244) +----+-----------+
| pgsql1 | 10.54.2.1 | Replica | running |  2 |         0 |
| pgsql2 | 10.54.2.2 | Leader  | running |  2 |           |
| pgsql3 | 10.54.2.4 | Replica | running |  2 |         0 |
+--------+-----------+---------+---------+----+-----------+
root@db1:~#
```
Остановим второй сервер:
```shell
root@db1:~# patronictl -c /etc/patroni.yml list
+--------+-----------+---------+---------+----+-----------+
| Member | Host      | Role    | State   | TL | Lag in MB |
+ Cluster: patroni (7102258055171846244) +----+-----------+
| pgsql1 | 10.54.2.1 | Replica | running |  3 |         0 |
| pgsql2 | 10.54.2.2 | Replica | stopped |    |   unknown |
| pgsql3 | 10.54.2.4 | Leader  | running |  3 |           |
+--------+-----------+---------+---------+----+-----------+
root@db1:~#
```
Проверим, что через HAProxy мы можем подключиться к мастеру PostgreSQL:
```shell
root@haproxy:~# PGPASSWORD=zalando_321 psql -p 5432 -d postgres -h 10.54.2.55 -U postgres
psql (14.3 (Ubuntu 14.3-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# CREATE DATABASE pgbench2;
CREATE DATABASE
postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
-----------+----------+----------+---------+---------+-----------------------
 pgbench   | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 pgbench2  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(5 rows)

postgres=#
```

Запустим второй сервер и остановим третий:
```shell
root@db1:~# patronictl -c /etc/patroni.yml list
+--------+-----------+---------+---------+----+-----------+
| Member | Host      | Role    | State   | TL | Lag in MB |
+ Cluster: patroni (7102258055171846244) +----+-----------+
| pgsql1 | 10.54.2.1 | Leader  | running |  4 |           |
| pgsql2 | 10.54.2.2 | Replica | running |  4 |         0 |
| pgsql3 | 10.54.2.4 | Replica | stopped |    |   unknown |
+--------+-----------+---------+---------+----+-----------+
root@db1:~#
```
Проверим, что через HAProxy мы можем подключиться к мастеру PostgreSQL:
```shell
root@haproxy:~# PGPASSWORD=zalando_321 psql -p 5432 -d postgres -h 10.54.2.55 -U postgres
psql (14.3 (Ubuntu 14.3-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# CREATE DATABASE pgbench3;
CREATE DATABASE
postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
-----------+----------+----------+---------+---------+-----------------------
 pgbench   | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 pgbench2  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 pgbench3  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(6 rows)

postgres=#
```
