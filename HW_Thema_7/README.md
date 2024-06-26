# Логический уровень PostgreSQL 

Используем ВМ Oracle Virtual Box с Ubuntu (24.04 LTS (Noble Numbat))
PostgreSQL 16

## 1. создайте новую базу данных testdb
```sql
postgres=# CREATE DATABASE testdb;
CREATE DATABASE
```

## 2. зайдите в созданную базу данных под пользователем postgres
```sql
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
```

## 3. создайте новую схему testnm
```sql
testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA
```

## 4. создайте новую таблицу t1 с одной колонкой c1 типа integer
```sql
testdb=# CREATE TABLE ti (c1 integer);
CREATE TABLE
```

## 5. вставьте строку со значением c1=1
```sql
testdb=# INSERT INTO ti values(1);
INSERT 0 1
```

## 6. создайте новую роль readonly
```sql
testdb=# CREATE ROLE readonly;
CREATE ROLE
```

## 7. дайте новой роли право на подключение к базе данных testdb
```sql
testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT
```

## 8. дайте новой роли право на использование схемы testnm
```sql
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT
```

## 9. дайте новой роли право на select для всех таблиц схемы testnm
```sql
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
```

## 10. создайте пользователя testread с паролем test123
```sql
testdb=# CREATE USER testread WITH PASSWORD 'test123';
CREATE ROLE
```

## 11. дайте роль readonly пользователю testread
```sql
testdb=# GRANT readonly TO testread;
GRANT ROLE
```

## 12. зайдите под пользователем testread в базу данных testdb
```bash
postgres@ubunt:~$ psql -h 127.0.0.1 -d testdb -U testread -W
Password:
psql (16.3 (Ubuntu 16.3-0ubuntu0.24.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.
```

## 14. сделайте select * from ti;
**testread:**
```sql
testdb=> select * from ti;
ERROR:  permission denied for table ti

```

_У пользователя нет полномочий на доступ к таблице public.ti. Мы права давали на схему testnm. dt об этом и говорит ниже._

**postgres:**
```sql
testdb=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | ti   | table | postgres
(1 row)

testdb=# ALTER TABLE ti SET SCHEMA testnm;
ALTER TABLE
```

**testread:**
```sql
testdb=> select * from ti;
ERROR:  relation "ti" does not exist
LINE 1: select * from ti;
                      ^
testdb=> show search_path;
   search_path
-----------------
 "$user", public
(1 row)
```
_Схема testnm не в пути поиска_

**testread:**
```sql
testdb=> select * from testnm.ti;
ERROR:  permission denied for table ti
```

_У пользователя нет полномочий на доступ к таблице. Нужно дать полномочия на таблицу принудительно, либо ещё раз присвоить схему, т.к. операция "GRANT SELECT ON ALL TABLES IN SCHEM" статичная_

**postgres:**
```sql
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
```

**testread:**
```sql
testdb=> select * from testnm.ti;
 c1
----
  1
(1 row)

testdb=> set search_path to testnm, public, "$user", pg_catalog, pg_temp;
SET
testdb=> select * from ti;
 c1
----
  1
(1 row)
```

## 15. вернитесь в базу данных testdb под пользователем postgres, удалите таблицу ti
**postgres:**
```sql
testdb=# DROP TABLE testnm.ti;
DROP TABLE
```

## 16. создайте ее заново но уже с явным указанием имени схемы testnm, вставьте строку со значением c1=1
**postgres:**
```sql
testdb=# DROP TABLE testnm.ti;
DROP TABLE
testdb=# CREATE TABLE testnm.ti (c1 integer);
CREATE TABLE
testdb=# INSERT INTO testnm.ti values(1);
INSERT 0 1
```

## 17. зайдите под пользователем testread в базу данных testdb, сделайте select * from testnm.ti;
**testread:**
```sql
testdb=> select * from testnm.ti;
ERROR:  permission denied for table ti
```
_Причина таже, что и ранее, GRANT на схему был сделан до создания тпблицы_
_Дадим GRANT постоянный_

**postgres:**
```sql
testdb=# ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly;
ALTER DEFAULT PRIVILEGES
testdb=# DROP TABLE testnm.ti;
DROP TABLE
testdb=# CREATE TABLE testnm.ti (c1 integer);
CREATE TABLE
testdb=# INSERT INTO testnm.ti values(1);
INSERT 0 1
```

## 18. сделайте select * from testnm.t1;
**testread:**
```sql
testdb=> select * from testnm.ti;
 c1
----
  1
(1 row)
```
получилось?
ура!
_Ничего удивительного_

## 19. теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
**testread:**
```sql
testdb=> create table t2(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t2(c1 integer);
                     ^
ERROR:  relation "t2" does not exist
LINE 1: insert into t2 values (2);
                    ^
testdb=> insert into ti values (2);
ERROR:  permission denied for table ti
```
а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly? есть идеи как убрать эти права? если нет - смотрите шпаргалку

_- Пришлось смотреть шпаргалку, в PG16 дырка закрыта_

если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды

_- Логично что прав нет, если не дадены_

## 20. теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2); 
расскажите что получилось и почему
_Видимо тоже, что и в п.19, нет прав_
_Для закрытия вопросов, в схеме testnm тоже нет прав_
**testread:**
```sql
testdb=> create table testnm.t3(c1 integer);
ERROR:  permission denied for schema testnm
LINE 1: create table testnm.t3(c1 integer);
                     ^
```

_Подчищаем хвосты_
**postgres:**
```sql
postgres=# drop database testdb;
DROP DATABASE
```
