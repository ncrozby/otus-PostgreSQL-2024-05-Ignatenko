# Журналы. 
## Работа с журналами

Используем ВМ Oracle Virtual Box с Ubuntu (24.04 LTS (Noble Numbat))
PostgreSQL 16.2

### Настройте выполнение контрольной точки раз в 30 секунд. 

```sql
postgres=# alter system set checkpoint_timeout = '30s';
ALTER SYSTEM
```
Рестарт кластера:
```bash
# pg_ctlcluster 16 main restart
```

Проверим что параметр применился
```sql
postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 30s
(1 row)
```

### 10 минут c помощью утилиты pgbench подавайте нагрузку.

Фиксируем LSN:
```sql
postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 2/95D66448
(1 row)
```

Очищаем статистику:
```sql
postgres=# SELECT pg_stat_reset_shared('bgwriter');
 pg_stat_reset_shared
----------------------

(1 row)

postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 1
checkpoints_req       | 0
checkpoint_write_time | 0
checkpoint_sync_time  | 0
buffers_checkpoint    | 0
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 0
buffers_backend_fsync | 0
buffers_alloc         | 2
stats_reset           | 2024-07-05 11:47:01.364129+03
```

Запускаем в паралелльном окне pgbench:
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
done in 0.29 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.12 s, vacuum 0.09 s, primary keys 0.08 s).


>>> pgbench -c8 -P 6 -T 600 -U postgres postgres
pgbench -c8 -P 6 -T 600 -U postgres postgres
pgbench (16.3 (Ubuntu 16.3-0ubuntu0.24.04.1))
starting vacuum...end.
progress: 6.0 s, 743.8 tps, lat 10.693 ms stddev 6.503, 0 failed
progress: 12.0 s, 752.0 tps, lat 10.637 ms stddev 6.291, 0 failed
...
progress: 594.0 s, 724.5 tps, lat 11.037 ms stddev 6.409, 0 failed
progress: 600.0 s, 723.2 tps, lat 11.063 ms stddev 6.794, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 439737
number of failed transactions: 0 (0.000%)
latency average = 10.912 ms
latency stddev = 6.540 ms
initial connection time = 26.671 ms
tps = 732.904367 (without initial connection time)
```

### Измерьте, какой объем журнальных файлов был сгенерирован за это время. 

```sql
postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 2/B3AF8610
(1 row)

postgres=# SELECT '2/B3AF8610'::pg_lsn - '2/95D66448'::pg_lsn;
 ?column?
-----------
 500769224
(1 row)
```

**500769224 Байта**

### Оцените, какой объем приходится в среднем на одну контрольную точку. 

За 10 минут с интервалом 30сек, создаётся 20 контрольных точек, но за счёт сдвига наши данные могли попасть ещё в одну.

```sql
postgres=# select 500769224/20/1024/1024;
 ?column?
----------
       23
(1 row)

postgres=# select 500769224/21/1024/1024;
 ?column?
----------
       22
(1 row)
```
Итого 22-23ГБ на одну контрольную точку.

### Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?

Проверяем максимальный размер WAL:
```sql
postgres=# show max_wal_size;
 max_wal_size
--------------
 16GB
(1 row)
```

Смотрим статистику:
```sql
postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 23
checkpoints_req       | 0
checkpoint_write_time | 538935
checkpoint_sync_time  | 532
buffers_checkpoint    | 40362
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 2805
buffers_backend_fsync | 0
buffers_alloc         | 4983
stats_reset           | 2024-07-05 11:47:01.364129+03
```
И вот тут разрыв шаблона, все контрольные точки создавались во время. checkpoints_req = 0! По логике 5-10 должно быть по требованию. 

_Файлы WAL проверяем_:
```
-rw------- 1 postgres postgres 16777216 Jul  5 11:47 000000010000000300000066
-rw------- 1 postgres postgres 16777216 Jul  5 11:47 000000010000000300000068
-rw------- 1 postgres postgres 16777216 Jul  5 11:48 000000010000000300000067
-rw------- 1 postgres postgres 16777216 Jul  5 11:48 000000010000000300000069
-rw------- 1 postgres postgres 16777216 Jul  5 11:48 00000001000000030000006A
-rw------- 1 postgres postgres 16777216 Jul  5 11:49 00000001000000030000006B
-rw------- 1 postgres postgres 16777216 Jul  5 11:49 00000001000000030000006C
-rw------- 1 postgres postgres 16777216 Jul  5 11:49 00000001000000030000006D
-rw------- 1 postgres postgres 16777216 Jul  5 11:50 00000001000000030000006E
-rw------- 1 postgres postgres 16777216 Jul  5 11:50 00000001000000030000006F
-rw------- 1 postgres postgres 16777216 Jul  5 11:50 000000010000000300000071
-rw------- 1 postgres postgres 16777216 Jul  5 11:51 000000010000000300000070
-rw------- 1 postgres postgres 16777216 Jul  5 11:51 000000010000000300000072
-rw------- 1 postgres postgres 16777216 Jul  5 11:51 000000010000000300000073
-rw------- 1 postgres postgres 16777216 Jul  5 11:52 000000010000000300000074
-rw------- 1 postgres postgres 16777216 Jul  5 11:52 000000010000000300000075
-rw------- 1 postgres postgres 16777216 Jul  5 11:52 000000010000000300000076
-rw------- 1 postgres postgres 16777216 Jul  5 11:53 000000010000000300000077
-rw------- 1 postgres postgres 16777216 Jul  5 11:53 000000010000000300000078
-rw------- 1 postgres postgres 16777216 Jul  5 11:53 000000010000000300000079
-rw------- 1 postgres postgres 16777216 Jul  5 11:54 00000001000000030000007A
-rw------- 1 postgres postgres 16777216 Jul  5 11:54 00000001000000030000007B
-rw------- 1 postgres postgres 16777216 Jul  5 11:54 00000001000000030000007C
-rw------- 1 postgres postgres 16777216 Jul  5 11:55 00000001000000030000007D
-rw------- 1 postgres postgres 16777216 Jul  5 11:55 00000001000000030000007F
-rw------- 1 postgres postgres 16777216 Jul  5 11:55 00000001000000030000007E
-rw------- 1 postgres postgres 16777216 Jul  5 11:56 000000010000000300000080
-rw------- 1 postgres postgres 16777216 Jul  5 11:56 000000010000000300000081
-rw------- 1 postgres postgres 16777216 Jul  5 11:56 000000010000000300000082
-rw------- 1 postgres postgres 16777216 Jul  5 11:57 000000010000000300000083
```
Всё сходится 3 файла в минуту. Не понимаю.

### Сравните tps в синхронном/асинхронном режмие утилитой pgbench. Объясните полученный результат.

|Вариант|fsync|synchronous_commit|tps|
|-|--------|---|----|
|1 |on|on| 732.904367|
|2 |off|on|2813.895689|
|3 |on|off|2490.226183|

В первом варианте ждём завершения физической записи на диск. Во 2-м и 3-м не ждём, но в 3-м после сбоя можем восстановить базу данных с небольшими потерями, а во втором, совсем беда.

### Создайте новый кластер с включенной контрольной суммой страниц.
Создаём кластер
```
postgres@ubunt:~$ pg_ctl -D /var/lib/PGHW9/data initdb
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

creating directory /var/lib/PGHW9/data ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Europe/Moscow
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

initdb: warning: enabling "trust" authentication for local connections
initdb: hint: You can change this by editing pg_hba.conf or using the option -A, or --auth-local and --auth-host, the next time you run initdb.

Success. You can now start the database server using:

    /usr/lib/postgresql/16/bin/pg_ctl -D /var/lib/PGHW9/data -l logfile start
```
Пробуем запустить:
```bash
postgres > pg_ctl -D /var/lib/PGHW9/data -l logfile start
waiting for server to start.... stopped waiting
pg_ctl: could not start server
Examine the log output.
```

Ага, порт то занят.
Меняем порт:
```
postgres > vi  /var/lib/PGHW9/data/postrgesql.conf 
...
port = 5332
...
```

Пробуем запустить ещё раз:
```bash
postgres > pg_ctl -D /var/lib/PGHW9/data -l logfile start
waiting for server to start.... done
server started
```
_Вот что знания и опыт творят!_
Включаем контрольную сумму, предварительно останавливаем кластер.

```
postgres > /usr/lib/postgresql/16/bin/pg_ctl -D /var/lib/PGHW9/data -l logfile stop
waiting for server to shut down.... done
server stopped
```
Для изменения параметра нужен root:
```
# /usr/lib/postgresql/16/bin/pg_checksums --enable -D /var/lib/PGHW9/data
Checksum operation completed
Files scanned:   948
Blocks scanned:  2834
Files written:  780
Blocks written: 2834
pg_checksums: syncing data directory
pg_checksums: updating control file
Checksums enabled in cluster
```

Запускаем ещё раз:
```bash
postgres > pg_ctl -D /var/lib/PGHW9/data -l logfile start
waiting for server to start.... done
server started
```

Проверяем что параметр поменялся:
```sql
postgres=# SHOW data_checksums;
 data_checksums
----------------
 on
(1 row)
```
Есть такой.
### Создайте таблицу. Вставьте несколько значений.
```sql
postgres=# create table cats(id_d serial primary key,
                         name varchar(15),
                         age int);
CREATE TABLE
postgres=# INSERT INTO cats(name,age) SELECT 'noname', 3 FROM generate_series(1,10);
INSERT 0 10
```
Посмтрим, что нужно ломать:
```sql
postgres=# SELECT pg_relation_filepath('cats');
 pg_relation_filepath
----------------------
 base/5/16389
(1 row)
```
### Выключите кластер. 
```
postgres@ubunt:/var/lib/PGHW9/data$ /usr/lib/postgresql/16/bin/pg_ctl -D /var/lib/PGHW9/data -l logfile stop
waiting for server to shut down.... done
server stopped
```

### Измените пару байт в таблице. 
```
postrges> vi /var/lib/PGHW9/data/base/5/16389
```

### Включите кластер и сделайте выборку из таблицы. 
Запускаем ещё раз:
```bash
postgres > pg_ctl -D /var/lib/PGHW9/data -l logfile start
waiting for server to start.... done
server started
```
Читаем таблицу:
```sql
postgres=# select * from cats;
WARNING:  page verification failed, calculated checksum 58745 but expected 31403
ERROR:  invalid page in block 0 of relation base/5/16389
```
>Что и почему произошло? как проигнорировать ошибку и продолжить работу?

_Сломали таблицу. контрольная сумма страницы не сходится._

_"проигнорировать ошибку и продолжить работу" - очень оптимистично! Но попробуем_

```sql
postgres=# SET ignore_checksum_failure = on;
SET
postgres=# select * from cats;
WARNING:  page verification failed, calculated checksum 58745 but expected 31403
ERROR:  invalid memory alloc request size 18446744073709551613
```
Видимо поменял не те "пару байтов"

После попыток вернуть в файле, то что поменял, стало ещё хуже:
```sql
postgres=# select * from cats;
WARNING:  page verification failed, calculated checksum 56039 but expected 38762
server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.
The connection to the server was lost. Attempting reset: Failed.
The connection to the server was lost. Attempting reset: Failed.
!?>
```
Остаётс только удалять таблицу.

Читаем гуру - Евгения Рогова:(https://habr.com/ru/companies/postgrespro/articles/461523 )
> Параметр ignore_checksum_failure позволяет попробовать прочитать таблицу, естественно с риском получить искаженные данные.

**попробовали не получилось, даже искажённые прочитать**

И вот что Евгений ещё пишет по этому поводу:

> Но что делать, если данные невозможно восстановить из резервной копии?

_Бежать!_

