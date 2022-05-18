# Тюнинг PostgreSQL

1. Создадим виртуальную машину c Ubuntu 22.04 LTS на инстансе с 4 vCPU и 4 GB RAM.
2. Установим PostgreSQL 14:

```shell
~# echo "deb [signed-by=/usr/share/keyrings/download-archive-keyring.gpg] http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | tee /etc/apt/sources.list.d/pgdg.list
~# curl https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor > /usr/share/keyrings/download-archive-keyring.gpg
~# apt update
~# apt install -y postgresql-14
~# systemctl status postgresql@14-main.service
```
3. Протестируем инстанс с помощью `pg_bench` на дефолтных настройках PostgreSQL:
```shell
~# sudo -i -u postgres psql -c "create database pgbench;"
~# sudo -i -u postgres pgbench -i -s 10 pgbench
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
1000000 of 1000000 tuples (100%) done (elapsed 1.51 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 2.33 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 1.54 s, vacuum 0.25 s, primary keys 0.52 s).
~# 
~# 
~# sudo -i -u postgres pgbench -P 1 -T 180 pgbench
pgbench (14.3 (Ubuntu 14.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 5.0 s, 1069.6 tps, lat 0.933 ms stddev 0.117
...
progress: 180.0 s, 1035.2 tps, lat 0.966 ms stddev 0.119
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 1
number of threads: 1
duration: 180 s
number of transactions actually processed: 234965
latency average = 0.766 ms
latency stddev = 0.203 ms
initial connection time = 6.002 ms
tps = 1305.397952 (without initial connection time)
```
3. Посмотрим на top запросов по CPU c помощью `pg_stat_statements`:
```shell
                             short_query                             | total_time | calls | rows  | avg_time | percentage_cpu
---------------------------------------------------------------------+------------+-------+-------+----------+----------------
 UPDATE pgbench_accounts SET abalance = abalance + $1 WHERE aid = $2 |     479.75 | 13202 | 13202 |     0.04 |          43.76
 UPDATE pgbench_tellers SET tbalance = tbalance + $1 WHERE tid = $2  |     243.74 | 13202 | 13202 |     0.02 |          22.23
 UPDATE pgbench_branches SET bbalance = bbalance + $1 WHERE bid = $2 |     140.83 | 13202 | 13202 |     0.01 |          12.85
(3 rows)
```
4. Посмотрим на `explain` этих запросов в `pg_top`:
```shell
Current query plan for procpid 5910:

 Statement:

UPDATE pgbench_accounts SET abalance = abalance + -2214 WHERE aid = 360047;

Query Plan:

Update on pgbench_accounts  (cost=0.42..8.45 rows=0 width=0)
  ->  Index Scan using pgbench_accounts_pkey on pgbench_accounts  (cost=0.42..8.45 rows=1 width=10)
        Index Cond: (aid = 360047)
```

```shell
Current query plan for procpid 5910:

 Statement:

UPDATE pgbench_tellers SET tbalance = tbalance + -2946 WHERE tid = 90;

Query Plan:

Update on pgbench_tellers  (cost=0.14..8.16 rows=0 width=0)
  ->  Index Scan using pgbench_tellers_pkey on pgbench_tellers  (cost=0.14..8.16 rows=1 width=10)
        Index Cond: (tid = 90)
```

```shell
Current query plan for procpid 5910:

 Statement:

UPDATE pgbench_branches SET bbalance = bbalance + 665 WHERE bid = 7;

Query Plan:

Update on pgbench_branches  (cost=0.00..6.13 rows=0 width=0)
  ->  Seq Scan on pgbench_branches  (cost=0.00..6.13 rows=1 width=10)
        Filter: (bid = 7)
```
5. Посмотрим на размеры таблиц:
```shell
                                         List of relations
 Schema |       Name       | Type  |  Owner   | Persistence | Access method |  Size  | Description
--------+------------------+-------+----------+-------------+---------------+--------+-------------
 public | pgbench_accounts | table | postgres | permanent   | heap          | 133 MB |
 public | pgbench_branches | table | postgres | permanent   | heap          | 88 kB  |
 public | pgbench_history  | table | postgres | permanent   | heap          | 12 MB  |
 public | pgbench_tellers  | table | postgres | permanent   | heap          | 160 kB |
(4 rows)
```
6. Посмотрим на размеры индексов:
```shell
~# sudo -i -u postgres psql pgbench -c "SELECT version,index_size FROM pgstatindex('pgbench_accounts_pkey');"
 version | index_size
---------+------------
       4 |   22487040
(1 row)

~# sudo -i -u postgres psql pgbench -c "SELECT version,index_size FROM pgstatindex('pgbench_tellers_pkey');"
 version | index_size
---------+------------
       4 |      40960
(1 row)

~# sudo -i -u postgres psql pgbench -c "SELECT version,index_size FROM pgstatindex('pgbench_branches_pkey');"
 version | index_size
---------+------------
       4 |      16384
(1 row)
```

7. Попробуем применить настройки с сайта https://pgtune.leopard.in.ua/ и безопасные настройки с http://pgconfigurator.cybertec.at/ и проведем тестирование ещё раз.
8. Несколько итераций выполнений `pgbench` с разными настройками показали, что наилучший показатель TPS был с настройками PostgreSQL по-умолчанию. 
9. Выключим синхронный коммит `synchronous_commit = off` и повторим тестирование:
```shell
~# sudo -i -u postgres pgbench -P 5 -T 180 pgbench
pgbench (14.3 (Ubuntu 14.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 5.0 s, 2494.4 tps, lat 0.400 ms stddev 0.075
...
progress: 180.0 s, 2762.2 tps, lat 0.362 ms stddev 0.057
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 1
number of threads: 1
duration: 180 s
number of transactions actually processed: 500031
latency average = 0.360 ms
latency stddev = 0.050 ms
initial connection time = 7.007 ms
tps = 2778.054918 (without initial connection time)
```
10. Включим синхронный коммит, скопируем данные PostgreSQL на RAM диск и повторим тестирование:
```shell
~# sudo -i -u postgres pgbench -P 1 -T 180 pgbench
pgbench (14.3 (Ubuntu 14.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 1.0 s, 2168.0 tps, lat 0.458 ms stddev 0.162
...
progress: 180.0 s, 2795.0 tps, lat 0.358 ms stddev 0.021
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 1
number of threads: 1
duration: 180 s
number of transactions actually processed: 492486
latency average = 0.365 ms
latency stddev = 0.049 ms
initial connection time = 6.396 ms
tps = 2736.127319 (without initial connection time)
```
11. Протестируем с выключенным синхронным коммитом и данными PostgreSQL на RAM диске:
```shell
~# sudo -i -u postgres pgbench -P 1 -T 180 pgbench
pgbench (14.3 (Ubuntu 14.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 1.0 s, 1939.9 tps, lat 0.512 ms stddev 0.188
...
progress: 180.0 s, 2647.1 tps, lat 0.378 ms stddev 0.008
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 1
number of threads: 1
duration: 180 s
number of transactions actually processed: 465044
latency average = 0.387 ms
latency stddev = 0.037 ms
initial connection time = 6.188 ms
tps = 2583.664804 (without initial connection time)
```

Выводы: из-за небольших размеров базы данных и выполенения pgbench в один поток, 
оптимизация настроек PostgreSQL без возможной потери данных не дала улучшения значения TPS. 
Выключение синхронного комита или перемещение данных на RAM-диск, 
которые могут привети к потери данных увеличило значение TPS примерно в два раза.
Комбинация выключения синхронного комита и перемещение данных на RAM-диск не дало существенных результатов.