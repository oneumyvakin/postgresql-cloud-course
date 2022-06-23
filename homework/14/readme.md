# Работа с горизонтально масштабируемым кластером

1. Установим Kubernetes и инициализируем кластер на сервере с Ubuntu 20.04:
```shell
root@kube1:~# kubectl get nodes
NAME    STATUS   ROLES           AGE   VERSION
kube1   Ready    control-plane   41m   v1.24.2
kube2   Ready    <none>          16m   v1.24.2
kube3   Ready    <none>          56s   v1.24.2
root@kube1:~#
```
2. Развернем CockroachDB c помощью Helm:
```shell
root@kube1:~# cat values.yaml
storage:
  persistentVolume:
    size: 10Gi
    
root@kube1:~# helm repo add cockroachdb https://charts.cockroachdb.com/
root@kube1:~# helm install cockroachdb cockroachdb/cockroachdb -f values.yaml
root@kube1:~# kubectl get pods
NAME                     READY   STATUS      RESTARTS   AGE
cockroachdb-0            1/1     Running     0          50m
cockroachdb-1            1/1     Running     0          50m
cockroachdb-2            1/1     Running     0          100s
cockroachdb-init-mswdn   0/1     Completed   0          50m
root@kube1:~#
```
3. Подключимся к CockroachDB c помощью нативного клиента используя безопасное подключение
```shell
root@kube1:~# cat client-secure-operator.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cockroachdb-client-secure
spec:
  serviceAccountName: cockroachdb
  containers:
  - name: cockroachdb-client-secure
    image: cockroachdb/cockroach:v22.1.1
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: client-certs
      mountPath: /cockroach/cockroach-certs/
    command:
    - sleep
    - "2147483648" # 2^31
  terminationGracePeriodSeconds: 0
  volumes:
  - name: client-certs
    projected:
        sources:
          - secret:
              name: cockroachdb-node-secret
              items:
                - key: ca.crt
                  path: ca.crt
          - secret:
              name: cockroachdb-client-secret
              items:
                - key: tls.crt
                  path: client.root.crt
                - key: tls.key
                  path: client.root.key
        defaultMode: 256
root@kube1:~# kubectl apply -f client-secure-operator.yaml
root@kube1:~# kubectl exec -it cockroachdb-client-secure -- ./cockroach sql --certs-dir=/cockroach/cockroach-certs --host=cockroachdb-public
#
# Welcome to the CockroachDB SQL shell.
# All statements must be terminated by a semicolon.
# To exit, type: \q.
#
# Client version: CockroachDB CCL v22.1.1 (x86_64-pc-linux-gnu, built 2022/06/06 16:38:56, go1.17.6)
# Server version: CockroachDB CCL v22.1.2 (x86_64-pc-linux-gnu, built 2022/06/22 15:54:12, go1.17.11)
# Cluster ID: 09e42a0c-ce0e-4713-a11d-6c499f75bad2
#
# Enter \? for a brief introduction.
#
root@cockroachdb-public:26257/defaultdb>
```
4. Создадим таблицу и загрузим в неё данные:
```shell
root@cockroachdb-public:26257/defaultdb> CREATE TABLE taxi_trips (
                                      -> unique_key text,
                                      -> taxi_id text,
                                      -> trip_start_timestamp TIMESTAMP,
                                      -> trip_end_timestamp TIMESTAMP,
                                      -> trip_seconds bigint,
                                      -> trip_miles numeric,
                                      -> pickup_census_tract bigint,
                                      -> dropoff_census_tract bigint,
                                      -> pickup_community_area bigint,
                                      -> dropoff_community_area bigint,
                                      -> fare numeric,
                                      -> tips numeric,
                                      -> tolls numeric,
                                      -> extras numeric,
                                      -> trip_total numeric,
                                      -> payment_type text,
                                      -> company text,
                                      -> pickup_latitude numeric,
                                      -> pickup_longitude numeric,
                                      -> pickup_location text,
                                      -> dropoff_latitude numeric,
                                      -> dropoff_longitude numeric,
                                      -> dropoff_location text
                                      -> );
CREATE TABLE


Time: 32ms total (execution 31ms / network 1ms)

root@cockroachdb-public:26257/defaultdb> IMPORT INTO taxi_trips (unique_key, 
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
dropoff_location
) CSV DATA ('s3://test/taxi0.csv?AWS_ACCESS_KEY_ID=x&AWS_SECRET_ACCESS_KEY=x') WITH skip = '1', nullif = '';
        job_id       |  status   | fraction_completed |  rows   | index_entries |   bytes
---------------------+-----------+--------------------+---------+---------------+-------------
  773097183396888580 | succeeded |                  1 | 7889032 |             0 | 2928164889
(1 row)


Time: 615.142s total (execution 615.117s / network 0.026s)

root@cockroachdb-public:26257/defaultdb> SELECT count(*) FROM taxi_trips;
-----------
  9889032
(1 row)


Time: 6.686s total (execution 6.685s / network 0.002s)

root@cockroachdb-public:26257/defaultdb> SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c
                                      -> FROM taxi_trips
                                      -> group by payment_type
                                      -> order by 3;
  payment_type | tips_percent |    c
---------------+--------------+----------
  Prepaid      |            0 |     345
  Dispute      |            0 |    6944
  No Charge    |            1 |   10826
  Mobile       |           16 |  257200
  Unknown      |            0 |  562774
  Prcard       |            1 |  748746
  Credit Card  |           17 | 3631819
  Cash         |            0 | 4670378
(8 rows)


Time: 14.174s total (execution 14.172s / network 0.003s)
```
5. Описать, что и как делали и с какими проблемами столкнулись:
- обеспечить безопасное подключение клиента к серверу CockroachDB;
- нативный клиент `сockroachdb` не поддерживает импорт данных напрямую в базу данных из локальных файлов, можно только загрузить файлов в хранилище базы данных, 
и только потом импортировать данные из этих файлов через схему `userfile://`, т.е. нужно 2х дискового пространства.
