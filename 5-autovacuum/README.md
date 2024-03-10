# MVCC, `vacuum` и `autovacuum`

## Подготовка окружения
Для выполнения этого задания будет использоваться виртуальная машина, созданная в [самом первом домашнем задании](../1-postgresql-isolation-levels/README.md#подготовка-виртуальной-машины-и-установка-субд). Она использует HDD в качестве хранилища данных, но в остальном её параметры соответствуют параметрам, требуемым в домашнем задании (2 ядра CPU, 4 ГБ ОЗУ), поэтому мне кажется, что она будет достаточна для выполнения этого задания.

На этой машине уже был развёрнут кластер PostgreSQL 14, но задание требует развёртывания 15 версии. Не знаю, насколько существенна разница между 14 и 15 версией в контексте этого задания, поэтому на всякий случай я решил строго следовать заданию и развернуть именно 15 версию в контейнере Docker:
```bash
a-vasenev@pg-dba-vm:~$ sudo mkdir /var/lib/postgresql-docker
a-vasenev@pg-dba-vm:~$ sudo docker run --name pg-server-15 --network pg-network -e POSTGRES_PASSWORD=postgres -p 5433:5432 -v /var/lib/postgresql-docker:/var/lib/postgresql/data -d postgres:15
1a9644f084dd4f1c0a9486ae4d40d1aed33a859167f797498241f8df1d898759
a-vasenev@pg-dba-vm:~$
```

При создании контейнера было указано, что он будет доступен на порту `5433` виртуальной машины-хоста (`-p 5433:5432`). По умолчанию в нём создан пользователь СУБД `postgres` с паролем, указанным при создании контейнера (`-e POSTGRES_PASSWORD=postgres`). Используем эти параметры в утилите pgbench для того, чтобы подключиться к кластеру в контейнере (`-h localhost -p 5433 -U postgres`) и инициализировать БД `postgres` для выполнения нагрузочных тестов (`-i postgres`):
```bash
a-vasenev@pg-dba-vm:~$ pgbench -h localhost -p 5433 -U postgres -i postgres
Password:
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.02 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 2.87 s (drop tables 0.21 s, create tables 0.65 s, client-side generate 1.14 s, vacuum 0.59 s, primary keys 0.29 s).
```
На этом всё готово к первому тесту.

## Тестирование производительности кластера
Выполним первый нагрузочный тест, чтобы оценить производительность кластера с параметрами по умолчанию (8 клиентов, общая длительность теста - 60 секунд, вывод прогресса раз в 6 секунд):
```bash
a-vasenev@pg-dba-vm:~$ pgbench -h localhost -p 5433 -U postgres -c 8 -P 6 -T 60 postgres
Password:
pgbench (14.11 (Ubuntu 14.11-0ubuntu0.22.04.1), server 15.6 (Debian 15.6-1.pgdg120+2))
starting vacuum...end.
progress: 6.0 s, 735.0 tps, lat 10.746 ms stddev 6.022
progress: 12.0 s, 771.7 tps, lat 10.367 ms stddev 5.169
progress: 18.0 s, 779.0 tps, lat 10.270 ms stddev 5.168
progress: 24.0 s, 676.5 tps, lat 11.820 ms stddev 8.385
progress: 30.0 s, 768.8 tps, lat 10.405 ms stddev 5.200
progress: 36.0 s, 714.7 tps, lat 11.195 ms stddev 17.552
progress: 42.0 s, 771.0 tps, lat 10.372 ms stddev 5.106
progress: 48.0 s, 754.0 tps, lat 10.612 ms stddev 5.074
progress: 54.0 s, 647.2 tps, lat 12.360 ms stddev 8.934
progress: 60.0 s, 746.0 tps, lat 10.728 ms stddev 5.664
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 60 s
number of transactions actually processed: 44191
latency average = 10.850 ms
latency stddev = 8.029 ms
initial connection time = 69.689 ms
tps = 737.183179 (without initial connection time)
```
Видно, что с настройками по умолчанию кластер смог выполнять в среднем 737 транзакций в секунду, выполнение каждой из них длилось в среднем 10.85 мс.

Теперь попробуем поменять параметры кластера и проверим, как это скажется на производительности. Но перед этим было бы интересно сравнить текущие параметры кластера с параметрами, прикреплёнными к домашнему заданию, чтобы знать, что именно предлагается изменить. 🙂 Прочитаем текущие значения параметров кластера, которые предлагается изменить:
```
a-vasenev@pg-dba-vm:~$ psql -h localhost -p 5433 -U postgres
Password for user postgres:
psql (14.11 (Ubuntu 14.11-0ubuntu0.22.04.1), server 15.6 (Debian 15.6-1.pgdg120+2))
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
Type "help" for help.

postgres=# select name, setting, unit from pg_settings where name in ('max_connections', 'shared_buffers', 'effective_cache_size', 'maintenance_work_mem', 'checkpoint_completion_target', 'wal_buffers', 'default_statistics_target', 'random_page_cost', 'effective_io_concurrency', 'work_mem', 'min_wal_size', 'max_wal_size');
             name             | setting | unit
------------------------------+---------+------
 checkpoint_completion_target | 0.9     |
 default_statistics_target    | 100     |
 effective_cache_size         | 524288  | 8kB
 effective_io_concurrency     | 1       |
 maintenance_work_mem         | 65536   | kB
 max_connections              | 100     |
 max_wal_size                 | 1024    | MB
 min_wal_size                 | 80      | MB
 random_page_cost             | 4       |
 shared_buffers               | 16384   | 8kB
 wal_buffers                  | 512     | 8kB
 work_mem                     | 4096    | kB
(12 rows)

```
Ниже приведено сравнение значений параметров:

| Название параметра             | Значение по умолчанию | Предлагаемое значение | 
| ------------------------------ | --------------------- | --------------------- |
| `max_connections`              | 100                   | 40                    |
| `shared_buffers`               | 16384 * 8kB = 128MB   | 1GB                   |
| `effective_cache_size`         | 524288 * 8kB = 4GB    | 3GB                   |
| `maintenance_work_mem`         | 65536kB = 64MB        | 512MB                 |
| `checkpoint_completion_target` | 0.9                   | 0.9                   |
| `wal_buffers`                  | 512 * 8kB = 4MB       | 16MB                  |
| `default_statistics_target`    | 100                   | 500                   |
| `random_page_cost`             | 4                     | 4                     |
| `effective_io_concurrency`     | 1                     | 2                     |
| `work_mem`                     | 4096kB                | 6553kB                |
| `min_wal_size`                 | 80MB                  | 4GB                   |
| `max_wal_size`                 | 1024MB = 1GB          | 16GB                  |

> **Примечание**: Возможно, в значении параметра `work_mem` в файле из задания опечатка - вероятно, имелось в виду `65536kB` (ровно 64 МБ), а не `6553kB` (что-то около 6.4 МБ). Для чистоты эксперимента значение из файла, прикреплённого к заданию, использовалось как есть.

Как видно, нам предлагается:
- Снизить значение `max_connections`. В целом разумно - в нагрузочном тесте участвует всего 8 клиентов, поэтому даже значение 40 выбрано с большим запасом).
- Увеличить значение `effective_io_concurrency` - дать знать кластеру, что дисковая подсистема сервера может обрабатывать не 1, а 2 запроса параллельно. На моей тестовой машине используется HDD, поэтому предложение спорное, но, учитывая рекомендацию использовать SSD - наверное, объяснимое и разумное.
- Существенно увеличить значение `shared_buffers` - со 128 МБ до 1 ГБ. В целом значение согласуется с [рекомендациями из документации](https://www.postgresql.org/docs/15/runtime-config-resource.html#GUC-SHARED-BUFFERS) (25% от объёма ОЗУ сервера: 4 ГБ * 0.25 = 1 ГБ), поэтому предложение звучит разумно.
- Существенно увеличить значение `maintenance_work_mem` (в 8 раз - с 64 до 512 МБ). Судя по документации, это значение напрямую влияет на служебные операции (создание индексов, добавление FK, ручной запуск `vacuum`...) и, в том числе, косвенно влияет на выделение памяти при выполнении `autovacuum` (СУБД будет потреблять до `autovacuum_max_workers` * `maintenance_work_mem` памяти - в нашем случае 3 * 512 МБ = 1.5 ГБ). Наверное, звучит разумно.
- Немного увеличить значение параметра `work_mem` (с 4 МБ до 6.4 МБ). Почему бы и нет, учитывая что у сервера достаточно ОЗУ.
- Незначительно снизить значение `effective_cache_size` (с 4 до 3 ГБ). Судя по [документации](https://www.postgresql.org/docs/15/runtime-config-query.html#GUC-EFFECTIVE-CACHE-SIZE), этот параметр влияет на построение плана - чем оно выше, тем с большей вероятностью планировщик запросов будет использовать индексы вместо полного перебора записей таблицы. Не уверен, что это изменение оправдано, но пусть будет так.
- Увеличить значение параметра `default_statistics_target` со 100 до 500. Судя по документации, это может замедлить построение статистики, но сделать предсказания планировщика запросов более точными, поэтому, возможно, изменение разумное.
- Изменить параметры ведения журнала - увеличить значение `wal_buffers` и существенно увеличить значения `min_wal_size` и `max_wal_size`. Пока не знаю, к чему это приведёт, поэтому попробуем применить и посмотрим.

Поскольку PostgreSQL запущен в Docker-контейнере, файлы конфигурации лежат в директории с данными (её мы указали при создании контейнера - она мапится на директорию `/var/lib/postgresql-docker` машины-хоста). Добавим параметры конфигурации из прикреплённого к заданию файла туда.
```bash
a-vasenev@pg-dba-vm:~$ cat > ~/homework.conf
# DB Version: 11
# OS Type: linux
# DB Type: dw
# Total Memory (RAM): 4 GB
# CPUs num: 1
# Data Storage: hdd

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
^C
a-vasenev@pg-dba-vm:~$ sudo mkdir /var/lib/postgresql-docker/conf.d
a-vasenev@pg-dba-vm:~$ sudo mv ~/homework.conf /var/lib/postgresql-docker/conf.d/
a-vasenev@pg-dba-vm:~$ sudo chown -R lxd:docker /var/lib/postgresql-docker/conf.d/
```

Добавим созданную директорию в файл с параметрами кластера, чтобы сервер СУБД читал параметры и из неё тоже:
```bash
a-vasenev@pg-dba-vm:~$ sudo nano -w /var/lib/postgresql-docker/postgresql.conf
```
```
...

#------------------------------------------------------------------------------
# CONFIG FILE INCLUDES
#------------------------------------------------------------------------------

include_dir = 'conf.d'

...
```
Перезапустим контейнер с кластером, чтобы применить изменения:
```bash
a-vasenev@pg-dba-vm:~$ sudo docker restart pg-server-15
pg-server-15
```
Снова подключимся к нему и убедимся, что параметры действительно применились:
```
a-vasenev@pg-dba-vm:~$ psql -h localhost -p 5433 -U postgres
Password for user postgres:
psql (14.11 (Ubuntu 14.11-0ubuntu0.22.04.1), server 15.6 (Debian 15.6-1.pgdg120+2))
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
Type "help" for help.

postgres=# select name, setting, unit from pg_settings where name in ('max_connections', 'shared_buffers', 'effective_cache_size', 'maintenance_work_mem', 'checkpoint_completion_target', 'wal_buffers', 'default_statistics_target', 'random_page_cost', 'effective_io_concurrency', 'work_mem', 'min_wal_size', 'max_wal_size');
             name             | setting | unit
------------------------------+---------+------
 checkpoint_completion_target | 0.9     |
 default_statistics_target    | 100     |
 effective_cache_size         | 524288  | 8kB
 effective_io_concurrency     | 1       |
 maintenance_work_mem         | 65536   | kB
 max_connections              | 100     |
 max_wal_size                 | 1024    | MB
 min_wal_size                 | 80      | MB
 random_page_cost             | 4       |
 shared_buffers               | 16384   | 8kB
 wal_buffers                  | 512     | 8kB
 work_mem                     | 4096    | kB
(12 rows)

```