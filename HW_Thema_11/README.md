# Настройка PostgreSQL  
## Нагрузочное тестирование и тюнинг PostgreSQL

Используем ВМ Oracle Virtual Box с Ubuntu (24.04 LTS (Noble Numbat))
PostgreSQL 16.2
ЦП-4; ОП-4ГБ

### 1. развернуть виртуальную машину любым удобным способом
### 2. поставить на неё PostgreSQL 15 любым способом
### 3. настроить кластер PostgreSQL 15 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины

Отключаем синхронный коммит:
```sql
postgres=# alter system set synchronous_commit = off;
ALTER SYSTEM
```
### 4. нагрузить кластер через утилиту через утилиту pgbench
Проводим эксперименты с системой сос тандартной конфигурацией и после изменения параметров.
Для каждой системы выполняем 2 теста с 8 параллельными процессами и 3. 8 - перегружает подсистему ввода-вывода, 3 оптимальная нагрузка.

#### Тест 1
Стнадартная система:
8 потоков:
```
postgres@ubunt:~$ pgbench -c8 -P 6 -T 120 -U postgres postgres
pgbench (16.3 (Ubuntu 16.3-0ubuntu0.24.04.1))
starting vacuum...end.

transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 120 s
number of transactions actually processed: 393980
number of failed transactions: 0 (0.000%)
latency average = 2.432 ms
latency stddev = 1.860 ms
initial connection time = 29.265 ms
tps = 3283.621804 (without initial connection time)
```
#### Тест 2
Стнадартная система:
3 потока
```
postgres@ubunt:~$ pgbench -c3 -P 20 -T 60 -U postgres postgres
pgbench (16.3 (Ubuntu 16.3-0ubuntu0.24.04.1))
starting vacuum...end.
progress: 20.0 s, 3730.0 tps, lat 0.801 ms stddev 0.626, 0 failed
progress: 40.0 s, 3340.8 tps, lat 0.895 ms stddev 0.589, 0 failed
progress: 60.0 s, 3586.6 tps, lat 0.834 ms stddev 0.482, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 3
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 213152
number of failed transactions: 0 (0.000%)
latency average = 0.842 ms
latency stddev = 0.571 ms
initial connection time = 10.682 ms
tps = 3553.108937 (without initial connection time)
```
#### Меняем параметры
```
shared_buffers = 1GB
effective_cache_size = 3GB
work_mem = 10MB
maintenance_work_mem = 205MB
```
#### Тест 3
Изменённая система:
8 потоков
```
postgres@ubunt:~$ pgbench -c8 -P 20 -T 60 -U postgres postgres
pgbench (16.3 (Ubuntu 16.3-0ubuntu0.24.04.1))
starting vacuum...end.

transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 207688
number of failed transactions: 0 (0.000%)
latency average = 2.306 ms
latency stddev = 1.337 ms
initial connection time = 28.907 ms
tps = 3462.415888 (without initial connection time)
```
#### Тест 4
Изменённая система:
3 потока:
```
postgres@ubunt:~$ pgbench -c3 -P 20 -T 60 -U postgres postgres
pgbench (16.3 (Ubuntu 16.3-0ubuntu0.24.04.1))
starting vacuum...end.

transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 3
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 247252
number of failed transactions: 0 (0.000%)
latency average = 0.726 ms
latency stddev = 0.163 ms
initial connection time = 9.813 ms
tps = 4121.451157 (without initial connection time)
```

### 5. написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему
|потоки|стандарт|изменённая|
|------|--------|----------|
|8|3283|3462|
|3|3553|4121|

При 8 патоках подсистема ввода вывода захлёбывается, и становится узким местом и изменение параметров не даёт всего около 5%. При нормальной работе увеличение памяти имеет более существенное преемущество уже около 15%.
Увеличение объёма кэша, сокращает количество операций чтения с диска, от этого и быстрее.
