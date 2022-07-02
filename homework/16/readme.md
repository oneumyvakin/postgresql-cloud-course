# Работа с Citus в Kubernetes

1. Развернем Citus на трех нодах в Kubernetes кластере с помощью [манифеста](manifest/citus).
2. Подключимся к мастеру:
```shell
root@kube2:~# kubectl exec -it pod/citus-master-6cb9c775c4-blhw6 -- psql --username=postgres
psql (12.8 (Debian 12.8-1.pgdg100+1))
Type "help" for help.

postgres=# SELECT * FROM master_get_active_worker_nodes();
          node_name           | node_port
------------------------------+-----------
 citus-worker-2.citus-workers |      5432
 citus-worker-0.citus-workers |      5432
 citus-worker-1.citus-workers |      5432
(3 rows)

postgres=#
```
3. Создадим таблицу для загрузки данных и последующего шардирования:
```shell
postgres=# CREATE TABLE taxi_trips (
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
CREATE TABLE
postgres=#
```
4. Загрузим данные:
```shell
COPY taxi_trips(unique_key, 
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
FROM PROGRAM 'awk FNR-1 /var/lib/postgresql/data/pgdata/taxi.csv | cat' DELIMITER ',' CSV HEADER;
COPY 9999990
Time: 133342.673 ms (01:13.343)

postgres=# select count(*) from taxi_trips;
---------
 9999990
(1 row)

Time: 1028.647 ms (00:01.029)
postgres=#
```
5. Шардируем таблицу:
```shell
postgres=# SELECT * FROM pg_extension;
  oid  | extname | extowner | extnamespace | extrelocatable | extversion | extconfig | extcondition
-------+---------+----------+--------------+----------------+------------+-----------+--------------
 13394 | plpgsql |       10 |           11 | f              | 1.0        |           |
 16384 | citus   |       10 |           11 | f              | 10.1-1     |           |
(2 rows)

Time: 2.151 ms
postgres=# 
postgres=# CREATE EXTENSION pgcrypto;
CREATE EXTENSION
Time: 39.666 ms
postgres=# ALTER TABLE taxi_trips ADD COLUMN id UUID PRIMARY KEY DEFAULT gen_random_uuid();
ALTER TABLE
Time: 76484.586 ms (01:16.485)
postgres=#
postgres=# \dt+
                       List of relations
 Schema |    Name    | Type  |  Owner   |  Size   | Description
--------+------------+-------+----------+---------+-------------
 public | taxi_trips | table | postgres | 4354 MB |
(1 row)

postgres=#
postgres=# SELECT create_distributed_table('taxi_trips', 'id');
NOTICE:  Copying data from local table...
NOTICE:  copying the data has completed
DETAIL:  The local data in the table is no longer visible, but is still on disk.
HINT:  To remove the local data, run: SELECT truncate_local_data_after_distributing_table($$public.taxi_trips$$)
 create_distributed_table
--------------------------

(1 row)

Time: 62024.290 ms (01:02.024)
postgres=#
```
6. Проверим что таблица отшардировалась на воркерах:
```shell
postgres=# \dt+
                          List of relations
 Schema |       Name        | Type  |  Owner   |  Size  | Description
--------+-------------------+-------+----------+--------+-------------
 public | taxi_trips_102040 | table | postgres | 136 MB |
 public | taxi_trips_102043 | table | postgres | 136 MB |
 public | taxi_trips_102046 | table | postgres | 136 MB |
 public | taxi_trips_102049 | table | postgres | 136 MB |
 public | taxi_trips_102052 | table | postgres | 136 MB |
 public | taxi_trips_102055 | table | postgres | 136 MB |
 public | taxi_trips_102058 | table | postgres | 136 MB |
 public | taxi_trips_102061 | table | postgres | 136 MB |
 public | taxi_trips_102064 | table | postgres | 136 MB |
 public | taxi_trips_102067 | table | postgres | 136 MB |
 public | taxi_trips_102070 | table | postgres | 136 MB |
(11 rows)

postgres=#
```
7. Выполним запрос:
```shell
postgres=# SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c
FROM taxi_trips
group by payment_type
order by 3;
 payment_type | tips_percent |    c
--------------+--------------+---------
 Prepaid      |            0 |     610
 Dispute      |            0 |    5930
 No Charge    |            2 |   13590
 Unknown      |            0 |  120500
 Prcard       |            1 |  159730
 Mobile       |           15 |  200520
 Credit Card  |           17 | 4683030
 Cash         |            0 | 4816080
(8 rows)

Time: 718.172 ms
postgres=#
```
