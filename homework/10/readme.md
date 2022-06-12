# Работа с большим объемом реальных данных

1. Установим PostgreSQL.
2. Загрузим данные:
```shell
# du -hs taxi/
73G     taxi/
# ls -la /var/lib/postgresql/taxi | wc -l
203
# ls -lh /var/lib/postgresql/taxi/001.csv
-rw-r--r-- 1 postgres postgres 423M May 11 18:23 /var/lib/postgresql/taxi/001.csv
```
3. Получим доступ к данным файла `/var/lib/postgresql/taxi/001.csv` через Foreign Data Wrappers:
```shell
create extension file_fdw;
create server pgcsv foreign data wrapper file_fdw;

create foreign table taxi_trips_fdw_2 (
unique_key text, 
taxi_id text, 
trip_start_timestamp TIMESTAMP, 
trip_end_timestamp TIMESTAMP, 
trip_seconds bigint, 
trip_miles numeric, 
pickup_census_tract bigint, 
dropoff_census_tract bigint, 
pickup_community_area bigint, 
dropoff_community_area bigint, 
fare numeric, 
tips numeric, 
tolls numeric, 
extras numeric, 
trip_total numeric, 
payment_type text, 
company text, 
pickup_latitude numeric, 
pickup_longitude numeric, 
pickup_location text, 
dropoff_latitude numeric, 
dropoff_longitude numeric, 
dropoff_location text
)
server pgcsv
options(filename '/var/lib/postgresql/taxi/001.csv', format 'csv', header 'true', delimiter ',');

```
4. Посчитаем время выборки количества строк из созданной внешней таблицы:
```shell
postgres=# \timing
Timing is on.
postgres=# select count(*) from taxi_trips_fdw_2;
  count
---------
 1000000
(1 row)

Time: 5453.021 ms (00:05.453)
```
5. Загрузим все данные в PostgreSQL c помощью `COPY`:
```shell
postgres-# CREATE TABLE taxi_trips (
unique_key text, 
taxi_id text, 
trip_start_timestamp TIMESTAMP, 
trip_end_timestamp TIMESTAMP, 
trip_seconds bigint, 
trip_miles numeric, 
pickup_census_tract bigint, 
dropoff_census_tract bigint, 
pickup_community_area bigint, 
dropoff_community_area bigint, 
fare numeric, 
tips numeric, 
tolls numeric, 
extras numeric, 
trip_total numeric, 
payment_type text, 
company text, 
pickup_latitude numeric, 
pickup_longitude numeric, 
pickup_location text, 
dropoff_latitude numeric, 
dropoff_longitude numeric, 
dropoff_location text
);

postgres-# COPY taxi_trips(unique_key, 
taxi_id, 
trip_start_timestamp, 
trip_end_timestamp, 
trip_seconds, 
trip_miles, 
pickup_census_tract, 
dropoff_census_tract, 
pickup_community_area, 
dropoff_community_area, 
fare, 
tips, 
tolls, 
extras, 
trip_total, 
payment_type, 
company, 
pickup_latitude, 
pickup_longitude, 
pickup_location, 
dropoff_latitude, 
dropoff_longitude, 
dropoff_location)
FROM PROGRAM 'awk FNR-1 /var/lib/postgresql/taxi/*.csv | cat' DELIMITER ',' CSV HEADER;
COPY 173238753
Time: 3593402.782 ms (59:53.403)
postgres=# select count(*) from taxi_trips;
   count
-----------
 173238753
(1 row)
postgres-# \dt+
                                     List of relations
 Schema |    Name    | Type  |  Owner   | Persistence | Access method | Size  | Description
--------+------------+-------+----------+-------------+---------------+-------+-------------
 public | taxi_trips | table | postgres | permanent   | heap          | 69 GB |
(1 row)
```

6. Выполним запрос
```shell
postgres-# SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c
FROM taxi_trips
group by payment_type
order by 3;
 payment_type | tips_percent |    c
--------------+--------------+----------
 Way2ride     |           14 |      142
 Prepaid      |            0 |     1813
 Split        |           17 |     3442
 Pcard        |            2 |    30985
 Dispute      |            0 |    71449
 No Charge    |            4 |   654209
 Mobile       |           16 |   844121
 Unknown      |            1 |   915895
 Prcard       |            1 |  1039144
 Credit Card  |           17 | 71202996
 Cash         |            0 | 98474557
(11 rows)

Time: 208119.085 ms (03:28.119)
```
7. clickhouse
```shell
CREATE TABLE taxi_trips (
    unique_key              String, 
    taxi_id                 String, 
    trip_start_timestamp    DateTime, 
    trip_end_timestamp      String, 
    trip_seconds            Int64, 
    trip_miles              Decimal(6, 2), 
    pickup_census_tract     Nullable(Int64), 
    dropoff_census_tract    Nullable(Int64), 
    pickup_community_area   Nullable(Int64), 
    dropoff_community_area  Nullable(Int64), 
    fare                    Nullable(Decimal(6, 2)), 
    tips                    Nullable(Decimal(6, 2)), 
    tolls                   Nullable(Decimal(6, 2)), 
    extras                  Nullable(Decimal(6, 2)), 
    trip_total              Nullable(Decimal(6, 2)), 
    payment_type            Nullable(String), 
    company                 Nullable(String), 
    pickup_latitude         Nullable(Float64), 
    pickup_longitude        Nullable(Float64), 
    pickup_location         Nullable(String), 
    dropoff_latitude        Nullable(Float64), 
    dropoff_longitude       Nullable(Float64), 
    dropoff_location        Nullable(String)
) ENGINE = Log;
```
8. Загрузим данные в ClickHouse:
```shell
root@clickhouse:~# du -hs /root/taxi/
73G     /root/taxi/
root@clickhouse:~#
root@clickhouse:~# time awk FNR-1 /root/taxi/*.csv | clickhouse-client --password  --query="INSERT INTO taxi_trips FORMAT CSV"
Password for user (default):

real    4m52.129s
user    8m32.041s
sys     2m3.796s
root@clickhouse:~#
```
9. Размер таблицы в ClickHouse:
```shell
clickhouse :) SELECT name, total_rows, total_bytes FROM system.tables where name='taxi_trips' FORMAT Vertical;

SELECT
    name,
    total_rows,
    total_bytes
FROM system.tables
WHERE name = 'taxi_trips'
FORMAT Vertical

Query id: 5b9a077b-b9ac-4da8-ae27-0e4cee834fae

Row 1:
──────
name:        taxi_trips
total_rows:  177454534
total_bytes: 36892553626

1 row in set. Elapsed: 0.003 sec.

clickhouse :)
```
10. Выполним запрос:
```shell
clickhouse :) SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c
:-] FROM taxi_trips
:-] group by payment_type
:-] order by 3;

SELECT
    payment_type,
    round((sum(tips) / sum(trip_total)) * 100, 0) + 0 AS tips_percent,
    count(*) AS c
FROM taxi_trips
GROUP BY payment_type
ORDER BY 3 ASC

Query id: 1ac88f8b-e35f-40cc-a611-d1e0456abffb

┌─payment_type─┬─tips_percent─┬───c─┐
│ Way2ride     │           14 │ 142 │
└──────────────┴──────────────┴─────┘
┌─payment_type─┬─tips_percent─┬────c─┐
│ Split        │           17 │ 3442 │
└──────────────┴──────────────┴──────┘
┌─payment_type─┬─tips_percent─┬────────c─┐
│ Pcard        │            2 │    30985 │
│ Credit Card  │           16 │ 72659243 │
└──────────────┴──────────────┴──────────┘
┌─payment_type─┬─tips_percent─┬─────────c─┐
│ Cash         │            0 │ 100107924 │
└──────────────┴──────────────┴───────────┘
┌─payment_type─┬─tips_percent─┬───────c─┐
│ Mobile       │           15 │ 1254177 │
└──────────────┴──────────────┴─────────┘
┌─payment_type─┬─tips_percent─┬──────c─┐
│ No Charge    │            4 │ 656895 │
└──────────────┴──────────────┴────────┘
┌─payment_type─┬─tips_percent─┬────c─┐
│ Prepaid      │            0 │ 1833 │
└──────────────┴──────────────┴──────┘
┌─payment_type─┬─tips_percent─┬─────c─┐
│ Dispute      │            0 │ 74502 │
└──────────────┴──────────────┴───────┘
┌─payment_type─┬─tips_percent─┬───────c─┐
│ Unknown      │            0 │ 1207677 │
└──────────────┴──────────────┴─────────┘
┌─payment_type─┬─tips_percent─┬───────c─┐
│ Prcard       │            0 │ 1457714 │
└──────────────┴──────────────┴─────────┘

11 rows in set. Elapsed: 9.444 sec. Processed 177.45 million rows, 4.78 GB (18.79 million rows/s., 506.17 MB/s.)

clickhouse :)
```
11. С какими проблемами столкнулся:
- при импорте данных в ClickHouse, для колонок `trip_start_timestamp` и `trip_end_timestamp` пришлось указать формат `String`,
так как для импорта нужен определенный формат `YYYY-MM-TT hh:mm:ss` или unix timestamp, а в CSV файлах используется формат `"2022-05-01T00:00:00.000"`
- разное количество заимпортченных строк в PostgreSQL (173238753) и ClickHouse (177454534)
- разные результаты запроса в PostgreSQL и ClickHouse.
