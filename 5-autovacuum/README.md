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
progress: 6.0 s, 700.8 tps, lat 11.264 ms stddev 5.811
progress: 12.0 s, 711.8 tps, lat 11.192 ms stddev 5.253
progress: 18.0 s, 681.0 tps, lat 11.793 ms stddev 6.305
progress: 24.0 s, 602.8 tps, lat 13.271 ms stddev 7.083
progress: 30.0 s, 473.5 tps, lat 16.872 ms stddev 13.911
progress: 36.0 s, 629.3 tps, lat 12.729 ms stddev 7.808
progress: 42.0 s, 602.2 tps, lat 13.222 ms stddev 21.806
progress: 48.0 s, 695.8 tps, lat 11.551 ms stddev 6.438
progress: 54.0 s, 695.0 tps, lat 11.501 ms stddev 6.250
progress: 60.0 s, 492.8 tps, lat 16.188 ms stddev 14.353
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 60 s
number of transactions actually processed: 37719
latency average = 12.711 ms
latency stddev = 10.519 ms
initial connection time = 73.559 ms
tps = 629.286355 (without initial connection time)
```
Видно, что с настройками по умолчанию кластер смог выполнять в среднем 629 транзакций в секунду, выполнение каждой из них длилось в среднем 12.71 мс.

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
```sql
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
 default_statistics_target    | 500     |
 effective_cache_size         | 393216  | 8kB
 effective_io_concurrency     | 2       |
 maintenance_work_mem         | 524288  | kB
 max_connections              | 40      |
 max_wal_size                 | 16384   | MB
 min_wal_size                 | 4096    | MB
 random_page_cost             | 4       |
 shared_buffers               | 131072  | 8kB
 wal_buffers                  | 2048    | 8kB
 work_mem                     | 6553    | kB
(12 rows)
```

Видно, что параметры применились, но как они повлияли на производительность кластера?
```bash
a-vasenev@pg-dba-vm:~$ pgbench -h localhost -p 5433 -U postgres -c 8 -P 6 -T 60 postgres
Password:
pgbench (14.11 (Ubuntu 14.11-0ubuntu0.22.04.1), server 15.6 (Debian 15.6-1.pgdg120+2))
starting vacuum...end.
progress: 6.0 s, 720.3 tps, lat 10.953 ms stddev 4.976
progress: 12.0 s, 576.7 tps, lat 13.872 ms stddev 8.284
progress: 18.0 s, 528.8 tps, lat 15.128 ms stddev 12.606
progress: 24.0 s, 662.0 tps, lat 12.085 ms stddev 6.437
progress: 30.0 s, 627.2 tps, lat 12.752 ms stddev 16.450
progress: 36.0 s, 689.7 tps, lat 11.601 ms stddev 5.539
progress: 42.0 s, 662.5 tps, lat 12.074 ms stddev 5.769
progress: 48.0 s, 404.2 tps, lat 19.797 ms stddev 15.632
progress: 54.0 s, 623.8 tps, lat 12.802 ms stddev 6.892
progress: 60.0 s, 653.3 tps, lat 12.263 ms stddev 6.382
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 60 s
number of transactions actually processed: 36899
latency average = 12.993 ms
latency stddev = 9.579 ms
initial connection time = 75.842 ms
tps = 615.620870 (without initial connection time)
```
Как видно, производительность несколько упала - с 629.2 транзакций в секунду в среднем до 615.6 транзакций в секунду (-2.2%). Объяснить это изменение я не могу, но лично я бы списал его либо на неудачные настройки для дисковой подсистемы кластера (параметр `effective_io_concurrency`), либо на погрешность измерения.

> **Примечание**: Не уверен, что этот тест прошёл по плану - возможно, изменение производительности кластера должно было быть более существенным. Возможно, причина в том, что в этом задании предлагалось использовать виртуальную машину с SSD, а я использовал HDD. Может быть, повлиял тот факт, что я использовал Docker, и его применение существенно снизило производительность. Так или иначе, не думаю, что результатам этого теста можно доверять.

## MVCC и autovacuum

Снова подключимся к кластеру и создадим тестовую таблицу, на которой будут проводиться эксперименты с размером таблицы. Заодно найдём идентификатор БД и таблицы, чтобы найти файл с её данными на диске.
```sql
a-vasenev@pg-dba-vm:~$ psql -h localhost -p 5433 -U postgres
Password for user postgres:
psql (14.11 (Ubuntu 14.11-0ubuntu0.22.04.1), server 15.6 (Debian 15.6-1.pgdg120+2))
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
Type "help" for help.

postgres=# create table test_data (value varchar(50));
CREATE TABLE
postgres=# select oid from pg_database where datname = 'postgres';
 oid
-----
   5
(1 row)

postgres=# select relfilenode, reltoastrelid from pg_class where relname = 'test_data';
 relfilenode | reltoastrelid
-------------+---------------
       16467 |             0
(1 row)
```

> **Примечание**: При создании таблицы я использовал для текстового поля тип `varchar(50)`. Таким образом, максимальный размер записи для строки оказался определён заранее и сравнительно невелик (8 байт поля `id` + до 50 символов поля `text` + небольшой оверхед на служебную информацию о строке). Поскольку он не превысил порога в 2 КБ, вся таблица размещается в одном файле и не имеет TOAST-файла (это видно по `reltoastrelid = 0`). Если бы вместо `varchar(50)` использовался тип `text`, данные таблицы хранились бы в двух файлах - собственные данные таблицы и TOAST-данные.

Итак, видно, что у БД `postgres` значение `oid` равно `5`, а у тестовой таблицы значение `relfilenode` равно `16467`. Стало быть, данные этой таблицы лежат в файле `$PGDATA/base/5/16467`.
```
a-vasenev@pg-dba-vm:~$ sudo su -
root@pg-dba-vm:~# ls -lA /var/lib/postgresql-docker/base/5/16467*
-rw------- 1 lxd docker 0 Mar 11 10:45 /var/lib/postgresql-docker/base/5/16467
```

В таблице пока нет данных, поэтому файл с данными пуст. Попробуем заполнить её строками с последовательными числами:
```sql
postgres=# insert into test_data (value) select to_char(generate_series(1, 1000000), '00000000');
INSERT 0 1000000
postgres=# select * from test_data offset 999990 limit 10;
   value
-----------
  00999991
  00999992
  00999993
  00999994
  00999995
  00999996
  00999997
  00999998
  00999999
  01000000
(10 rows)
```
```
root@pg-dba-vm:~# ls -lA /var/lib/postgresql-docker/base/5/16467*
-rw------- 1 lxd docker 44285952 Mar 11 10:54 /var/lib/postgresql-docker/base/5/16467
-rw------- 1 lxd docker    32768 Mar 11 10:50 /var/lib/postgresql-docker/base/5/16467_fsm
-rw------- 1 lxd docker     8192 Mar 11 10:50 /var/lib/postgresql-docker/base/5/16467_vm
```

Теперь файл с данными таблицы (`16467`) занимает на диске 44285952 байт (около 42.2 МБ). Такое же значение сообщает и СУБД.
```sql
postgres=# select fork, pg_relation_size('public.test_data', fork) from (values ('main'), ('fsm'), ('vm'), ('init')) as t (fork);
 fork | pg_relation_size
------+------------------
 main |         44285952
 fsm  |            32768
 vm   |             8192
 init |                0
(4 rows)
```

Теперь несколько раз обновим записи в таблице. Для этого напишем процедуру, которая 5 раз переберёт все записи в таблице и добавит к ним номер итерации.
```sql
postgres=# do $$
begin
    for i in 1..5 loop
        update test_data set value = value || i::text;
        raise notice 'Updating test_data entries: pass #% done', i;
    end loop;
end $$;
NOTICE:  Updating test_data entries: pass #1 done
NOTICE:  Updating test_data entries: pass #2 done
NOTICE:  Updating test_data entries: pass #3 done
NOTICE:  Updating test_data entries: pass #4 done
NOTICE:  Updating test_data entries: pass #5 done
DO
postgres=# select * from test_data offset 999990 limit 10;
     value
----------------
  0099999112345
  0099999212345
  0099999312345
  0099999412345
  0099999512345
  0099999612345
  0099999712345
  0099999812345
  0099999912345
  0100000012345
(10 rows)
```

Снова проверим размер таблицы. Ранее мы убедились, что СУБД и ОС возвращают одинаковый размер файлов таблицы, поэтому здесь и далее я буду получать его при помощи функций СУБД - так будет меньше переключений между окнами с сессиями SSH. 😅
```sql
postgres=# select fork, pg_relation_size('public.test_data', fork) from (values ('main'), ('fsm'), ('vm'), ('init')) as t (fork);
 fork | pg_relation_size
------+------------------
 main |        265691136
 fsm  |            81920
 vm   |             8192
 init |                0
(4 rows)

postgres=# \x
Expanded display is on.
postgres=# select * from pg_stat_all_tables where relname = 'test_data';
-[ RECORD 1 ]-------+------------------------------
relid               | 16467
schemaname          | public
relname             | test_data
seq_scan            | 7
seq_tup_read        | 7000000
idx_scan            |
idx_tup_fetch       |
n_tup_ins           | 1000000
n_tup_upd           | 5000000
n_tup_del           | 0
n_tup_hot_upd       | 0
n_live_tup          | 1000000
n_dead_tup          | 0
n_mod_since_analyze | 0
n_ins_since_vacuum  | 0
last_vacuum         |
last_autovacuum     | 2024-03-11 11:19:00.473734+00
last_analyze        |
last_autoanalyze    | 2024-03-11 11:19:01.201097+00
vacuum_count        | 0
autovacuum_count    | 2
analyze_count       | 0
autoanalyze_count   | 2
```

Как видно, файл с данными таблицы увеличился почти ровно в 6 раз. В целом это объяснимо: изначально в таблице содержался 1 млн строк, и дальше в ходе работы процедуры каждая строка была обновлена 5 раз. Каждое обновление привело к созданию новой версии кортежей (tuples) каждой строки и выделению места под эти кортежи в файле с данными таблицами. Далее сработал `autovacuum`, который пометил это место как свободное, но не вернул его операционной системе - как результат, файл с данными таблицы по-прежнему имеет размер в 6 раз больше исходного размера.

Повторим эксперимент:
```sql
postgres=# do $$
begin
    for i in 1..5 loop
        update test_data set value = value || i::text;
        raise notice 'Updating test_data entries: pass #% done', i;
    end loop;
end $$;
NOTICE:  Updating test_data entries: pass #1 done
NOTICE:  Updating test_data entries: pass #2 done
NOTICE:  Updating test_data entries: pass #3 done
NOTICE:  Updating test_data entries: pass #4 done
NOTICE:  Updating test_data entries: pass #5 done
DO
postgres=# select * from pg_stat_all_tables where relname = 'test_data';
-[ RECORD 1 ]-------+------------------------------
relid               | 16467
schemaname          | public
relname             | test_data
seq_scan            | 12
seq_tup_read        | 12000000
idx_scan            |
idx_tup_fetch       |
n_tup_ins           | 1000000
n_tup_upd           | 10000000
n_tup_del           | 0
n_tup_hot_upd       | 106
n_live_tup          | 1000000
n_dead_tup          | 5000000
n_mod_since_analyze | 5000000
n_ins_since_vacuum  | 0
last_vacuum         |
last_autovacuum     | 2024-03-11 11:19:00.473734+00
last_analyze        |
last_autoanalyze    | 2024-03-11 11:19:01.201097+00
vacuum_count        | 0
autovacuum_count    | 2
analyze_count       | 0
autoanalyze_count   | 2
```

На этот раз я успел поймать момент перед запуском autovacuum. Видно, что в таблице находится 5 млн мёртвых кортежей: первый апдейт привёл к созданию новой версии и пометил предыдущую активную версию как мёртвую, второй апдейт сделал то же самое с версией строк от первого апдейта, и так до пятого апдейта.

Подождём немного и попробуем запросить состояние таблицы ещё раз:
```sql
postgres=# select * from pg_stat_all_tables where relname = 'test_data';
-[ RECORD 1 ]-------+------------------------------
relid               | 16467
schemaname          | public
relname             | test_data
seq_scan            | 14
seq_tup_read        | 14000000
idx_scan            |
idx_tup_fetch       |
n_tup_ins           | 1000000
n_tup_upd           | 10000000
n_tup_del           | 0
n_tup_hot_upd       | 106
n_live_tup          | 1000000
n_dead_tup          | 0
n_mod_since_analyze | 0
n_ins_since_vacuum  | 0
last_vacuum         |
last_autovacuum     | 2024-03-11 12:12:02.002864+00
last_analyze        |
last_autoanalyze    | 2024-03-11 12:12:03.032957+00
vacuum_count        | 0
autovacuum_count    | 3
analyze_count       | 0
autoanalyze_count   | 3
```
Теперь `autovacuum` (видно по изменившейся дате в поле `last_autovacuum`), и мёртвых кортежей больше не осталось.

Проверим, что записи действительно обновились, и ещё раз посмотрим на размер файла с данными таблицы:
```sql
postgres=# select * from test_data offset 999990 limit 10;
        value
---------------------
  009974031234512345
  009974041234512345
  009974051234512345
  009974061234512345
  009974071234512345
  009974081234512345
  009974091234512345
  009974101234512345
  009974111234512345
  009974121234512345
(10 rows)

postgres=# select fork, pg_relation_size('public.test_data', fork) from (values ('main'), ('fsm'), ('vm'), ('init')) as t (fork);
 fork | pg_relation_size
------+------------------
 main |        297279488
 fsm  |            90112
 vm   |            16384
 init |                0
(4 rows)
```
Размер таблицы снова увеличился, но прирост совсем не такой существенный, каким был в первый раз - тогда размер таблицы увеличился на 500%, теперь же всего лишь на 11.8%. Вероятно, из-за того, что длина строкового поля увеличилась, СУБД не хватило места, выделенного под кортежи ранее и освобождённого предыдущим запуском `autovacuum`, и она была вынуждена выделить ещё немного места.

> **Примечание**: К слову, в выводе предыдущего запроса можно наглядно наблюдать важность явной сортировки в запросах, если бизнес-логика приложения-клиента БД её требует. 🙂 При создании таблицы `test_data` записи в таблицу вставлялись по возрастанию, и запрос последних десяти записей без сортировки возвращал записи с номерами `00999990` до `01000000`. Однако, видимо, в процессе жонглирования записями в файле таблицы при втором массовом обновлении записей *или* при запуске `autovacuum` после этого обновления в структуре таблицы что-то изменилось, и теперь запрос последних 10 строк возвращает практически случайный блок записей - возвращённые записи не постоянны, даже если запросы выполняются друг за другом:
> ```sql
> postgres=# select * from test_data offset 999990 limit 10;
>        value
> ---------------------
>   009898671234512345
>   009898681234512345
>   009898691234512345
>   009898701234512345
>   009898711234512345
>   009898721234512345
>   009898731234512345
>   009898741234512345
>   009898751234512345
>   009898761234512345
> (10 rows)
> 
> postgres=# select * from test_data offset 999990 limit 10;
>         value
> ---------------------
>   009873551234512345
>   009873561234512345
>   009873571234512345
>   009873581234512345
>   009873591234512345
>   009873601234512345
>   009873611234512345
>   009873621234512345
>   009873631234512345
>   009873641234512345
> (10 rows)
> ```
> При этом сами записи таблицы на месте - если запросить последние 10 записей с явной сортировкой, то снова вернутся записи от `00999990` до `01000000`:
> ```sql
> postgres=# select * from test_data order by value asc offset 999990 limit 10;
>         value
> ---------------------
>   009999911234512345
>   009999921234512345
>   009999931234512345
>   009999941234512345
>   009999951234512345
>   009999961234512345
>   009999971234512345
>   009999981234512345
>   009999991234512345
>   010000001234512345
> (10 rows)
> ```
> **Мораль**: если порядок следования записей (или хотя бы постоянство порядка) важны - например, при постраничном отображении списка элементов таблицы пользователям программы или при пагинации по записям таблицы для их пакетной обработки - **ни в коем случае** нельзя полагаться на неявную сортировку записей самой СУБД! Вместо этого в запросе нужно явно указывать сортировку по какому-нибудь полю (по возможности индексированному), даже если это какое-то тривиальное поле вроде первичного ключа.

Теперь отключим для тестовой таблицы `autovacuum` и повторим эксперимент:
```sql
postgres=# do $$
begin
    for i in 1..10 loop
        update test_data set value = value || i::text;
        raise notice 'Updating test_data entries: pass #% done', i;
    end loop;
end $$;
NOTICE:  Updating test_data entries: pass #1 done
NOTICE:  Updating test_data entries: pass #2 done
NOTICE:  Updating test_data entries: pass #3 done
NOTICE:  Updating test_data entries: pass #4 done
NOTICE:  Updating test_data entries: pass #5 done
NOTICE:  Updating test_data entries: pass #6 done
NOTICE:  Updating test_data entries: pass #7 done
NOTICE:  Updating test_data entries: pass #8 done
NOTICE:  Updating test_data entries: pass #9 done
NOTICE:  Updating test_data entries: pass #10 done
DO
postgres=# select fork, pg_relation_size('public.test_data', fork) from (values ('main'), ('fsm'), ('vm'), ('init')) as t (fork);
 fork | pg_relation_size
------+------------------
 main |        674480128
 fsm  |           188416
 vm   |            16384
 init |                0
(4 rows)

postgres=# \x
Expanded display is on.
postgres=# select * from pg_stat_all_tables where relname = 'test_data';
-[ RECORD 1 ]-------+------------------------------
relid               | 16467
schemaname          | public
relname             | test_data
seq_scan            | 32
seq_tup_read        | 31000000
idx_scan            |
idx_tup_fetch       |
n_tup_ins           | 1000000
n_tup_upd           | 21000000
n_tup_del           | 0
n_tup_hot_upd       | 286
n_live_tup          | 1000000
n_dead_tup          | 10999910
n_mod_since_analyze | 10000000
n_ins_since_vacuum  | 0
last_vacuum         |
last_autovacuum     | 2024-03-11 12:12:02.002864+00
last_analyze        |
last_autoanalyze    | 2024-03-11 12:12:03.032957+00
vacuum_count        | 0
autovacuum_count    | 3
analyze_count       | 0
autoanalyze_count   | 3
```
Таблица увеличилась ещё в два с лишним раза - до 643 МБ, так как её содержимое снова было обновлено, и механизм MVCC снова создал новые версии записей. Однако теперь, поскольку `autovacuum` выключен, мёртвые записи так и останутся мёртвыми, и разрешить повторное использование занятого ими места теперь можно только ручным запуском `vacuum`. Либо, как вариант, можно снова включить `autovacuum` и переложить эту работу на СУБД. 🙂
```sql
postgres=# alter table test_data set (autovacuum_enabled = true);
ALTER TABLE
```