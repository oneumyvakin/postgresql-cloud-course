# Установка и настройка PostgreSQL в контейнере Docker

1. Установить Docker Engine на Ubuntu 20.04.

   Docker Можно установить из стокового репозитория операционной системы: `apt install docker.io` или 
из репозиториев компании Docker, так мы сможем получить новые версии Docker Engine, containerd 
и плагина Docker Compose:
```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update
apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```
2. Создать каталог `/var/lib/postgres`:
```shell
mkdir -p /var/lib/postgres/data # сразу создадим директорию "data", чтобы быть максимально точными
ls -la /var/lib/postgres
total 12
drwxr-xr-x  3 root root 4096 May  6 15:06 .
drwxr-xr-x 44 root root 4096 May  6 15:06 ..
drwxr-xr-x  2 root root 4096 May  6 15:06 data
#
```
3. Создать контейнер с PostgreSQL 14 смонтировав в него `/var/lib/postgres`:
```shell
docker pull postgres:14.2-alpine3.15 # скачаем образ с точной версией PostgreSQL и версией OS
docker run -d \
	--name postgresql14-server \
	-e POSTGRES_USER=otus \
	-e POSTGRES_PASSWORD=otus \
	-v /var/lib/postgres/data:/var/lib/postgresql/data \ # переопредилим destination вольюма, который указан в образе
	-p 5432:5432 \ # сразу откроем на хосте порт для подключения снаружи
	postgres:14.2-alpine3.15
```
4. Создать контейнер с клиентом PostgreSQL:
```shell
docker run -d \
	--name postgresql14-client \
	-e POSTGRES_PASSWORD=otus \
	postgres:14.2-alpine3.15
```
5. Подключиться из контейнера с клиентом к контейнеру с сервером и создать таблицу с парой строк:
```shell
docker network create --driver bridge postgresql-network
docker network connect postgresql-network postgresql14-server
docker network connect postgresql-network postgresql14-client
docker exec -e PGPASSWORD=otus -ti postgresql14-client /bin/bash # сразу зададим пароль для psql
bash-5.1# psql -U otus -h postgresql14-server
otus=# CREATE TABLE test_table (id integer, name text);
CREATE TABLE
otus=# INSERT INTO test_table(id, name) VALUES (1, 'name1');
INSERT 0 1
otus=# INSERT INTO test_table(id, name) VALUES (2, 'name2');
INSERT 0 1
```
6. Подключиться к контейнеру с сервером извне инстанса:
```shell
psql -U otus -h 157.90.100.131
Password for user otus:
psql (14.2)
Type "help" for help.

otus=#
```
7. Удалить контейнер с сервером:
```shell
docker stop postgresql14-server
docker rm postgresql14-server
```
8. Создать заново контейнер с сервером:
```shell
docker run -d \
	--name postgresql14-server \
	-e POSTGRES_USER=otus \
	-e POSTGRES_PASSWORD=otus \
	-v /var/lib/postgres/data:/var/lib/postgresql/data \
	-p 5432:5432 \
	postgres:14.2-alpine3.15
docker network connect postgresql-network postgresql14-server
```
9. Подключиться снова из контейнера с клиентом к контейнеру с сервером:
```shell
otus=# SELECT * FROM test_table;
FATAL:  terminating connection due to administrator command
server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.
The connection to the server was lost. Attempting reset: Succeeded.
otus=# SELECT * FROM test_table;
 id | name
----+-------
  1 | name1
  2 | name2
(2 rows)

otus=#
```
