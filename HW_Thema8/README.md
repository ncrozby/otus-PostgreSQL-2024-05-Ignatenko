# MVCC, vacuum и autovacuum. 
## Настройка autovacuum с учетом особеностей производительности

Используем ВМ Oracle Virtual Box с Ubuntu (24.04 LTS (Noble Numbat))
PostgreSQL 16.2

### Создать БД для тестов: 
```bash
postgres@ubunt:~$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.06 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.37 s (drop tables 0.01 s, create tables 0.03 s, client-side generate 0.21 s, vacuum 0.06 s, primary keys 0.06 s).
```

### Запустить pgbench первый раз
```bash
postgres@ubunt:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (16.3 (Ubuntu 16.3-0ubuntu0.24.04.1))
starting vacuum...end.
progress: 6.0 s, 790.9 tps, lat 10.053 ms stddev 5.753, 0 failed
progress: 12.0 s, 798.5 tps, lat 10.017 ms stddev 5.504, 0 failed
progress: 18.0 s, 798.0 tps, lat 10.020 ms stddev 5.733, 0 failed
progress: 24.0 s, 784.7 tps, lat 10.198 ms stddev 6.687, 0 failed
progress: 30.0 s, 800.1 tps, lat 9.992 ms stddev 5.493, 0 failed
progress: 36.0 s, 796.3 tps, lat 10.047 ms stddev 5.576, 0 failed
progress: 42.0 s, 798.0 tps, lat 10.020 ms stddev 5.785, 0 failed
progress: 48.0 s, 793.8 tps, lat 10.076 ms stddev 5.483, 0 failed
progress: 54.0 s, 744.8 tps, lat 10.737 ms stddev 6.536, 0 failed
progress: 60.0 s, 699.0 tps, lat 11.439 ms stddev 7.040, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 46834
number of failed transactions: 0 (0.000%)
latency average = 10.243 ms
latency stddev = 5.980 ms
initial connection time = 28.403 ms
tps = 780.708105 (without initial connection time)
```

### Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла

Параметры поменяли в файле _postgresql.conf_

```
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
```
Перегрузили систему

### Второй тест pgbench
```bash
postgres@ubunt:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (16.3 (Ubuntu 16.3-0ubuntu0.24.04.1))
starting vacuum...end.
progress: 6.0 s, 792.2 tps, lat 10.036 ms stddev 5.471, 0 failed
progress: 12.0 s, 781.3 tps, lat 10.238 ms stddev 5.925, 0 failed
progress: 18.0 s, 785.5 tps, lat 10.176 ms stddev 5.868, 0 failed
progress: 24.0 s, 738.3 tps, lat 10.838 ms stddev 6.444, 0 failed
progress: 30.0 s, 777.8 tps, lat 10.283 ms stddev 5.971, 0 failed
progress: 36.0 s, 783.7 tps, lat 10.206 ms stddev 5.959, 0 failed
progress: 42.0 s, 773.6 tps, lat 10.337 ms stddev 6.781, 0 failed
progress: 48.0 s, 788.4 tps, lat 10.144 ms stddev 5.787, 0 failed
progress: 54.0 s, 789.3 tps, lat 10.131 ms stddev 5.813, 0 failed
progress: 60.0 s, 790.2 tps, lat 10.123 ms stddev 5.791, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 46810
number of failed transactions: 0 (0.000%)
latency average = 10.248 ms
latency stddev = 5.990 ms
initial connection time = 29.842 ms
tps = 780.372256 (without initial connection time)
```
Что изменилось и почему?
**Ничего не поменялось, всё в рамках погрешности. Возможно стандартные значения PG16 лучше чем 15**

### Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк
```sql
postgres=# CREATE DATABASE testdb8;
CREATE DATABASE
postgres=# \c testdb8
You are now connected to database "testdb8" as user "postgres".
testdb8=# create table cats(id_d serial primary key,
                         name varchar(10),
                         age int);
CREATE TABLE
testdb8=# INSERT INTO cats(name,age) SELECT 'name', 3 FROM generate_series(1,1000000);
INSERT 0 1000000
```

Посмотреть размер файла с таблицей
```sql
testdb8=# SELECT pg_size_pretty(pg_total_relation_size('cats'));
 pg_size_pretty
----------------
 64 MB
(1 row)

testdb8=# SELECT pg_relation_filepath('cats');
 pg_relation_filepath
----------------------
 base/24730/24746
(1 row)

testdb8=#
```

```bash
> /var/lib/postgresql/16/main/base# ls -l 24730/24746
-rw------- 1 postgres postgres 44285952 Jun 29 16:39 24730/24746
```

### 5 раз обновить все строчки и добавить к каждой строчке любой символ

```sql
testdb8=# UPDATE cats SET name = name || '1';
UPDATE 1000000
testdb8=# UPDATE cats SET name = name || '2';
UPDATE 1000000
testdb8=# UPDATE cats SET name = name || '3';
UPDATE 1000000
testdb8=# UPDATE cats SET name = name || '4';
UPDATE 1000000
testdb8=# UPDATE cats SET name = name || '5';
UPDATE 1000000
testdb8=# SELECT pg_size_pretty(pg_total_relation_size('cats'));
 pg_size_pretty
----------------
 332 MB
(1 row)

testdb8=#
```
### Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
```sql
testdb8=# SELECT relname, n_live_tup, n_dead_tup,
trunc(100*n_dead_tup/(n_live_tup+1))::float AS "ratio%", last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'cats';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
 cats    |    1000000 |    4999211 |    499 | 2024-06-29 16:43:19.433266+03
(1 row)

```
### Подождать некоторое время, проверяя, пришел ли автовакуум

```sql
testdb8=# SELECT relname, n_live_tup, n_dead_tup,
trunc(100*n_dead_tup/(n_live_tup+1))::float AS "ratio%", last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'cats';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 cats    |    1000000 |          0 |      0 | 2024-06-29 16:47:20.194184+03
(1 row)

```
### 5 раз обновить все строчки и добавить к каждой строчке любой символ. Посмотреть размер файла с таблицей
```sql
testdb8=# UPDATE cats SET name = 'cat';
UPDATE cats SET name = name || '1';
UPDATE cats SET name = name || '2';
UPDATE cats SET name = name || '3';
UPDATE cats SET name = name || '4';
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
testdb8=# SELECT pg_size_pretty(pg_total_relation_size('cats'));
 pg_size_pretty
----------------
 333 MB
(1 row)
```

### Отключить Автовакуум на конкретной таблице
```sql
testdb8=# ALTER TABLE cats SET (autovacuum_enabled = off);
ALTER TABLE
```

### 10 раз обновить все строчки и добавить к каждой строчке любой символ. Посмотреть размер файла с таблицей
```sql
testdb8=# UPDATE cats SET name = '0';
UPDATE cats SET name = name || '1';
UPDATE cats SET name = name || '2';
UPDATE cats SET name = name || '3';
UPDATE cats SET name = name || '4';
UPDATE cats SET name = name || '5';
UPDATE cats SET name = name || '6';
UPDATE cats SET name = name || '7';
UPDATE cats SET name = name || '8';
UPDATE cats SET name = name || '9';
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
testdb8=# SELECT pg_size_pretty(pg_total_relation_size('cats'));
 pg_size_pretty
----------------
 551 MB
(1 row)
```

Объясните полученный результат
_К каждой живой строке 10 мёртвых_

### Не забудьте включить автовакуум)
```sql
testdb8=# ALTER TABLE cats SET (autovacuum_enabled = off);
ALTER TABLE
```

### Задание со *:
_Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице.
Не забыть вывести номер шага цикла._

```sql
DO $$
DECLARE
    i INT := 0;
BEGIN
    WHILE i < 10 LOOP
        UPDATE cats
        SET name = name || CAST(i AS TEXT);
		i := i + 1;
    END LOOP;
END $$;
```

### Чистим мусор
1. возвращаем параметы.
2. Удаляем БД
```sql
postgres=# drop database testdb8;
DROP DATABASE
```
3. Удаляем таблицы
```sql
postgres=# \dt
              List of relations
 Schema |       Name       | Type  |  Owner
--------+------------------+-------+----------
 public | pgbench_accounts | table | postgres
 public | pgbench_branches | table | postgres
 public | pgbench_history  | table | postgres
 public | pgbench_tellers  | table | postgres
(4 rows)

postgres=# drop table pgbench_branches;
DROP TABLE
postgres=# drop table pgbench_accounts;
DROP TABLE
postgres=# drop table pgbench_history;
DROP TABLE
postgres=# drop table pgbench_tellers;
DROP TABLE
```
