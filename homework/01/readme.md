# Работа с уровнями изоляции транзакции в PostgreSQL

1. Выключить auto commit
```shell
# \echo :AUTOCOMMIT
on
# \set AUTOCOMMIT off
# \echo :AUTOCOMMIT
off
```
2. Сделать в первой сессии новую таблицу и наполнить ее данными
```shell
CREATE table persons(id serial, first_name text, second_name text); 

INSERT INTO persons(first_name, second_name) VALUES('ivan', 'ivanov'); 

INSERT INTO persons(first_name, second_name) VALUES('petr', 'petrov');

COMMIT;
```
3. Посмотреть текущий уровень изоляции транзакций
```shell
SHOW TRANSACTION ISOLATION LEVEL;
 transaction_isolation
-----------------------
 read committed
(1 row)
```
4. Начать новую транзакцию в обоих сессиях с дефолтным уровнем изоляции
```shell
BEGIN; -- or START TRANSACTION
```
5. В первой сессии добавить новую запись
```shell
INSERT INTO persons(first_name, second_name) VALUES('sergey', 'sergeev');
```
6. Во второй сессии выполнить
```shell
SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
7. Видите ли вы новую запись и если да то почему?

   Ответ: Нет, новой записи нет, потому что:
   - транзакция в первой сессии не закомичена
   - уровень изоляции транзакции `READ COMMITTED` не позволяет видеть данные изменённые не закомиченными транзакциями
8. Завершить первую транзакцию 
```shell
COMMIT;
```
9. Во второй сессии выполнить
```shell
SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
10. Видите ли вы новую запись и если да то почему?

    Ответ: Да, новая запись появилась, потому что транзакция, в которой она была добавлена, была закомичена.
11. Начать новые `REPEATABLE READ` транзакции
```shell
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```
12. В первой сессии добавить новую запись 
```shell
INSERT INTO persons(first_name, second_name) VALUES('sveta', 'svetova');
```
13. Во второй сессии выполнить
```shell
SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
14. Видите ли вы новую запись и если да то почему?

    Ответ: Нет, новой записи нет, потому что:
    - транзакция в первой сессии не закомичена
    - уровень изоляции транзакции `REPEATABLE READ` не позволяет видеть данные изменённые не закомиченными транзакциями.
15. Завершить первую транзакцию
```shell
COMMIT;
```
16. Во второй сессии выполнить
```shell
SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
17. Видите ли вы новую запись и если да то почему?

    Ответ: Нет, новой записи нет, потому что:
    - на шаге 13 мы прочитали данные в открытой транзакции с уровнем изоляции `REPEATABLE READ`
    - не смотря на то, что первая транзакция была закомиченна, 
уровень изоляции транзакции `REPEATABLE READ` не позволяет "видеть" новые изменения в данных,
которые были сделаны после первого запроса в текущей транзакции.
19. Завершить вторую транзакцию
```shell
COMMIT;
```
19. Во второй сессии выполнить
```shell
SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```
20. Видите ли вы новую запись и если да, то почему?

    Ответ: Да, новая запись появилась, потому что:
    - во второй сессии мы закомитили текущую транзакцию, вышли из режима открытой транзакции и теперь можем видеть данные,
которые были закомичены после первого запроса в открытой транзакции.
