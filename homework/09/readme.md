# PostgreSQL в Kubernetes

1. Установим Kubernetes и инициализируем кластер на сервере с Ubuntu 20.04:
```shell
modprobe br_netfilter

echo 'br_netfilter' > /etc/modules-load.d/k8s.conf
echo 'net.bridge.bridge-nf-call-ip6tables = 1' >  /etc/sysctl.d/k8s.conf
echo 'net.bridge.bridge-nf-call-iptables = 1'  >> /etc/sysctl.d/k8s.conf
echo 'net.ipv4.ip_forward = 1'                 >> /etc/sysctl.d/k8s.conf
sysctl --system

curl https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor > /usr/share/keyrings/download-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/download-archive-keyring.gpg] https://download.docker.com/linux/ubuntu focal stable" | tee /etc/apt/sources.list.d/docker.list
apt-get update
apt install containerd.io
ls /run/containerd/containerd.sock

sed -i 's/"cri"//g' /etc/containerd/config.toml # disabled_plugins = ["cri"] -> disabled_plugins = []
cat /etc/containerd/config.toml
systemctl restart containerd.service

apt install -y apt-transport-https ca-certificates curl && \
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg && \
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list && \
apt update && \
apt install -y kubelet kubeadm kubectl kubernetes-cni && \
apt-mark hold kubelet kubeadm kubectl kubernetes-cni

kubeadm init 

export KUBECONFIG=/etc/kubernetes/admin.conf

kubectl taint node node1 node-role.kubernetes.io/master:NoSchedule-
kubectl taint node node1 node-role.kubernetes.io/master-
kubectl taint node node1 node-role.kubernetes.io/control-plane-
```
2. Установим Cilium в качестве CNI:
```shell
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor > /usr/share/keyrings/baltocdn-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/baltocdn-archive-keyring.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
apt update
apt install helm

helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.11.5 --namespace kube-system
```
3. Проверим, что системные поды успешно запустились:
```shell
root@db1:~# kubectl get pods --all-namespaces
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   cilium-444jj                       1/1     Running   0          19m
kube-system   cilium-operator-7bf4fdcbb4-q94md   1/1     Running   0          19m
kube-system   cilium-operator-7bf4fdcbb4-tw8s7   0/1     Pending   0          19m
kube-system   coredns-6d4b75cb6d-4bw4p           1/1     Running   0          20m
kube-system   coredns-6d4b75cb6d-qvgk4           1/1     Running   0          20m
kube-system   etcd-db1                           1/1     Running   0          20m
kube-system   kube-apiserver-db1                 1/1     Running   0          20m
kube-system   kube-controller-manager-db1        1/1     Running   0          20m
kube-system   kube-proxy-nnl7q                   1/1     Running   0          20m
kube-system   kube-scheduler-db1                 1/1     Running   0          20m
root@db1:~#
```
4. Создадим директорию для данных PostgreSQL:
```shell
mkdir -p /mnt/data/pv-postgres-1
```
5. Создадим манифест для PostgreSQL:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  type: NodePort
  ports:
   - port: 5432
  selector:
    app: postgres
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pg1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  local:
    path: "/mnt/data/pv-postgres-1"
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - db1
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: pg1
spec:
  serviceName: "postgres"
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14.3-alpine3.16
        ports:
        - containerPort: 5432
        env:
          - name: POSTGRES_DB
            value: myapp
          - name: POSTGRES_USER
            value: myuser
          - name: POSTGRES_PASSWORD
            value: passwd
        volumeMounts:
        - name: postgredb
          mountPath: /var/lib/postgresql/data
          subPath: postgres
  volumeClaimTemplates:
  - metadata:
      name: postgredb
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 1Gi
```
6. Применим манифест: `kubectl apply -f pg1.yaml`
7. Проверим, что все ресурсы манифеста создались успешно:
```shell
root@db1:~# kubectl get statefulset
NAME   READY   AGE
pg1    1/1     7m15s
root@db1:~# kubectl get pv
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS   REASON   AGE
pg1    1Gi        RWO            Retain           Bound    default/postgredb-pg1-0   standard                5m46s
root@db1:~# kubectl get pvc
NAME              STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
postgredb-pg1-0   Bound    pg1      1Gi        RWO            standard       7m28s
root@db1:~# kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
pg1-0   1/1     Running   0          7m42s
root@db1:~# kubectl get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          7h18m
postgres     NodePort    10.109.210.11   <none>        5432:32361/TCP   9m48s
```
8. Теперь мы можем подключиться к PostgreSQL по IP адресу ноды:
```shell
root@db1:~# PGPASSWORD=passwd psql -U myuser -h 10.54.2.1 -p 32361 myapp
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1), server 14.3)
WARNING: psql major version 12, server major version 14.
         Some psql features might not work.
Type "help" for help.

myapp=# CREATE TABLE test (id int);
CREATE TABLE
myapp=# \d+
                   List of relations
 Schema | Name | Type  | Owner  |  Size   | Description
--------+------+-------+--------+---------+-------------
 public | test | table | myuser | 0 bytes |
(1 row)
```
9. Проверим, что данные сохраняются между рестартами пода:
```shell
root@db1:~# kubectl get pod
NAME    READY   STATUS    RESTARTS      AGE
pg1-0   1/1     Running   1 (49m ago)   86m
root@db1:~#
root@db1:~# kubectl delete pod pg1-0
pod "pg1-0" deleted
root@db1:~# kubectl get pod
NAME    READY   STATUS    RESTARTS   AGE
pg1-0   1/1     Running   0          29s
root@db1:~# PGPASSWORD=passwd psql -U myuser -h 10.54.2.1 -p 30055 myapp
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1), server 14.3)
WARNING: psql major version 12, server major version 14.
         Some psql features might not work.
Type "help" for help.

myapp=# \d+
                   List of relations
 Schema | Name | Type  | Owner  |  Size   | Description
--------+------+-------+--------+---------+-------------
 public | test | table | myuser | 0 bytes |
(1 row)

myapp=#
```
10. Создадим новый Helm chart командой: `helm create custom-postgresql`
11. Модифицируем файлы шаблонов и `values.yaml` таким образом, чтобы можно было модифицировать имя базы данных, имя пользователя, пароль и образ PostgreSQL. (см. [файлы чарта](custom-postgresql))
12. Выполним установку чарта в режиме `--dry-run`
```shell
root@db1:~# helm install pg2  custom-postgresql --dry-run
NAME: pg2
LAST DEPLOYED: Fri Jun  3 19:11:56 2022
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
HOOKS:
MANIFEST:
---
# Source: custom-postgresql/templates/postgresql.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-pg2-custom-postgresql
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  local:
    path: "/mnt/data/pv-pg2-custom-postgresql"
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - db1
---
# Source: custom-postgresql/templates/service.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-pg2-custom-postgresql
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  local:
    path: "/mnt/data/pv-pg2-custom-postgresql"
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - db1
---
# Source: custom-postgresql/templates/postgresql.yaml
apiVersion: v1
kind: Service
metadata:
  name: pg2-custom-postgresql
spec:
  type: NodePort
  ports:
    - port: 5432
      protocol: TCP
  selector:
    app: pg2-custom-postgresql
---
# Source: custom-postgresql/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: pg2-custom-postgresql
spec:
  type: NodePort
  ports:
    - port: 5432
      protocol: TCP
  selector:
    app: pg2-custom-postgresql
---
# Source: custom-postgresql/templates/postgresql.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: pg2-custom-postgresql
spec:
  serviceName: pg2-custom-postgresql
  replicas: 1
  selector:
    matchLabels:
      app: pg2-custom-postgresql
  template:
    metadata:
      labels:
        app: pg2-custom-postgresql
    spec:
      containers:
        - name: postgres
          image: postgres:14.3-alpine3.16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: myapp
            - name: POSTGRES_USER
              value: myuser
            - name: POSTGRES_PASSWORD
              value: passwd
          volumeMounts:
            - name: postgredb
              mountPath: /var/lib/postgresql/data
              subPath: postgres
  volumeClaimTemplates:
    - metadata:
        name: postgredb
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: standard
        resources:
          requests:
            storage: 1Gi
---
# Source: custom-postgresql/templates/service.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: pg2-custom-postgresql
spec:
  serviceName: pg2-custom-postgresql
  replicas: 1
  selector:
    matchLabels:
      app: pg2-custom-postgresql
  template:
    metadata:
      labels:
        app: pg2-custom-postgresql
    spec:
      containers:
        - name: postgres
          image: postgres:14.3-alpine3.16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: myapp
            - name: POSTGRES_USER
              value: myuser
            - name: POSTGRES_PASSWORD
              value: passwd
          volumeMounts:
            - name: postgredb
              mountPath: /var/lib/postgresql/data
              subPath: postgres
  volumeClaimTemplates:
    - metadata:
        name: postgredb
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: standard
        resources:
          requests:
            storage: 1Gi

NOTES:
1. Create directory for PostgreSQL data: mkdir -p /mnt/data/pv-pg2-custom-postgresql
2. Use following commands to connect to database:
export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services pg2-custom-postgresql)
export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
PGPASSWORD=passwd psql -U myuser -h $NODE_IP -p $NODE_PORT myapp
```
13. Установим чарт:
```yaml
root@db1:~# helm install pg2 custom-postgresql
NAME: pg2
LAST DEPLOYED: Fri Jun  3 19:38:42 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Create directory for PostgreSQL data: mkdir -p /mnt/data/pv-pg2-custom-postgresql
2. Use following commands to connect to database:
export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services pg2-custom-postgresql)
export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
PGPASSWORD=passwd psql -U myuser -h $NODE_IP -p $NODE_PORT myapp
root@db1:~# export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services pg2-custom-postgresql)
root@db1:~# export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
root@db1:~# PGPASSWORD=passwd psql -U myuser -h $NODE_IP -p $NODE_PORT myapp
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1), server 14.3)
WARNING: psql major version 12, server major version 14.
         Some psql features might not work.
Type "help" for help.

myapp=# exit
root@db1:~#
```
