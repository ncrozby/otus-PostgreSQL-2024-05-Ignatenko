# Работа с уровнями изоляции транзакции в PostgreSQL

## (Сессия 1)(Сессия 2) Запустить везде psql из под пользователя postgres
```bash
postgres@ubunt:~$ psql
psql (16.2 (Ubuntu 16.2-1ubuntu4))
Type "help" for help.
```

## (Сессия 1)(Сессия 2) Выключить auto commit
```sql
postgres=# \set AUTOCOMMIT OFF
```

## (Сессия 1) Создать новую таблицу и наполнить ее данными 
```sql
postgres=*# create table cats(id_d serial primary key,
                         name_cats varchar(50),
                         date_born date);
CREATE TABLE
postgres=*# insert into cats(name_cats, date_born)
values ('Мурзик','12/20/2023'),
       ('Ваха','12/05/2023'),
       ('Бегемот','12/10/2023');
INSERT 0 3
postgres=*# commit;
```

## (Сессия 1) посмотреть текущий уровень изоляции: 
```sql
postgres=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)

postgres=*# end;
COMMIT
```
## (Сессия 1)(Сессия 2) начать новую транзакцию не меняя уровtym изоляции
```sql
postgres=# begin;
BEGIN
```

## (Сессия 1) добавить новую запись 
```sql
postgres=*# insert into cats(name_cats, date_born) values ('Барсик','01/12/2024');
INSERT 0 1
```

## (Сессия 2) сделать select from cats
```sql
postgres=*# select * from cats;
 id_d | name_cats | date_born
------+-----------+------------
    1 | Мурзик    | 2023-12-20
    2 | Ваха      | 2023-12-05
    3 | Бегемот   | 2023-12-10
(3 rows)
```
_Транзакция в первой сессии ещё не завершена, никто кроме неё с Барсиком ещё не знакомы._

## (Сессия 1) завершить первую транзакцию
```sql
postgres=*# commit;
COMMIT
```

## (Сессия 2) сделать select from persons во второй сессии
```sql
postgres=*# select * from cats;
 id_d | name_cats | date_born
------+-----------+------------
    1 | Мурзик    | 2023-12-20
    2 | Ваха      | 2023-12-05
    3 | Бегемот   | 2023-12-10
    4 | Барсик    | 2024-01-12
(4 rows)
```
_Транзакция в первой сессии завершена, и все узнали о Барсике._

## (Сессия 2) завершить транзакцию
```sql
postgres=*# commit;
COMMIT
```

## (Сессия 1)(Сессия 2) начать новые но уже repeatable read транзации - 
```sql
postgres=# begin transaction isolation level repeatable read;
BEGIN
```

## (Сессия 1) в первой сессии добавить новую запись 
```sql
postgres=*# insert into cats(name_cats, date_born) values ('Кися','12/10/2023');
INSERT 0 1
```

## (Сессия 2) сделать select* from cats
```sql
postgres=*# select * from cats;
 id_d | name_cats | date_born
------+-----------+------------
    1 | Мурзик    | 2023-12-20
    2 | Ваха      | 2023-12-05
    3 | Бегемот   | 2023-12-10
    4 | Барсик    | 2024-01-12
(4 rows)
```
_Про Кисю, никто ещё не знает, кроме сессии 1_

## (Сессия 1) завершить транзакцию
```sql
postgres=*# commit;
COMMIT
```

## (Сессия 2) сделать select from cats 
```sql
postgres=*# select * from cats;
 id_d | name_cats | date_born
------+-----------+------------
    1 | Мурзик    | 2023-12-20
    2 | Ваха      | 2023-12-05
    3 | Бегемот   | 2023-12-10
    4 | Барсик    | 2024-01-12
(4 rows)
```
_Мы в домике (repeatable read), пока не выдем не узнаем ничего нового._

## (Сессия 2) завершить вторую транзакцию и сделать select
```sql
postgres=*# commit;
COMMIT
postgres=# select * from cats;
 id_d | name_cats | date_born
------+-----------+------------
    1 | Мурзик    | 2023-12-20
    2 | Ваха      | 2023-12-05
    3 | Бегемот   | 2023-12-10
    4 | Барсик    | 2024-01-12
    5 | Кися      | 2023-12-10
(5 rows)
```

_После commit, мы вышли на улицу и узнале о Кисе_

