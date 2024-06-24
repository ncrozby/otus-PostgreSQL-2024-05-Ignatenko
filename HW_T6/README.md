# Установка и настройка PostgteSQL в контейнере Docker

Используем ВМ Oracle Virtual Box с Ubuntu (24.04 LTS (Noble Numbat))
Docker инсталлирован (Docker version 26.1.4, build 5650f9b)

## Подготовка и запуск Postgesql 16 в контейнере

### Создаём общую сеть pg-net
```bash
# docker network create pg-net
e9139075b505b8f5e10d9e9591fd5572110293bafa44b36e407ed966696fd16d
```

### Запуск контейнера
Директория для файлов БД родительской ОС будет размещена в /var/lib/postgres. Директория будет создана автоматически

```bash
# docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:16
Unable to find image 'postgres:16' locally
16: Pulling from library/postgres
2cc3ae149d28: Pull complete
d1a63825d58e: Pull complete
ed6f372fe58d: Pull complete
35f975e69306: Pull complete
40c4fe86e99d: Pull complete
4795e1a32ff6: Pull complete
bcb5a54ae87d: Pull complete
d3983228bec6: Pull complete
5378bf7229e9: Pull complete
bba3241011a6: Pull complete
5e1d0413d05a: Pull complete
6a489170d05e: Pull complete
440b39aff272: Pull complete
582c79113570: Pull complete
Digest: sha256:46aa2ee5d664b275f05d1a963b30fff60fb422b4b594d509765c42db46d48881
Status: Downloaded newer image for postgres:16
dd18507dce292a98e1243118b14a81577fb9c96e8dc2513d7ea7856d802a4ea0
docker: Error response from daemon: driver failed programming external connectivity on endpoint pg-server (2c7b418b836c969f8c7d775962dd2c3a5034d24720e4b8537dbe15f055a9b2d3): Error starting userland proxy: listen tcp4 0.0.0.0:5432: bind: address already in use.
```
В родительской ОС запущен экземпляр Postgresql, как следствие, порт 5432 занят. Останавливаем службу Postgresql, и продолжаем.
Предварительно удаляем старый образ:

```bash
# docker rm dd18507dce292a98e1243118b14a81577fb9c96e8dc2513d7ea7856d802a4ea0
```
Запускаем повторно:
```bash
#docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:16

```
### Подключаемся к контейнеру

```bash
# docker run -it --rm --network pg-net --name pg-client postgres:16 psql -h pg-server -U postgres
3bd77762e5af2d3bfbafa0fe8c28be66093dd19cfe555a3f7d4df2ba6b386dbf
```
теперь, всё работает.

### Создаём таблицу и заполняем её

```sql
postgres=# сreate table departments(id_d serial primary key,
                                                name_d varchar(50),
                                                date_create date);
CREATE TABLE
postgres=# insert into departments(name_d, date_create)
values ('юридический','12/20/2023'),
       ('бухгалтерия','12/05/2023'),
       ('продажи','12/10/2023');
INSERT 0 3
postgres=# select * from departments;
 id_d |   name_d    | date_create
------+-------------+-------------
    1 | юридический | 2023-12-20
    2 | бухгалтерия | 2023-12-05
    3 | продажи     | 2023-12-10
(3 rows)
```

### Подключаемся к БД из родительской ОС

```bash
> psql -h 172.18.0.1 -U postgres -d postgres
Password for user postgres:
psql (16.2 (Ubuntu 16.2-1ubuntu4), server 16.3 (Debian 16.3-1.pgdg120+1))
```

### Проверим нашу таблицу

```sql
postgres=#  select * from departments;
 id_d |   name_d    | date_create
------+-------------+-------------
    1 | юридический | 2023-12-20
    2 | бухгалтерия | 2023-12-05
    3 | продажи     | 2023-12-10
(3 rows)
```

### Проверяем подключение из Windows используя DBeaver.
Настраиваем соединение на IP ВМ и порт 5432.

Пробуем подключится к БД postgres, соединение устанавливается. Таблица существует и в ней записи.

Как сюда вставить картинки не знаю, поэтому придётся поверить на слово.

### Удаляем контейнер

Для начала найдём ID контейнера:
```bash
# docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED       STATUS       PORTS                                       NAMES
0edcaf37cf64   postgres:16   "docker-entrypoint.s…"   2 hours ago   Up 2 hours   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
```

Уничтожаем контейнер, предварительно оставновив:
```bash
# docker stop 0edcaf37cf64
0edcaf37cf64
# docker rm 0edcaf37cf64
0edcaf37cf64
```

Смотрим осталось ли что:
```bash
# # docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

Проверим, вдруг работает.
```bash
# docker run -it --rm --network pg-net --name pg-client postgres:16 psql -h pg-server -U postgres
psql: error: could not translate host name "pg-server" to address: Temporary failure in name resolution
```
Может удалённо?:
```bash
# psql -h 172.18.0.1 -U postgres -d postgres
psql: error: connection to server at "172.18.0.1", port 5432 failed: Connection refused
        Is the server running on that host and accepting TCP/IP connections?
```
Чуда не произошло.

Проверим что стало с файлами данных, после удаления контейнера:
```bash
# ls /var/lib/postgres/
base          pg_hba.conf    pg_notify     pg_stat      pg_twophase  postgresql.auto.conf
global        pg_ident.conf  pg_replslot   pg_stat_tmp  PG_VERSION   postgresql.conf
pg_commit_ts  pg_logical     pg_serial     pg_subtrans  pg_wal       postmaster.opts
pg_dynshmem   pg_multixact   pg_snapshots  pg_tblspc    pg_xact 
```
Файлы на месте есть шанс, что данные не потеряются.

### Создаём новый контейнер

```bash
# # docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:16
dc1a28d62ab7c5e3d5d707c3c7ac15c9f1919b1d788ed35c81aa2edddb18286d
```

### Проверяем

```bash
# docker run -it --rm --network pg-net --name pg-client postgres:16 psql -h pg-server -U postgres
Password for user postgres:
psql (16.3 (Debian 16.3-1.pgdg120+1))
Type "help" for help.
```
Уже хорошо.

А есть ли таблица?
```sql
postgres=# select * from departments;
 id_d |   name_d    | date_create
------+-------------+-------------
    1 | юридический | 2023-12-20
    2 | бухгалтерия | 2023-12-05
    3 | продажи     | 2023-12-10
(3 rows)

postgres=#
```

### Удаляем мусор

```bash
# docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS                                       NAMES
dc1a28d62ab7   postgres:16   "docker-entrypoint.s…"   3 minutes ago   Up 3 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server

# docker stop dc1a28d62ab7
dc1a28d62ab7

# docker rm dc1a28d62ab7
dc1a28d62ab7

# docker network rm pg-net
pg-net
```

## готовы к следующим заданиям
