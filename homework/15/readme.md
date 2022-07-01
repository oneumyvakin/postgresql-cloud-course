# Работа с горизонтально масштабируемым кластером

1. Запустим один инстанс YugabyteDB c помощью Docker:
```shell
mkdir /mnt/yb_data
docker run -d --name yugabyte \
         -p7000:7000 -p9000:9000 -p5433:5433 -p9042:9042 \
         -v /mnt/yb_data:/home/yugabyte/yb_data \
         yugabytedb/yugabyte:latest bin/yugabyted start \
         --base_dir=/home/yugabyte/yb_data --daemon=false
```
2. Создадим таблицу для загрузки данных:
```shell
# docker exec -it yugabyte /home/yugabyte/bin/ysqlsh --echo-queries
ysqlsh (11.2-YB-2.13.1.0-b0)
Type "help" for help.

yugabyte=# CREATE TABLE taxi_trips (
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
yugabyte=#
```
2. Загрузим данные:
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
FROM PROGRAM 'awk FNR-1 /home/yugabyte/yb_data/taxi.csv | cat' DELIMITER ',' CSV HEADER;
COPY 9999990
Time: 133342.673 ms (20:13.343)

yugabyte=# select count(*) from taxi_trips;
select count(*) from taxi_trips;
  count
---------
 9999990
(1 row)

Time: 104983.986 ms (01:44.984)
yugabyte=#
```
3. Выполним запрос:
```shell
yugabyte-# SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c
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

Time: 354763.430 ms (05:54.763)
yugabyte=#
```
4. Возьмем Kubernetes кластер версии 1.21.x из трёх нод:
```shell
kubectl get nodes
NAME    STATUS   ROLES    AGE    VERSION
kube1   Ready    <none>   89m    v1.21.14
kube2   Ready    <none>   110m   v1.21.14
kube3   Ready    <none>   101m   v1.21.14
```
5. Создадим PersistentVolumes:

<details>
  <summary>PersistentVolumes</summary>

```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: datadir0-yb-master-0
  namespace: yb-demo
spec:
  claimRef:
    name: datadir0-yb-master-0
    namespace: yb-demo
  storageClassName: standard
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/datadir0-yb-master-0"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - kube1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: datadir0-yb-tserver-0
  namespace: yb-demo
spec:
  claimRef:
    name: datadir0-yb-tserver-0
    namespace: yb-demo
  storageClassName: standard
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/datadir0-yb-tserver-0"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - kube1

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: datadir0-yb-master-1
  namespace: yb-demo
spec:
  claimRef:
    name: datadir0-yb-master-1
    namespace: yb-demo
  storageClassName: standard
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/datadir0-yb-master-1"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - kube2
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: datadir0-yb-tserver-1
  namespace: yb-demo
spec:
  claimRef:
    name: datadir0-yb-tserver-1
    namespace: yb-demo
  storageClassName: standard
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/datadir0-yb-tserver-1"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - kube2

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: datadir0-yb-master-2
  namespace: yb-demo
spec:
  claimRef:
    name: datadir0-yb-master-2
    namespace: yb-demo
  storageClassName: standard
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/datadir0-yb-master-2"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - kube3
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: datadir0-yb-tserver-2
  namespace: yb-demo
spec:
  claimRef:
    name: datadir0-yb-tserver-2
    namespace: yb-demo
  storageClassName: standard
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/datadir0-yb-tserver-2"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - kube3
---

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: datadir1-yb-master-0
  namespace: yb-demo
spec:
  claimRef:
    name: datadir1-yb-master-0
    namespace: yb-demo
  storageClassName: standard
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/datadir1-yb-master-0"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - kube1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: datadir1-yb-tserver-0
  namespace: yb-demo
spec:
  claimRef:
    name: datadir1-yb-tserver-0
    namespace: yb-demo
  storageClassName: standard
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/datadir1-yb-tserver-0"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - kube1

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: datadir1-yb-master-1
  namespace: yb-demo
spec:
  claimRef:
    name: datadir1-yb-master-1
    namespace: yb-demo
  storageClassName: standard
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/datadir1-yb-master-1"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - kube2
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: datadir1-yb-tserver-1
  namespace: yb-demo
spec:
  claimRef:
    name: datadir1-yb-tserver-1
    namespace: yb-demo
  storageClassName: standard
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/datadir1-yb-tserver-1"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - kube2

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: datadir1-yb-master-2
  namespace: yb-demo
spec:
  claimRef:
    name: datadir1-yb-master-2
    namespace: yb-demo
  storageClassName: standard
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/datadir1-yb-master-2"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - kube3
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: datadir1-yb-tserver-2
  namespace: yb-demo
spec:
  claimRef:
    name: datadir1-yb-tserver-2
    namespace: yb-demo
  storageClassName: standard
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/datadir1-yb-tserver-2"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - kube3
---
```
</details>

6. Установим кластер YugabyteDB следуя документации пользователя https://docs.yugabyte.com/preview/deploy/kubernetes/single-zone/oss/helm-chart/
7. Проверим, что поды запустились и работают:
```shell
kubectl get pods -n yb-demo
NAME           READY   STATUS    RESTARTS   AGE
yb-master-0    2/2     Running   2          92m
yb-master-1    2/2     Running   2          92m
yb-master-2    2/2     Running   2          92m
yb-tserver-0   2/2     Running   0          92m
yb-tserver-1   2/2     Running   0          92m
yb-tserver-2   2/2     Running   0          92m
```
5. Создадим шардированную таблицу:
```shell
root@kube2:~# kubectl exec --namespace yb-demo -it yb-tserver-0 -- /home/yugabyte/bin/ysqlsh -h yb-tserver-0.yb-tservers.yb-demo
Defaulted container "yb-tserver" out of: yb-tserver, yb-cleanup
ysqlsh (11.2-YB-2.15.0.0-b0)
Type "help" for help.

yugabyte=# CREATE TABLE taxi_trips (
id int GENERATED ALWAYS AS IDENTITY,
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
dropoff_location text,
PRIMARY KEY (id HASH)
);
CREATE TABLE
yugabyte=#
```
6. Загрузим данные:
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
FROM PROGRAM 'awk FNR-1 /mnt/disk0/taxi.csv | cat' DELIMITER ',' CSV HEADER;
COPY 9999990
Time: 4253591.109 ms (70:53.591)

yugabyte=# select count(*) from taxi_trips;
  count
---------
 9999990
(1 row)

Time: 38348.313 ms (00:38.348)
```
7. Выполним запрос:
```shell
yugabyte=# SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c
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

Time: 526301.271 ms (08:46.301)
```
8. Описать, что и как делали и с какими проблемами столкнулись:
- для разворачивания YugabyteDB требуется определенная версия Kubernetes, т.к. Helm чарты не поддерживают новую версию
