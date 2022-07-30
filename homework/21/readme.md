#  PostgreSQL и VKcloud 

1. Создадим новый интанс базы данных типа "кластер" из трех нод и пользователем `user`.
2. Подключимся к базе данных:
```shell
~# psql -h89.208.229.159 -Uuser PostgreSQL-8882
Password for user user:
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1), server 13.7)
WARNING: psql major version 12, server major version 13.
         Some psql features might not work.
Type "help" for help.

PostgreSQL-8882=> \l
                                     List of databases
      Name       |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------------+----------+----------+-------------+-------------+-----------------------
 PostgreSQL-8882 | os_admin | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =Tc/os_admin         +
                 |          |          |             |             | os_admin=CTc/os_admin+
                 |          |          |             |             | user=CTc/os_admin
 os_admin        | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =Tc/postgres         +
                 |          |          |             |             | postgres=CTc/postgres+
                 |          |          |             |             | os_admin=CTc/postgres
 postgres        | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0       | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
                 |          |          |             |             | postgres=CTc/postgres
 template1       | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
                 |          |          |             |             | postgres=CTc/postgres
(5 rows)

PostgreSQL-8882=>
```
3. Создадим базу данных `benchmark` и дадим пользователю `user` к ней доступ.
```shell
PostgreSQL-8882=> \l
                                     List of databases
      Name       |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------------+----------+----------+-------------+-------------+-----------------------
 PostgreSQL-8882 | os_admin | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =Tc/os_admin         +
                 |          |          |             |             | os_admin=CTc/os_admin+
                 |          |          |             |             | user=CTc/os_admin
 benchmark       | os_admin | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =Tc/os_admin         +
                 |          |          |             |             | os_admin=CTc/os_admin+
                 |          |          |             |             | user=CTc/os_admin
 os_admin        | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =Tc/postgres         +
                 |          |          |             |             | postgres=CTc/postgres+
                 |          |          |             |             | os_admin=CTc/postgres
 postgres        | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0       | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
                 |          |          |             |             | postgres=CTc/postgres
 template1       | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
                 |          |          |             |             | postgres=CTc/postgres
(6 rows)

PostgreSQL-8882=> \c benchmark;
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1), server 13.7)
WARNING: psql major version 12, server major version 13.
         Some psql features might not work.
You are now connected to database "benchmark" as user "user".
benchmark=>
```
4. Протестируем:
```shell
~# pgbench -h89.208.229.159 -p5432 -Uuser -i -s 15 benchmark
dropping old tables...
creating tables...
generating data...
100000 of 1500000 tuples (6%) done (elapsed 0.54 s, remaining 7.62 s)
200000 of 1500000 tuples (13%) done (elapsed 0.67 s, remaining 4.33 s)
300000 of 1500000 tuples (20%) done (elapsed 0.76 s, remaining 3.06 s)
400000 of 1500000 tuples (26%) done (elapsed 0.95 s, remaining 2.62 s)
500000 of 1500000 tuples (33%) done (elapsed 1.15 s, remaining 2.29 s)
600000 of 1500000 tuples (40%) done (elapsed 1.32 s, remaining 1.98 s)
700000 of 1500000 tuples (46%) done (elapsed 1.50 s, remaining 1.72 s)
800000 of 1500000 tuples (53%) done (elapsed 1.68 s, remaining 1.47 s)
900000 of 1500000 tuples (60%) done (elapsed 1.86 s, remaining 1.24 s)
1000000 of 1500000 tuples (66%) done (elapsed 2.05 s, remaining 1.02 s)
1100000 of 1500000 tuples (73%) done (elapsed 2.29 s, remaining 0.83 s)
1200000 of 1500000 tuples (80%) done (elapsed 2.47 s, remaining 0.62 s)
1300000 of 1500000 tuples (86%) done (elapsed 2.70 s, remaining 0.42 s)
1400000 of 1500000 tuples (93%) done (elapsed 2.88 s, remaining 0.21 s)
1500000 of 1500000 tuples (100%) done (elapsed 3.06 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done.
~# 
~# pgbench -h89.208.229.159 -p5432 -Uuser -c 50 -j 2 -P 60 -T 180 benchmark
starting vacuum...end.
progress: 60.0 s, 45.6 tps, lat 916.899 ms stddev 340.707
progress: 120.0 s, 55.3 tps, lat 904.161 ms stddev 321.680
progress: 180.0 s, 56.0 tps, lat 893.886 ms stddev 292.265
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 15
query mode: simple
number of clients: 50
number of threads: 2
duration: 180 s
number of transactions actually processed: 9465
latency average = 905.149 ms
latency stddev = 318.035 ms
tps = 52.156983 (including connections establishing)
tps = 52.263582 (excluding connections establishing)
```
5. Создадим логическую резевную копию с помощью `pg_dump`:
```shell
~# pg_dump -h89.208.229.159 -p5432 -Uuser benchmark > pg_duml.sql
~# ls -lah pg_duml.sql
-rw-r--r-- 1 root root 139M Jul 30 08:54 pg_duml.sql
```
6. Создадим физическую резевную копию с помощью `pg_basebackup`. Для этого подключимся к серверу с синхронной репликой используя пользователя `admin` и SSH-ключ: 
```shell
[admin@postgresql-8882-member-2 ~]$ mkdir basebackup
[admin@postgresql-8882-member-2 ~]$ pg_basebackup -Upostgres -R -D ./basebackup/
[admin@postgresql-8882-member-2 ~]$ ls -la ./basebackup/
total 324
drwxrwxr-x. 21 admin admin   4096 Jul 30 09:41 .
drwx------.  4 admin admin    134 Jul 30 09:41 ..
-rwx------.  1 admin admin     36 Jul 30 09:41 .replpass
-rw-------.  1 admin admin      3 Jul 30 09:41 PG_VERSION
-rw-------.  1 admin admin    227 Jul 30 09:41 backup_label
-rw-------.  1 admin admin    224 Jul 30 09:41 backup_label.old
-rw-------.  1 admin admin 265818 Jul 30 09:41 backup_manifest
drwx------.  8 admin admin     80 Jul 30 09:41 base
drwx------.  2 admin admin   4096 Jul 30 09:41 global
drwx------.  2 admin admin     32 Jul 30 09:41 log
drwxr-xr-x.  2 admin admin     98 Jul 30 09:41 overrides
-rw-------.  1 admin admin    390 Jul 30 09:41 patroni.dynamic.json
drwx------.  2 admin admin      6 Jul 30 09:41 pg_commit_ts
drwx------.  2 admin admin      6 Jul 30 09:41 pg_dynshmem
-rwx------.  1 admin admin    408 Jul 30 09:41 pg_hba.conf
-rwx------.  1 admin admin    408 Jul 30 09:41 pg_hba.conf.backup
-rw-r-----.  1 admin admin   1636 Jul 30 09:41 pg_ident.conf
-rw-r-----.  1 admin admin   1636 Jul 30 09:41 pg_ident.conf.backup
drwx------.  4 admin admin     68 Jul 30 09:41 pg_logical
drwx------.  4 admin admin     36 Jul 30 09:41 pg_multixact
drwx------.  2 admin admin      6 Jul 30 09:41 pg_notify
drwx------.  2 admin admin      6 Jul 30 09:41 pg_replslot
drwx------.  2 admin admin      6 Jul 30 09:41 pg_serial
drwx------.  2 admin admin      6 Jul 30 09:41 pg_snapshots
drwx------.  2 admin admin      6 Jul 30 09:41 pg_stat
drwx------.  2 admin admin      6 Jul 30 09:41 pg_stat_tmp
drwx------.  2 admin admin      6 Jul 30 09:41 pg_subtrans
drwx------.  2 admin admin      6 Jul 30 09:41 pg_tblspc
drwx------.  2 admin admin      6 Jul 30 09:41 pg_twophase
drwx------.  3 admin admin     84 Jul 30 09:41 pg_wal
drwx------.  2 admin admin     18 Jul 30 09:41 pg_xact
-rw-r-----.  1 admin admin    317 Jul 30 09:41 postgresql.auto.conf
-rw-r-----.  1 admin admin    606 Jul 30 09:41 postgresql.base.conf
-rw-r-----.  1 admin admin    606 Jul 30 09:41 postgresql.base.conf.backup
-rw-r--r--.  1 admin admin   1032 Jul 30 09:41 postgresql.conf
-rw-r--r--.  1 admin admin   1032 Jul 30 09:41 postgresql.conf.backup
-rw-r--r--.  1 admin admin      0 Jul 30 09:41 standby.signal
[admin@postgresql-8882-member-2 ~]$
```
