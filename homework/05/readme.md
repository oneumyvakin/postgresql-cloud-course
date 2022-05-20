# Резервное копирование PostgreSQL

1. Создадим две виртуальные машины для мастера(db1) и реплики(db2) с Ubuntu 22.04 и установим на них PostgreSQL 14.
2. Создадим кластер с тестовыми данными и настроем репликацию с сервера db1 на сервер db2:
```shell
# db1
root@db1:~# systemctl stop postgresql@14-main
root@db1:~# pg_dropcluster 14 main
root@db1:~# pg_createcluster 14 main -- --data-checksums
root@db1:~# echo "listen_addresses = '10.54.2.1'" >> /etc/postgresql/14/main/postgresql.conf
root@db1:~# echo "host replication replica 10.54.0.0/21 scram-sha-256" >> /etc/postgresql/14/main/pg_hba.conf
root@db1:~# echo "wal_keep_size = '512'" >> /var/lib/postgresql/14/main/postgresql.auto.conf
root@db1:~# echo "archive_mode = always" >> /var/lib/postgresql/14/main/postgresql.auto.conf
root@db1:~# echo "archive_command = '/usr/bin/pg_probackup-14 archive-push -B /backups/ --instance=main --wal-file-path=%p --wal-file-name=%f --compress --remote-host=10.54.2.2 --remote-user=root'" >> /var/lib/postgresql/14/main/postgresql.auto.conf
root@db1:~# systemctl restart postgresql@14-main
root@db1:~# sudo -i -u postgres psql -c "CREATE USER replica WITH REPLICATION encrypted password 'replica'"
root@db1:~# sudo -i -u postgres psql -c "CREATE DATABASE pgbench;"
root@db1:~# sudo -i -u postgres pgbench -i -s 10 pgbench
```

На втором сервере для реплики:
```shell
# db2
root@db2:~# systemctl stop postgresql@14-main
root@db2:~# rm -rf /var/lib/postgresql/14/main/*
root@db2:~# export PGPASSWORD=replica
root@db2:~# sudo --preserve-env=PGPASSWORD -i -u postgres pg_basebackup --host=10.54.2.1 --port=5432 --username=replica --pgdata=/var/lib/postgresql/14/main/ --progress --write-recovery-conf --create-slot --slot=replica --checkpoint=fast
188126/188126 kB (100%), 1/1 tablespace
root@db2:~# sudo -i -u postgres touch /var/lib/postgresql/14/main/standby.signal
root@db2:~# systemctl start postgresql@14-main
root@db2:~# pg_lsclusters
Ver Cluster Port Status          Owner    Data directory              Log file
14  main    5432 online,recovery postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
root@db2:~# 
```
3. Проверим статус репликации на первом сервере с мастером:
```shell
root@db1:~# sudo -i -u postgres psql -c "select usename, application_name, client_addr, sync_state from pg_stat_replication"
 usename | application_name | client_addr | sync_state
---------+------------------+-------------+------------
 replica | 14/main          | 10.54.2.2   | async
(1 row)
```
4. Проверим что данные реплицировлись на втором сервере:
```shell
root@db2:~# sudo -i -u postgres psql pgbench -c "\d"
              List of relations
 Schema |       Name       | Type  |  Owner
--------+------------------+-------+----------
 public | pgbench_accounts | table | postgres
 public | pgbench_branches | table | postgres
 public | pgbench_history  | table | postgres
 public | pgbench_tellers  | table | postgres
(4 rows)
```
5. На мастере, создадим роль для выполнения резервного копирования, чтобы она отреплицировалась на реплику:
```shell
root@db1:~# sudo -i -u postgres psql <<EOL
BEGIN;
CREATE ROLE backup WITH LOGIN;
ALTER ROLE backup WITH REPLICATION;
GRANT USAGE ON SCHEMA pg_catalog TO backup;
GRANT SELECT ON TABLE pg_catalog.pg_database TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.current_setting(text) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.set_config(text, text, boolean) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_is_in_recovery() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_start_backup(text, boolean, boolean) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_stop_backup(boolean, boolean) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_create_restore_point(text) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_switch_wal() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_last_wal_replay_lsn() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current_snapshot() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_snapshot_xmax(txid_snapshot) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_control_checkpoint() TO backup;
COMMIT;
EOL
```
6. Настроем резервное копирование c помощью `pg_probackup` на сервере с репликой:
```shell
root@db2:~# curl https://repo.postgrespro.ru/pg_probackup/keys/GPG-KEY-PG_PROBACKUP | gpg --dearmor > /usr/share/keyrings/pgpro-keyring.gpg
root@db2:~# echo "deb [signed-by=/usr/share/keyrings/pgpro-keyring.gpg] https://repo.postgrespro.ru/pg_probackup/deb/ focal main-focal" | tee /etc/apt/sources.list.d/pgpro.list
root@db2:~# apt update
root@db2:~# export DEBIAN_FRONTEND=noninteractive
root@db2:~# apt install -y pg-probackup-14
root@db2:~# mkdir /backups
root@db2:~# chown -R postgres:postgres /backups
root@db2:~# sudo -i -u postgres pg_probackup-14 init -B /backups
root@db2:~# sudo -i -u postgres pg_probackup-14 add-instance --instance 'main' -D /var/lib/postgresql/14/main --backup-path /backups/
```

7. Выполним резервное копирование на сервере с репликой:
```shell
root@db2:~# sudo -i -u postgres pg_probackup-14 backup --instance 'main' -b FULL --stream --temp-slot --backup-path /backups/
INFO: Backup start, pg_probackup version: 2.5.5, instance: main, backup ID: RC84OO, backup mode: FULL, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
WARNING: This PostgreSQL instance was initialized without data block checksums. pg_probackup have no way to detect data block corruption without them. Reinitialize PGDATA with option '--data-checksums'.
WARNING: Current PostgreSQL role is superuser. It is not recommended to run pg_probackup under superuser.
INFO: Backup RC84OO is going to be taken from standby
INFO: wait for pg_start_backup()
INFO: Wait for WAL segment /backups/backups/main/RC84OO/database/pg_wal/000000010000000000000016 to be streamed
INFO: PGDATA size: 183MB
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 1s
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
WARNING: Could not read WAL record at 0/16032268: invalid record length at 0/16032268: wanted 24, got 0
...
INFO: Wait for LSN 0/16032268 in streamed WAL segment /backups/backups/main/RC84OO/database/pg_wal/000000010000000000000016
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 0
INFO: Validating backup RC84OO
INFO: Backup RC84OO data files are valid
INFO: Backup RC84OO resident size: 200MB
INFO: Backup RC84OO completed
```
Сообщения 
`WARNING: Could not read WAL record at 0/16032268: invalid record length at 0/16032268: wanted 24, got 0`
вызваны тем что не было получено новых данных с мастера (https://github.com/postgrespro/pg_probackup/issues/436)

8. Проверим что резервная копия была успешно создана:
```shell
root@db2:~# pg_probackup-14 show --backup-path /backups/

BACKUP INSTANCE 'main'
=====================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode  WAL Mode  TLI    Time   Data   WAL  Zratio  Start LSN   Stop LSN    Status
=====================================================================================================================================
 main      14       RC84OO  2022-05-21 08:24:26+00  FULL  STREAM    1/0  2m:41s  184MB  16MB    1.00  0/16032180  0/16032230  OK
```
9. Съемулируем непреднамеренное удаление данных на мастере:
```shell
root@db1:~# sudo -i -u postgres psql -c "DROP DATABASE pgbench;"
DROP DATABASE
```
10. Проверим, что удаление данных отреплицировалось на реплике:
```shell
root@db2:~# sudo -i -u postgres psql  -c "\l pgbench"
                       List of databases
 Name | Owner | Encoding | Collate | Ctype | Access privileges
------+-------+----------+---------+-------+-------------------
(0 rows)
```
11. Восстановим данные из резервной копии.
На мастере, установим `pg_probackup` и остановим PostgreSQL:
```shell
root@db1:~# systemctl stop postgresql@14-main
root@db1:~# rm -rf /var/lib/postgresql/14/main/* 
```

На реплике, запустим восстановление мастера из резервной копии:
```shell
root@db2:~# /usr/bin/pg_probackup-14 restore -B /backups/ --instance=main -i RC84OO --remote-host=10.54.2.1 --remote-user=root
INFO: Validating backup RC84OO
INFO: Backup RC84OO data files are valid
INFO: Backup RC84OO WAL segments are valid
INFO: Backup RC84OO is valid.
INFO: Restoring the database from backup at 2022-05-21 08:24:24+00
INFO: Start restoring backup files. PGDATA size: 199MB
INFO: Backup files are restored. Transfered bytes: 199MB, time elapsed: 2s
INFO: Restore incremental ratio (less is better): 100% (199MB/199MB)
INFO: Syncing restored files to disk
INFO: Restored backup files are synced, time elapsed: 3s
INFO: Restore of backup RC84OO completed.
root@db2:~#
```

На мастере, проверим что данные восстановились:
```shell
root@db1:~# rm -f /var/lib/postgresql/14/main/postgresql.auto.conf
root@db1:~# chown -R postgres:postgres /var/lib/postgresql
root@db1:~# systemctl start postgresql@14-main
root@db1:~# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
root@db1:~# sudo -i -u postgres psql -c "\l pgbench"
                           List of databases
  Name   |  Owner   | Encoding | Collate |  Ctype  | Access privileges
---------+----------+----------+---------+---------+-------------------
 pgbench | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
(1 row)
```
12. Так как при таком восстановлении мастера из бекапа, реплика больше не сможет принимать WAL-поток,
```shell
could not receive data from WAL stream: ERROR:  requested starting point 0/1B000000 is ahead of the WAL flush position of this server 0/1A1E8DD8
``` 
то чтобы решить эту проблему нужно заново настроить репликацию.
