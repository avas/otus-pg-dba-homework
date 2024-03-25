# Работа с журналами

## Подготовка окружения

Для выполнения домашнего задания будет использоваться виртуальная машина, созданная в рамках одной из предыдущих домашних работ:
- **CPU**: Intel Ice Lake, 2 ядра, гарантированная доля - 100%
- **RAM**: 4 ГБ
- **Диск**: HDD, 100 ГБ
- **ОС**: Ubuntu 22.04 LTS

В ходе подготовки к какой-то из предыдущих домашних работ на эту машину был локально установлен PostgreSQL 14 - его мы и будем использовать для выполнения этой домашней работы.

По умолчанию контрольные точки выполняются раз в 5 минут - поменяем конфигурацию кластера, чтобы изменить это время на 30 секунд. Заодно включим логирование контрольных точек - это поможет нам понять, насколько точно этот интервал соблюдался.
```bash
a-vasenev@pg-dba-vm:~$ sudo nano -w /etc/postgresql/14/main/postgresql.conf
```
```
...

#------------------------------------------------------------------------------
# WRITE-AHEAD LOG
#------------------------------------------------------------------------------

checkpoint_timeout = 30s                # range 30s-1d

...

#------------------------------------------------------------------------------
# REPORTING AND LOGGING
#------------------------------------------------------------------------------

...

# - What to Log -

log_checkpoints = on # значение было изменено, по умолчанию off

...
```

Перезапустим кластер для применения изменений:
```bash
a-vasenev@pg-dba-vm:~$ sudo pg_ctlcluster 14 main restart
a-vasenev@pg-dba-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```

Поскольку в ходе этого задания будет использоваться утилита `pgbench`, убедимся, что БД проинициализирована для работы с ней (чтобы не заморачиваться, будем использовать БД по умолчанию - `postgres`).
```bash
a-vasenev@pg-dba-vm:~$ pgbench -h localhost -p 5432 -U postgres -i postgres
Password:
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.03 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 4.96 s (drop tables 1.42 s, create tables 0.92 s, client-side generate 0.47 s, vacuum 1.34 s, primary keys 0.82 s).
```
Теперь всё готово для выполнения задания.

## Write Ahead Log (WAL) и контрольные точки

Подключимся к кластеру и посмотрим текущую статистику процесса `bgwriter`. Заодно посмотрим на текущее положение в буфере WAL.
```sql
a-vasenev@pg-dba-vm:~$ psql -h localhost -p 5432 -U postgres
Password for user postgres:
psql (14.11 (Ubuntu 14.11-0ubuntu0.22.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# select * from pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 5711
checkpoints_req       | 4
checkpoint_write_time | 1183525
checkpoint_sync_time  | 863
buffers_checkpoint    | 88161
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 8280
buffers_backend_fsync | 0
buffers_alloc         | 11549
stats_reset           | 2024-03-20 12:29:03.175493+00

postgres=# select pg_current_wal_insert_lsn(), pg_current_wal_lsn(), pg_current_wal_flush_lsn();
 pg_current_wal_insert_lsn | pg_current_wal_lsn | pg_current_wal_flush_lsn
---------------------------+--------------------+--------------------------
 0/3BF07010                | 0/3BF07010         | 0/3BF07010
(1 row)
```
По выводу второго запроса видно, что положение вставки в WAL (`pg_current_wal_insert_lsn()`), положение последней записи, отданной для записи ОС (`pg_current_wal_lsn()`) и положение последней записи, которая точно была сохранена на диск (`pg_current_wal_flush_lsn()`), совпадают. Таким образом, все данные были сохранены в долгосрочное хранилище, и несохранённых данных нет (что в общем-то объяснимо - кластер используется только для выполнения домашних заданий, и в данный момент на него нет совершенно никакой нагрузки).

Теперь запустим нагрузочный тест:
```bash
a-vasenev@pg-dba-vm:~$ pgbench -h localhost -p 5432 -U postgres -c 8 -P 30 -T 600 postgres
Password:
pgbench (14.11 (Ubuntu 14.11-0ubuntu0.22.04.1))
starting vacuum...end.
progress: 30.0 s, 458.7 tps, lat 17.375 ms stddev 12.271
progress: 60.0 s, 556.7 tps, lat 14.370 ms stddev 9.858
progress: 90.0 s, 631.5 tps, lat 12.669 ms stddev 8.166
progress: 120.0 s, 620.3 tps, lat 12.895 ms stddev 8.193
progress: 150.0 s, 599.3 tps, lat 13.350 ms stddev 8.562
progress: 180.0 s, 607.3 tps, lat 13.171 ms stddev 8.222
progress: 210.0 s, 558.4 tps, lat 14.327 ms stddev 10.462
progress: 240.0 s, 616.6 tps, lat 12.974 ms stddev 7.985
progress: 270.0 s, 615.0 tps, lat 13.007 ms stddev 8.238
progress: 300.0 s, 639.6 tps, lat 12.504 ms stddev 7.556
progress: 330.0 s, 609.0 tps, lat 13.136 ms stddev 8.454
progress: 360.0 s, 565.7 tps, lat 14.140 ms stddev 8.850
progress: 390.0 s, 603.6 tps, lat 13.253 ms stddev 8.283
progress: 420.0 s, 623.0 tps, lat 12.842 ms stddev 8.204
progress: 450.0 s, 612.6 tps, lat 13.059 ms stddev 8.188
progress: 480.0 s, 610.1 tps, lat 13.111 ms stddev 7.829
progress: 510.0 s, 552.9 tps, lat 14.468 ms stddev 10.574
progress: 540.0 s, 635.4 tps, lat 12.590 ms stddev 7.561
progress: 570.0 s, 593.4 tps, lat 13.479 ms stddev 8.885
progress: 600.0 s, 614.2 tps, lat 13.025 ms stddev 8.067
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 357712
latency average = 13.416 ms
latency stddev = 8.767 ms
initial connection time = 98.625 ms
tps = 596.268092 (without initial connection time)
```

Сразу после завершения теста прочитаем статистику `bgwriter` и положение в буфере WAL повторно:
```sql
postgres=# select * from pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 5731
checkpoints_req       | 4
checkpoint_write_time | 1722244
checkpoint_sync_time  | 1183
buffers_checkpoint    | 126527
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 10586
buffers_backend_fsync | 0
buffers_alloc         | 16075
stats_reset           | 2024-03-20 12:29:03.175493+00

postgres=# select pg_current_wal_insert_lsn(), pg_current_wal_lsn(), pg_current_wal_flush_lsn();
 pg_current_wal_insert_lsn | pg_current_wal_lsn | pg_current_wal_flush_lsn
---------------------------+--------------------+--------------------------
 0/57A30538                | 0/57A30538         | 0/57A30538
(1 row)
```
Видно, что в прошлом выводе содержимого `pg_stat_bgwriter` столбец `checkpoints_timed` имел значение 5711, а в текущем - 5731. Стало быть, с прошлого раза прошло ровно 20 контрольных точек, что сходится с ожидаемым значением (600 секунд теста / 30 секунд между контрольными точками = 20 контрольных точек).

По выводу второго запроса также видно, что все данные были записаны на диск (значения `pg_current_wal_insert_lsn()`, `pg_current_wal_lsn()` и `pg_current_wal_flush_lsn()` совпадают). Если сравнить значение с предыдущим значением (до начала теста), то можно получить разницу между положениями в буфере - это и будет объёмом сгенерированных в ходе теста данных в журнале.
```sql
postgres=# select pg_wal_lsn_diff('0/57A30538', '0/38F07010');
 pg_wal_lsn_diff
-----------------
       515020072
(1 row)

postgres=# select pg_size_pretty(pg_wal_lsn_diff('0/57A30538', '0/38F07010'));
 pg_size_pretty
----------------
 491 MB
(1 row)
```
Зная этот объём, можно примерно оценить объём данных, который в среднем пришёлся на одну контрольную точку:
515020072 байт / 20 контрольных точек = 25751003 байт (около 24.55 МБ, или около 3143.5 страниц буфера размером по 8 КБ).

Посмотрим на то, с какой периодичностью выполнялись контрольные точки:
```bash
a-vasenev@pg-dba-vm:~$ sudo nano -w /var/log/postgresql/postgresql-14-main.log
```
```
...

2024-03-25 11:01:50.016 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:02:17.069 UTC [1810] LOG:  checkpoint complete: wrote 1306 buffers (8.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.972 s, sync=0.020>
2024-03-25 11:02:20.072 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:02:47.090 UTC [1810] LOG:  checkpoint complete: wrote 1849 buffers (11.3%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.975 s, sync=0.01>
2024-03-25 11:02:50.092 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:03:17.138 UTC [1810] LOG:  checkpoint complete: wrote 1915 buffers (11.7%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.977 s, sync=0.01>
2024-03-25 11:03:20.142 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:03:47.082 UTC [1810] LOG:  checkpoint complete: wrote 1989 buffers (12.1%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.884 s, sync=0.01>
2024-03-25 11:03:50.084 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:04:17.039 UTC [1810] LOG:  checkpoint complete: wrote 1902 buffers (11.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.884 s, sync=0.02>
2024-03-25 11:04:20.042 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:04:47.111 UTC [1810] LOG:  checkpoint complete: wrote 2001 buffers (12.2%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.991 s, sync=0.01>
2024-03-25 11:04:50.112 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:05:17.071 UTC [1810] LOG:  checkpoint complete: wrote 1920 buffers (11.7%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.882 s, sync=0.01>
2024-03-25 11:05:20.074 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:05:47.102 UTC [1810] LOG:  checkpoint complete: wrote 1980 buffers (12.1%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.973 s, sync=0.01>
2024-03-25 11:05:50.104 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:06:17.066 UTC [1810] LOG:  checkpoint complete: wrote 1889 buffers (11.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.881 s, sync=0.02>
2024-03-25 11:06:20.068 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:06:47.091 UTC [1810] LOG:  checkpoint complete: wrote 2003 buffers (12.2%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.975 s, sync=0.01>
2024-03-25 11:06:50.092 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:07:17.059 UTC [1810] LOG:  checkpoint complete: wrote 1890 buffers (11.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.899 s, sync=0.01>
2024-03-25 11:07:20.062 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:07:47.097 UTC [1810] LOG:  checkpoint complete: wrote 2003 buffers (12.2%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.972 s, sync=0.01>
2024-03-25 11:07:50.100 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:08:17.075 UTC [1810] LOG:  checkpoint complete: wrote 1874 buffers (11.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.877 s, sync=0.01>
2024-03-25 11:08:20.078 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:08:47.131 UTC [1810] LOG:  checkpoint complete: wrote 2023 buffers (12.3%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.982 s, sync=0.01>
2024-03-25 11:08:50.132 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:09:17.083 UTC [1810] LOG:  checkpoint complete: wrote 1897 buffers (11.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.871 s, sync=0.01>
2024-03-25 11:09:20.086 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:09:47.114 UTC [1810] LOG:  checkpoint complete: wrote 1996 buffers (12.2%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.972 s, sync=0.01>
2024-03-25 11:09:50.116 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:10:17.067 UTC [1810] LOG:  checkpoint complete: wrote 1878 buffers (11.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.885 s, sync=0.02>
2024-03-25 11:10:20.070 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:10:47.110 UTC [1810] LOG:  checkpoint complete: wrote 2227 buffers (13.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.991 s, sync=0.01>
2024-03-25 11:10:50.112 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:11:17.066 UTC [1810] LOG:  checkpoint complete: wrote 1901 buffers (11.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.895 s, sync=0.02>
2024-03-25 11:11:20.068 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:11:47.103 UTC [1810] LOG:  checkpoint complete: wrote 1992 buffers (12.2%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.981 s, sync=0.01>
2024-03-25 11:12:20.136 UTC [1810] LOG:  checkpoint starting: time
2024-03-25 11:12:47.024 UTC [1810] LOG:  checkpoint complete: wrote 2273 buffers (13.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.873 s, sync=0.00>
```
Видно, что контрольные точки выполнялись по расписанию - интервал между ними составляет ровно 30 секунд (по крайней мере с точностью до 0.1 с), и задержек не было.

> **Примечание**: ранее мы рассчитали примерное расчётное количество данных, которое было записано за одну контрольную точку - 3143.5 страниц по 8 КБ. Однако в логе мы видим фактическое количество данных, сохраняемых в ходе каждой контрольной точки, и оно значительно меньше - порядка 1800-2200 страниц. Расхождение довольно существенное (примерно в полтора раза), и объяснения ему у меня, к сожалению, нет.

## Замер производительности в асинхронном режиме

По умолчанию кластер работает в синхронном режиме - при коммите транзакции изменения отправляются на запись сразу же. Посмотрим, что изменится при переходе в асинхронный режим. Для этого переведём кластер в асинхронный режим:
```sql
postgres=# alter system set synchronous_commit to off;
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# select * from pg_settings where name = 'synchronous_commit' \gx
-[ RECORD 1 ]---+------------------------------------------------------
name            | synchronous_commit
setting         | off
unit            |
category        | Write-Ahead Log / Settings
short_desc      | Sets the current transaction's synchronization level.
extra_desc      |
context         | user
vartype         | enum
source          | configuration file
min_val         |
max_val         |
enumvals        | {local,remote_write,remote_apply,on,off}
boot_val        | on
reset_val       | off
sourcefile      | /var/lib/postgresql/14/main/postgresql.auto.conf
sourceline      | 3
pending_restart | f
```

Повторим тест производительности с теми же параметрами:
```bash
a-vasenev@pg-dba-vm:~$ pgbench -h localhost -p 5432 -U postgres -c 8 -P 30 -T 600 postgres
Password:
pgbench (14.11 (Ubuntu 14.11-0ubuntu0.22.04.1))
starting vacuum...end.
progress: 30.0 s, 2188.4 tps, lat 3.642 ms stddev 1.002
progress: 60.0 s, 2166.9 tps, lat 3.691 ms stddev 1.045
progress: 90.0 s, 2165.8 tps, lat 3.693 ms stddev 1.553
progress: 120.0 s, 2168.9 tps, lat 3.688 ms stddev 1.010
progress: 150.0 s, 2180.4 tps, lat 3.668 ms stddev 1.878
progress: 180.0 s, 2178.2 tps, lat 3.672 ms stddev 1.025
progress: 210.0 s, 2195.6 tps, lat 3.643 ms stddev 2.629
progress: 240.0 s, 2198.4 tps, lat 3.638 ms stddev 1.012
progress: 270.0 s, 2189.5 tps, lat 3.653 ms stddev 1.000
progress: 300.0 s, 2185.5 tps, lat 3.659 ms stddev 1.019
progress: 330.0 s, 2188.6 tps, lat 3.654 ms stddev 0.997
progress: 360.0 s, 2186.7 tps, lat 3.658 ms stddev 1.017
progress: 390.0 s, 2209.7 tps, lat 3.619 ms stddev 0.974
progress: 420.0 s, 2244.2 tps, lat 3.564 ms stddev 1.021
progress: 450.0 s, 2264.2 tps, lat 3.532 ms stddev 0.969
progress: 480.0 s, 2246.1 tps, lat 3.561 ms stddev 0.973
progress: 510.0 s, 2274.4 tps, lat 3.516 ms stddev 0.967
progress: 540.0 s, 2272.0 tps, lat 3.520 ms stddev 0.975
progress: 570.0 s, 2255.5 tps, lat 3.546 ms stddev 0.954
progress: 600.0 s, 2226.0 tps, lat 3.593 ms stddev 1.132
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 1325561
latency average = 3.620 ms
latency stddev = 1.225 ms
initial connection time = 99.304 ms
tps = 2209.581144 (without initial connection time)
```

Видно, что производительность выросла, и выросла заметно - с 596.2 до 2209.5 транзакций в секунду (в 3.7 раза). Однако, к сожалению, за такой прирост придётся заплатить долговечностью данных. При синхронной записи изменения отправляются на диск прямо во время коммита транзакции, вне зависимости от того, сколько их там было, поэтому коммит транзакции гарантирует, что данные будут сохранены. При асинхронной же записи на диск отправляются только полные страницы буфера - страницы, заполненные не до конца, могут быть записаны с некоторой задержкой (когда `bgwriter` заметит, что с момента прошлой записи не появилось новых полных страниц буфера). Поэтому, если изменения, сделанные какой-то транзакцией, попадут в буфер, но перед сохранением на диск страницы буфера с этими изменениями сервер по какой-то причине будет неожиданно остановлен, то данные этой транзакции будут потеряны (даже несмотря на то, что транзакция была зафиксирована). Таким образом, к использованию асинхронного режима нужно подходить с осторожностью и тщательно взвешивать риски.

> **Интересная деталь**: Было интересно сравнить значения `pg_current_wal_insert_lsn()`, `pg_current_wal_ls()` и `pg_current_wal_flush_lsn()` во время нагрузочного теста в синхронном и асинхронном режиме. В синхронном режиме позиция последней записанной записи (`pg_current_wal_lsn()`) и позиция записи, которая была реально сохранена на диск (`pg_current_wal_flush_lsn()`), совпадают даже под нагрузкой:
> ```sql
> postgres=# select pg_current_wal_insert_lsn(), pg_current_wal_lsn(), pg_current_wal_flush_lsn();
>  pg_current_wal_insert_lsn | pg_current_wal_lsn | pg_current_wal_flush_lsn
> ---------------------------+--------------------+--------------------------
>  0/50CCCC18                | 0/50CCCAD8         | 0/50CCCAD8
> (1 row)
> ```
> 
> В асинхронном же режиме их значения могут отличаться:
> ```sql
> postgres=# select pg_current_wal_insert_lsn(), pg_current_wal_lsn(), pg_current_wal_flush_lsn();
>  pg_current_wal_insert_lsn | pg_current_wal_lsn | pg_current_wal_flush_lsn
> ---------------------------+--------------------+--------------------------
>  0/7E89B488                | 0/7E89A000         | 0/7E894000
> (1 row)
> ```
>
> Так можно наглядно увидеть разницу между этими режимами работы - становится понятно, откуда берётся такая разница в производительности. 🙂

## Повреждение данных таблицы

Теперь попробуем сымитировать повреждение файла с данными таблицы и посмотрим, что при этом произойдёт. Для этого создадим новый кластер (назовём его `checksum`):
```bash
a-vasenev@pg-dba-vm:~$ sudo pg_createcluster -p 5433 --start 14 checksum -- --data-checksums
Creating new PostgreSQL cluster 14/checksum ...
/usr/lib/postgresql/14/bin/initdb -D /var/lib/postgresql/14/checksum --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/14/checksum ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Ver Cluster  Port Status Owner    Data directory                  Log file
14  checksum 5433 online postgres /var/lib/postgresql/14/checksum /var/log/postgresql/postgresql-14-checksum.log
```
Чтобы к этому кластеру было проще подключаться, зададим пароль для пользователя СУБД `postgres`:
```sql
a-vasenev@pg-dba-vm:~$ sudo -u postgres psql -p 5433
could not change directory to "/home/a-vasenev": Permission denied
psql (14.11 (Ubuntu 14.11-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# alter user postgres with password 'postgres';
ALTER ROLE
postgres=# exit
```

Теперь подключимся к этому кластеру к БД `postgres`, создадим в ней какую-нибудь таблицу и заполним её какими-нибудь данными:
```sql
a-vasenev@pg-dba-vm:~$ psql -h localhost -p 5433 -U postgres
Password for user postgres:
psql (14.11 (Ubuntu 14.11-0ubuntu0.22.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# create table test_data (test_value integer not null);
CREATE TABLE
postgres=# insert into test_data (test_value) select generate_series(1, 1000);
INSERT 0 1000
postgres=# select * from test_data limit 10;
 test_value
------------
          1
          2
          3
          4
          5
          6
          7
          8
          9
         10
(10 rows)
```

Попробуем испортить данные в этой таблице. Для этого посмотрим, где лежит файл с её данными:
```sql
postgres=# select pg_relation_filepath('test_data');
 pg_relation_filepath
----------------------
 base/13761/16384
(1 row)
```

Проверим, действительно ли он там есть:
```bash
a-vasenev@pg-dba-vm:~$ sudo su - postgres
postgres@pg-dba-vm:~$ ls -lA /var/lib/postgresql/14/checksum/base/13761/16384*
-rw------- 1 postgres postgres 40960 Mar 25 13:01 /var/lib/postgresql/14/checksum/base/13761/16384
-rw------- 1 postgres postgres 24576 Mar 25 12:59 /var/lib/postgresql/14/checksum/base/13761/16384_fsm
```

Как видно, файл действительно существует, и данные таблицы занимают 40 КБ. Остановим кластер и заменим немного данных в начале этого файла - например, 128 байт - на случайный мусор (его можно прочитать из виртуального устройства `/dev/urandom`):
```bash
a-vasenev@pg-dba-vm:~$ sudo dd if=/dev/urandom of=/var/lib/postgresql/14/checksum/base/13761/16384 bs=128 count=1 conv=notrunc
1+0 records in
1+0 records out
128 bytes copied, 0.000105754 s, 1.2 MB/s
```
Попробуем запустить кластер обратно и посмотрим, что получится:
```
a-vasenev@pg-dba-vm:~$ sudo pg_ctlcluster 14 checksum start
a-vasenev@pg-dba-vm:~$ pg_lcclusters
Command 'pg_lcclusters' not found, did you mean:
  command 'pg_lsclusters' from deb postgresql-common (238)
a-vasenev@pg-dba-vm:~$ pg_lsclusters
Ver Cluster  Port Status Owner    Data directory                  Log file
14  checksum 5433 online postgres /var/lib/postgresql/14/checksum /var/log/postgresql/postgresql-14-checksum.log
14  main     5432 down   postgres /var/lib/postgresql/14/main     /var/log/postgresql/postgresql-14-main.log
```

Кластер `14/checksum` запустился без ошибок, в его логе никаких ошибок тоже нет. Но что будет, если попробовать прочитать данные из таблицы, файл которой мы повредили?
```sql
postgres=# select * from test_data limit 10;
WARNING:  page verification failed, calculated checksum 50372 but expected 26896
ERROR:  invalid page in block 0 of relation base/13761/16384
```
Итак, сервер обнаружил повреждение данных. Теперь любые попытки работы с этой таблицей будут приводить к прерыванию транзакции, которая попыталась к ней обратиться, и записи ошибок в лог.

> **Примечание**: В теории, PostgreSQL позволяет [обходить проверку контрольных сумм](https://www.postgresql.org/docs/current/runtime-config-developer.html#GUC-IGNORE-CHECKSUM-FAILURE) и пытаться обработать запрос несмотря ни на что. Но у меня так и не получилось заставить это работать - что бы я ни делал, сервер продолжает возвращать ошибку при попытке чтения из таблицы:
> ```sql
> postgres=# show ignore_checksum_failure;
>  ignore_checksum_failure
> -------------------------
>  off
> (1 row)
> 
> postgres=# set ignore_checksum_failure = on;
> SET
> postgres=# show ignore_checksum_failure;
>  ignore_checksum_failure
> -------------------------
>  on
> (1 row)
> 
> postgres=# select * from test_data limit 10;
> WARNING:  page verification failed, calculated checksum 50372 but expected 26896
> ERROR:  invalid page in block 0 of relation base/13761/16384
> ```
> Ни попытки задать этот параметр другими способами (например, через `alter system`), ни вызовы `pg_reload_conf()`, ни перезапуск кластера не помогли - даже после включения `ignore_checksum_failure` сервер продолжает возвращать ошибки при обращении к таблице. Поэтому эту часть домашнего задания проверить не удалось.