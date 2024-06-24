# Установка и настройка PostgreSQL

Используем ВМ Oracle Virtual Box с Ubuntu (24.04 LTS (Noble Numbat))
PostgreSQL 16

## Проверяем что кластер запущен
```bash
# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```
## Создаём таблицу и заполняем данными

```sql
$ psql
psql (16.2 (Ubuntu 16.2-1ubuntu4))
Type "help" for help.

postgres=# create table cats(id_d serial primary key,
                         name_cats varchar(50),
                         date_born date);
CREATE TABLE
postgres=# insert into cats(name_cats, date_born)
values ('Мурзик','12/20/2023'),
       ('Ваха','12/05/2023'),
       ('Бегемот','12/10/2023');
INSERT 0 3

postgres=# select * from cats;
 id_d | name_cats | date_born
------+-----------+------------
    1 | Мурзик    | 2023-12-20
    2 | Ваха      | 2023-12-05
    3 | Бегемот   | 2023-12-10
(3 rows)
```

## Останавливаем кластер

```bash
# pg_ctlcluster 16 main stop
```
Проверим:
```bash
# pg_ctlcluster 16 main status
```

## Создаём новую директорию и переносим файлы

Подключили новый диск к ВМ, создали новую ФС.

Создаём новую директорию , делаем postgres влвдельцем, монтируем
```bash
# mkdir -p /otus/pgdata01
# chown -R postgres:postgres /otus
# mount /dev/sdb /otus/pgdata01
```
Переносим файлы в новую директорию
```bash
# mv /var/lib/postgresql/16 /otus/pgdata01
```

## Пытаемся запустить кластер
```bash
# pg_ctlcluster 16 main start
Error: /var/lib/postgresql/16/main is not accessible or does not exist
```
Не получилось. Параметры всё ещё смотрят на старый путь

## Ищем нужный параметр и меняем его

Идём в директорию где находятся файлы с параметрами:
```bash
# cd /etc/postgresql/16/main
```

Ищем где нужный параметр:
```bash
# grep /var/lib *
grep: conf.d: Is a directory
postgresql.conf:data_directory = '/var/lib/postgresql/16/main'          # use data in another directory
```
Нашли, параметр data_directory в файле postgresql.conf

Делаем резервную копию файла postgresql.conf:
```bash
# cp postgresql.conf postgresql.conf.ORIG
```

В файле postgresql.conf, используя vi делаем изменения:
```bash

#------------------------------------------------------------------------------
# FILE LOCATIONS
#------------------------------------------------------------------------------

# The default values of these variables are driven from the -D command-line
# option or PGDATA environment variable, represented here as ConfigDir.

#data_directory = '/var/lib/postgresql/16/main'         # use data in another directory
data_directory = '/otus/pgdata01/16/main'               # use data in another directory

```

## Запускаем кластер

```bash
# pg_ctlcluster 16 main stаrt
```

Сразу проверим:
```bash
# pg_ctlcluster 16 main status
pg_ctl: server is running (PID: 9442)
/usr/lib/postgresql/16/bin/postgres "-D" "/otus/pgdata01/16/main" "-c" "config_file=/etc/postgresql/16/main/postgresql.conf"
```

## Проверяем результат
Проверим нашу таблицу, в psql выполним команду:
```sql
postgres=# select * from cats;
 id_d | name_cats | date_born
------+-----------+------------
    1 | Мурзик    | 2023-12-20
    2 | Ваха      | 2023-12-05
    3 | Бегемот   | 2023-12-10
(3 rows)
```

Ура, сработало
