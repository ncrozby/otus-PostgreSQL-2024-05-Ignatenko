# Резервное копирование и восстановление 

## Бэкапы

*Цель:* 
*применить логический бэкап. Восстановиться из бэкапа.*


Используем ВМ Oracle Virtual Box с Ubuntu (24.04 LTS (Noble Numbat))
PostgreSQL 16.2

## 1. Создаем ВМ/докер c ПГ.
Как обычно, используем существующую ВМ
## 2. Создаем БД, схему и в ней таблицу.

```
postgres=# create database zoo;
CREATE DATABASE

postgres=# \l
                                                       List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | ICU Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+-------------+-------------+------------+-----------+-----------------------
 postgres  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |
 template0 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
 zoo       | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |
(4 rows)

postgres=# create schema birds;
CREATE SCHEMA
```

### 3. Заполним таблицы автосгенерированными 100 записями.
```
postgres=# create table birds.animals as select generate_series(1, 100) as id, md5(random()::text)::char(10) as name, md5(random()::text)::char(10) as location;
SELECT 100

postgres=# select count(*) from birds.animals;
 count
-------
   100
(1 row)

postgres=#
postgres=# select * from birds.animals limit 5;
 id |    name    |  location
----+------------+------------
  1 | 557ade5ca4 | 5e5c963de7
  2 | 188319eb42 | beebf2c236
  3 | f8172397ea | fa2ac1f9d6
  4 | f470578280 | c41e64e2b5
  5 | d57b7b3c33 | f8068f607b
(5 rows)

postgres=#
```
### 4. Под линукс пользователем Postgres создадим каталог для бэкапов

```bash
mkdir /tmp/BackUP
```

### 5. Сделаем логический бэкап используя утилиту COPY

```
postgres=# \copy birds.animals to '/tmp/BackUP/animals.sql' with delimiter ';';
COPY 100
```
### 6. Восстановим в 2 таблицу данные из бэкапа.
```sql
postgres=# create table birds.cats(id_d serial primary key,
                         name varchar(10),
                         location varchar (10));
CREATE TABLE
postgres=# \copy birds.cats from '/tmp/BackUP/animals.sql' with delimiter ';';
COPY 100
```
Посмотрим что получилось:
```
postgres=# select * from birds.cats limit 5;
 id_d |    name    |  location
------+------------+------------
    1 | 557ade5ca4 | 5e5c963de7
    2 | 188319eb42 | beebf2c236
    3 | f8172397ea | fa2ac1f9d6
    4 | f470578280 | c41e64e2b5
    5 | d57b7b3c33 | f8068f607b
(5 rows)

postgres=# select count(*) from birds.cats;
 count
-------
   100
(1 row)

postgres=#
```

### 7. Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц
```bash
postgres@ubunt:~$ pg_dump -d zoo -Fc > /tmp/BackUP/zoo.gz
postgres@ubunt:~$ ls -l /tmp/BackUP/
total 8
-rw-rw-r-- 1 postgres postgres 2492 Jul 17 15:27 animals.sql
-rw-rw-r-- 1 postgres postgres 1589 Jul 17 15:36 zoo.gz
postgres@ubunt:~$
```

### 8. Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!
Создаём БД, и запускаем восстановление:
```bash
postgres createdb forest
pg_restore -d forest -t cats /tmp/BackUP/zoo.gz
```
Не многословно...


Появилась ли БД forest?
```
postgres=# \l
                                                       List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | ICU Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+-------------+-------------+------------+-----------+-----------------------
 forest    | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |
 postgres  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |
 template0 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
 zoo       | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |
(5 rows)
```
Есть.


Проверим есть ли то что нужно
```
postgres=# \c forest
You are now connected to database "forest" as user "postgres".
forest=# select count(*) from birds.cats;
 count
-------
   100
(1 row)
```
Появилось.


Проверим нет ли то чего не нужно
```
forest=# select count(*) from birds.animals;
ERROR:  relation "birds.animals" does not exist
LINE 1: select count(*) from birds.animals;
                             ^
forest=#
```
Всё как надо
