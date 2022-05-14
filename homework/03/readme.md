# Настройка дисков PostgreSQL

1. Создать виртуальную машину c Ubuntu 20.04 LTS.
2. Установить PostgreSQL 14:

```shell
root@server1:~# echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | tee /etc/apt/sources.list.d/pgdg.list
root@server1:~# curl https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
root@server1:~# apt update
root@server1:~# apt install -y postgresql-14
root@server1:~# systemctl status postgresql@14-main.service
● postgresql@14-main.service - PostgreSQL Cluster 14-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabl>
     Active: active (running) since Sat 2022-05-14 06:43:33 UTC; 6s ago
   Main PID: 5661 (postgres)
      Tasks: 7 (limit: 4676)
     Memory: 18.9M
     CGroup: /system.slice/system-postgresql.slice/postgresql@14-main.service
             ├─5661 /usr/lib/postgresql/14/bin/postgres -D /var/lib/postgresql/14/main -c config_fi>
             ├─5663 postgres: 14/main: checkpointer
             ├─5664 postgres: 14/main: background writer
             ├─5665 postgres: 14/main: walwriter
             ├─5666 postgres: 14/main: autovacuum launcher
             ├─5667 postgres: 14/main: stats collector
             └─5668 postgres: 14/main: logical replication launcher
```
2. Проверить, что кластер запущен:
```shell
root@server1:~# sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```
3. Создать произвольную таблицу с произвольным содержимым:
```shell
sudo -i -u postgres psql -c "CREATE TABLE test(c1 TEXT); INSERT INTO test VALUES('1');"
```
4. Остановить PostgreSQL
```shell
sudo -u postgres pg_ctlcluster 14 main stop
```
или
```shell
systemctl stop postgresql@14-main
```
5. Добавить диск к виртуальной машине:
```shell
root@server1:~# lsblk -o +label
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT        LABEL
loop0     7:0    0 61.9M  1 loop /snap/core20/1434
loop1     7:1    0 44.7M  1 loop /snap/snapd/15534
loop2     7:2    0 67.8M  1 loop /snap/lxd/22753
sda       8:0    0   50G  0 disk
├─sda1    8:1    0 49.9G  0 part /                 cloudimg-rootfs
├─sda14   8:14   0    4M  0 part
└─sda15   8:15   0  106M  0 part /boot/efi         UEFI
sdb       8:16   0   10G  0 disk
sr0      11:0    1  4.9M  0 rom                    cidata
```
6. Примонтировать диск в /mnt/data
```shell
root@server1:~# mkfs.xfs /dev/sdb
root@server1:~# xfs_admin -L pgdata /dev/sdb
root@server1:~# mkdir /mnt/data
root@server1:~# echo "/dev/sdb /mnt/data xfs defaults 0 0" >> /etc/fstab
root@server1:~# mount /mnt/data
root@server1:~# lsblk -o +label
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT        LABEL
loop0     7:0    0 61.9M  1 loop /snap/core20/1434
loop1     7:1    0 44.7M  1 loop /snap/snapd/15534
loop2     7:2    0 67.8M  1 loop /snap/lxd/22753
sda       8:0    0   50G  0 disk
├─sda1    8:1    0 49.9G  0 part /                 cloudimg-rootfs
├─sda14   8:14   0    4M  0 part
└─sda15   8:15   0  106M  0 part /boot/efi         UEFI
sdb       8:16   0   10G  0 disk /mnt/data         pgdata
sr0      11:0    1  4.9M  0 rom                    cidata
```
6. Перезагрузить инстанс:
```shell
root@server1:~# shutdown -r now
root@server1:~# uptime
 07:44:10 up 0 min,  1 user,  load average: 0.15, 0.03, 0.01
root@server1:~# lsblk -o +label
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT        LABEL
loop0     7:0    0 61.9M  1 loop /snap/core20/1434
loop1     7:1    0 44.7M  1 loop /snap/snapd/15534
loop2     7:2    0 67.8M  1 loop /snap/lxd/22753
sda       8:0    0   50G  0 disk
├─sda1    8:1    0 49.9G  0 part /                 cloudimg-rootfs
├─sda14   8:14   0    4M  0 part
└─sda15   8:15   0  106M  0 part /boot/efi         UEFI
sdb       8:16   0   10G  0 disk /mnt/data         pgdata
sr0      11:0    1  4.9M  0 rom                    cidata
```
7. Сделать пользователя postgres владельцем /mnt/data:
```shell
root@server1:~# chown -R postgres:postgres /mnt/data/
```
8. Перенести содержимое /var/lib/postgres/14 в /mnt/data:
```shell
root@server1:~#  ls -la /mnt/data
total 4
drwxr-xr-x 2 postgres postgres    6 May 14 07:31 .
drwxr-xr-x 3 root     root     4096 May 14 07:37 ..
root@server1:~#  mv /var/lib/postgresql/14 /mnt/data
root@server1:~#  ls -la /mnt/data/
total 20
drwx------ 19 postgres postgres 4096 May 14 07:47 .
drwxr-xr-x  3 postgres postgres   18 May 14 06:43 ..
drwxr-xr-x  3 postgres postgres   18 May 14 06:43 14
```
9. Запустить кластер, напишите получилось или нет и почему:
```shell
root@server1:~#  sudo -u postgres pg_ctlcluster 14 main start
Error: /var/lib/postgresql/14/main is not accessible or does not exist
```
Ответ: `pg_ctlcluster` это враппер над `pg_ctl`, 
который ищет исполняемый файл `pg_ctl` соответвующей версии переданной в аргументе, 
в данном случае "14", затем он ищет конфигурационные файлы кластера по шаблону
`/etc/postgresql/<cluster-version>/<cluster-name>/` в данном случае `/etc/postgresql/14/main/`
и формирует аргументы для `pg_ctl`.
Кластер не может запуститься, потому что параметр `data_directory` в
`/etc/postgresql/14/main/postgresql.conf` указывает на `/var/lib/postgresql/14/main`
```shell
root@server1:~# cat /etc/postgresql/14/main/postgresql.conf | grep data_directory
data_directory = '/var/lib/postgresql/14/main'
```
10. Поменять `data_directory` в `/etc/postgresql/14/main/postgresql.conf`
```shell
root@server1:~#  cat /etc/postgresql/14/main/postgresql.conf | grep data_directory
#data_directory = '/var/lib/postgresql/14/main'         # use data in another directory
data_directory = '/mnt/data/14/main'
```
11. Запустить кластер:
```shell
root@server1:~#  sudo -u postgres pg_ctlcluster 14 main start
root@server1:~#  sudo -u postgres pg_ctlcluster 14 main status
pg_ctl: server is running (PID: 2217)
/usr/lib/postgresql/14/bin/postgres "-D" "/mnt/data/14/main" "-c" "config_file=/etc/postgresql/14/main/postgresql.conf"
```
12. Проверить содержимое ранее созданной таблицы:
```shell
root@server1:~# sudo -i -u postgres psql -c "SELECT * FROM test;"
 c1
----
 1
(1 row)
```
13. Перенести диск с данными на другой инстанс.

Ответ:
- создать новый инстанс
- устнановить PostgreSQL как в п.2 
- остановить PostgreSQL 14
- удалить данные в /var/lib/postgresql/ 
- на инстансе №1: остановить PostgreSQL, отмонтировать диск, удалить запись из /etc/fstab, удалить PostgreSQL 
- подключить диск к инстансу №2 
- создать директорию /mnt/data 
- создать запись в /etc/fstab
- примонтировать диск в /mnt/data и проверить наличие файлов
- Сделать владельцем `/mnt/data` пользователя `postgres`: `chown -R postgres:postgres /mnt/data/`
- изменить `data_directory` в `/etc/postgresql/14/main/postgresql.conf`
- запустить PostgreSQL
- проверить наличие таблицы и данных:
```shell
root@server2:~# sudo -i -u postgres psql -c "SELECT * FROM test;"
 c1
----
 1
(1 row)
```
