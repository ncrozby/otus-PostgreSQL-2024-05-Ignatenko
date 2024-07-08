# Блокировки. 
## Механизм блокировок

Используем ВМ Oracle Virtual Box с Ubuntu (24.04 LTS (Noble Numbat))
PostgreSQL 16.2

### Создать БД для тестов: ## Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

Создаём БД и устанавливаем параметры
```sql
postgres=# create database les10;
CREATE DATABASE

les10=# SHOW lock_timeout;
 lock_timeout
--------------
 0
(1 row)

les10=# show deadlock_timeout;
 deadlock_timeout
------------------
 1s
(1 row)

les10=# alter system set deadlock_timeout = 200;
ALTER SYSTEM

les10=# ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM
```

пергружаем кластер. И продолжаем:

```sql
les10=# show log_lock_waits;
 log_lock_waits
----------------
 on
(1 row)

les10=# show deadlock_timeout;
 deadlock_timeout
------------------
 200ms
(1 row)
```

Создаём таблицу для издевательств:

```sql
les10=# create table cats(id_d serial primary key,
                         name_cats varchar(50),
                         date_born date);
CREATE TABLE
les10=# insert into cats(name_cats, date_born)
values ('Мурзик','12/20/2023'),
       ('Ваха','12/05/2023'),
       ('Бегемот','12/10/2023');
INSERT 0 3
```

Открываем 3 сессии и опрделяем PID процессов:
```sql
-- Сессия 1
les10=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
           3952
(1 row)
```

```sql
-- Сессия 2

les10=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
           3964
(1 row)
```

```sql
-- Сессия 3

les10=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
           4006
(1 row)
```

```sql
-- Сессия 1

les10=# begin;
BEGIN
les10=*# update cats set date_born = '12/12/2023' where id_d = 1 ;
UPDATE 1
```

```sql
-- Сессия 2

les10=# begin;
BEGIN
les10=*# update cats set date_born = '12/02/2023' where id_d = 1 ;
```

Смотрим журнал (/var/log/postgresql/postgresql-16-main.log), и находим что нужно:
```log
2024-07-07 18:53:42.176 MSK [3964] postgres@les10 CONTEXT:  while updating tuple (0,1) in relation "cats"
2024-07-07 18:53:42.176 MSK [3964] postgres@les10 STATEMENT:  update cats set date_born = '12/02/2023' where id_d = 1 ;
2024-07-07 18:54:25.455 MSK [3964] postgres@les10 LOG:  process 3964 still waiting for ShareLock on transaction 7435073 after 202.282 ms
```

## Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

```sql
--Сессия 1 (3952)
les10=# begin;
BEGIN
les10=*# update cats set date_born = '12/01/2023' where id_d = 1 ;
UPDATE 1
```

```sql
-- Сессия 2 (3964)
les10=# begin;
BEGIN
les10=*# update cats set date_born = '12/02/2023' where id_d = 1 ;
```

```sql
--Сессия 3 (4006)
les10=# begin;
BEGIN
les10=*# update cats set date_born = '12/03/2023' where id_d = 1 ;
```

```sql
--Сессия 1 (3952)
les10=*# SELECT ROW_NUMBER() OVER (ORDER BY pid) as NN, pid, locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks ;
 nn | pid  |   locktype    | relation  | virtxid |   xid   |       mode       | granted
----+------+---------------+-----------+---------+---------+------------------+---------
 1  | 3952 | virtualxid    |           | 4/7     |         | ExclusiveLock    | t
 2  | 3952 | transactionid |           |         | 7435078 | ExclusiveLock    | t
 3  | 3952 | relation      | pg_locks  |         |         | AccessShareLock  | t
 4  | 3952 | relation      | cats_pkey |         |         | RowExclusiveLock | t
 5  | 3952 | relation      | cats      |         |         | RowExclusiveLock | t
 6  | 3964 | virtualxid    |           | 5/8     |         | ExclusiveLock    | t
 7  | 3964 | transactionid |           |         | 7435079 | ExclusiveLock    | t
 8  | 3964 | relation      | cats_pkey |         |         | RowExclusiveLock | t
 9  | 3964 | relation      | cats      |         |         | RowExclusiveLock | t
 10 | 3964 | transactionid |           |         | 7435078 | ShareLock        | f
 11 | 3964 | tuple         | cats      |         |         | ExclusiveLock    | t
 12 | 4006 | virtualxid    |           | 6/3     |         | ExclusiveLock    | t
 13 | 4006 | transactionid |           |         | 7435080 | ExclusiveLock    | t
 14 | 4006 | relation      | cats_pkey |         |         | RowExclusiveLock | t
 15 | 4006 | relation      | cats      |         |         | RowExclusiveLock | t
 16 | 4006 | tuple         | cats      |         |         | ExclusiveLock    | f
(16 rows)
```
Строка 3 блокировка последнего запроса select from pg_locks

Строки 1,6,12 с locktype = virtualxid блокируют виртуальный номер собственной транзакции.

Строки 2, 7 и 13 с locktype = transactionid блокируют реальный номер собственной текущей транзакции.

Строки 3,4,8,9,14,15 c locktype = relation (cats и cats_pkey) блокировку выставили update from cats в соосветвующем процессе. RowExclusiveLock отправляет за дальнейшими блокировками в таблицу cats в поле xmax

Самое интересное:

Процесс 2, выполняет Update, выставляет блокировку tuple cats (срока 11), и пытается заблокировать строчку в таблице, но она уже занята процессом 1, об этом будет запись в поле xmax. Он пытается прверить свободен ли номер, но у него не получается. Об этом строка 10, locktype = transactionid, тип блокировки ShareLock и она не удачная (granted=f). Процесс уходит в ожидание. Далее процесс 3 хочет прочитать строку, но не может она уже заблокирована Процессом 2 и он уходит ждать (строка 16) 

## Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

```sql
--Сессия 1 (3952)
les10=# begin;
BEGIN
les10=*# update cats set date_born = '12/01/2023' where id_d = 1 ;
UPDATE 1
les10=*# update cats set date_born = '12/12/2023' where id_d = 2 ;
```
```sql
--Сессия 2 (3964)
les10=# begin;
BEGIN
les10=*# update cats set date_born = '12/02/2023' where id_d = 2 ;
UPDATE 1
les10=*# update cats set date_born = '12/13/2023' where id_d = 3 ;
```

```sql
--Сессия 3 (4006)
les10=# begin;
BEGIN
les10=*# update cats set date_born = '12/03/2023' where id_d = 3 ;
UPDATE 1
les10=*# update cats set date_born = '12/11/2023' where id_d = 1 ;
ERROR:  deadlock detected
DETAIL:  Process 4006 waits for ShareLock on transaction 7435084; blocked by process 3952.
Process 3952 waits for ShareLock on transaction 7435085; blocked by process 3964.
Process 3964 waits for ShareLock on transaction 7435086; blocked by process 4006.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "cats"
```

Смотрим журнал (/var/log/postgresql/postgresql-16-main.log):
```log
2024-07-07 19:24:20.919 MSK [4006] postgres@les10 LOG:  process 4006 detected deadlock while waiting for ShareLock on transaction 7435084 after 204.478 ms
2024-07-07 19:24:20.919 MSK [4006] postgres@les10 DETAIL:  Process holding the lock: 3952. Wait queue: .
2024-07-07 19:24:20.919 MSK [4006] postgres@les10 CONTEXT:  while updating tuple (0,1) in relation "cats"
2024-07-07 19:24:20.919 MSK [4006] postgres@les10 STATEMENT:  update cats set date_born = '12/11/2023' where id_d = 1 ;
2024-07-07 19:24:20.919 MSK [4006] postgres@les10 ERROR:  deadlock detected
2024-07-07 19:24:20.919 MSK [4006] postgres@les10 DETAIL:  Process 4006 waits for ShareLock on transaction 7435084; blocked by process 3952.
        Process 3952 waits for ShareLock on transaction 7435085; blocked by process 3964.
        Process 3964 waits for ShareLock on transaction 7435086; blocked by process 4006.
        Process 4006: update cats set date_born = '12/11/2023' where id_d = 1 ;
        Process 3952: update cats set date_born = '12/12/2023' where id_d = 2 ;
        Process 3964: update cats set date_born = '12/13/2023' where id_d = 3 ;
2024-07-07 19:24:20.919 MSK [4006] postgres@les10 HINT:  See server log for query details.
2024-07-07 19:24:20.919 MSK [4006] postgres@les10 CONTEXT:  while updating tuple (0,1) in relation "cats"
2024-07-07 19:24:20.919 MSK [4006] postgres@les10 STATEMENT:  update cats set date_born = '12/11/2023' where id_d = 1 ;
2024-07-07 19:24:20.920 MSK [3964] postgres@les10 LOG:  process 3964 acquired ShareLock on transaction 7435086 after 8444.585 ms
2024-07-07 19:24:20.920 MSK [3964] postgres@les10 CONTEXT:  while updating tuple (0,3) in relation "cats"
2024-07-07 19:24:20.920 MSK [3964] postgres@les10 STATEMENT:  update cats set date_born = '12/13/2023' where id_d = 3 ;
```
В журнале видно: кто упал, на каком запросе, кто мешал, кто мешал тому кто мешал, номера процессов и транзакций. Т.е. вполне достаточно для раследования.

## Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

Для запроса _update cats set date_born = '12/03/2023'_, перебор будет идти по PK, следовательно порядок в обоих сессиях одинаков, и заблокировать друг друга не смогут. Первый будет обновлять без проблем, второй застрянет на первой же строке.
