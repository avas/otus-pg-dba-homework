# Установка PostgreSQL

## Подготовка окружения

Для выполнения этого задания я решил использовать виртуальную машину, созданную в [предыдущем домашнем задании](../1-postgresql-isolation-levels/README.md#подготовка-виртуальной-машины-и-установка-субд). В теории, параметры этой ВМ более чем достаточны для запуска и комфортной работы контейнеров с PostgreSQL и/или с `psql`.

Для установки Docker Engine я воспользовался [инструкцией по установке из `apt`-репозитория](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository):
```bash
a-vasenev@pg-dba-vm:~$ sudo apt-get update
a-vasenev@pg-dba-vm:~$ sudo apt-get upgrade
a-vasenev@pg-dba-vm:~$ sudo apt-get install ca-certificates curl
a-vasenev@pg-dba-vm:~$ sudo install -m 0755 -d /etc/apt/keyrings
a-vasenev@pg-dba-vm:~$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
a-vasenev@pg-dba-vm:~$ sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Далее потребовалось выполнить ещё одну команду для добавления `apt`-репозитория с пакетами Docker...
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

...и установить оттуда всё необходимое:
```bash
a-vasenev@pg-dba-vm:~$ sudo apt-get update
a-vasenev@pg-dba-vm:~$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Проверим, что Docker Engine работает правильно:
```bash
a-vasenev@pg-dba-vm:~$ sudo docker run hello-world

...

Hello from Docker!
This message shows that your installation appears to be working correctly.

...
```

И раз уж он работает, можно перейти дальше, к созданию контейнера с PostgreSQL. 🙂

## Создание и запуск контейнера с PostgreSQL

Создадим директорию, в которую PostgreSQL в контейнере будет сохранять свои данные - например, базы данных:
```bash
a-vasenev@pg-dba-vm:~$ sudo mkdir /var/lib/postgresql-docker
a-vasenev@pg-dba-vm:~$ ls -lA /var/lib/postgresql*
/var/lib/postgresql:
total 12
drwxr-xr-x 3 postgres postgres 4096 Feb  2 11:32 14
-rw------- 1 postgres postgres   50 Feb  8 12:16 .bash_history
-rw------- 1 postgres postgres  257 Feb  8 11:44 .psql_history

/var/lib/postgresql-docker:
total 0
```

На этой машине уже был установлен локальный кластер PostgreSQL 14 (в ходе предыдущего домашнего задания), поэтому директория `/var/lib/postgresql` уже существует, и в ней уже есть какие-то данные этого кластера. Директория `/var/lib/postgresql-docker` же создана только что, и в ней пока ничего нет.

Создадим виртуальную сеть, в которой будут работать тестовые контейнеры:
```bash
a-vasenev@pg-dba-vm:~$ sudo docker network create pg-network
57625d67d5495ea7e7228dfcc7ca2c4e0b1f94267c203156f56b0937e9d06b6e
```

Запустим контейнер с сервером PostgreSQL 15:
```bash
a-vasenev@pg-dba-vm:~$ sudo docker run --name pg-server-15 --network pg-network -e POSTGRES_PASSWORD=postgres -p 5433:5432 -v /var/lib/postgresql-docker:/var/lib/postgresql/data -d postgres:15
e7431bb107ee1a4196cf18945308890b6862e33eb4b25846912507eb0d990cc0
```
Здесь мы передаём контейнеру следующие локальные ресурсы:
* Порт 5433 машины-хоста связывается с портом 5432 окружения внутри контейнера. Порт 5433 выбран из-за того, что на порту 5432 хоста уже работает локальный кластер с PostgreSQL 14.
* Локальная директория `/var/lib/postgresql-docker` хоста связывается с директорией `/var/lib/postgresql/data` контейнера, в которую сервер PostgreSQL в контейнере сохраняет данные по умолчанию. Таким образом, все данные, которыми будет манипулировать PostgreSQL внутри контейнера, окажется на машине-хосте в директории `/var/lib/postgresql-docker`.
> **Примечание**: На этом шаге у меня случился небольшой затык. Поначалу я пытался запустить контейнер командой вида:
> ```
> $ sudo docker run postgres:15 -e POSTGRES_PASSWORD=postgres # ...и прочие параметры с именем, сетью, портами и т.п.
> ```
>
> Такие попытки завершались неудачей со следующим сообщением об ошибке:
> ```
> Error: Database is uninitialized and superuser password is not specified.
>        You must specify POSTGRES_PASSWORD to a non-empty value for the
>        superuser. For example, "-e POSTGRES_PASSWORD=password" on "docker run".
> ```
>
> Причина оказалась не совсем очевидной, но ужасно банальной:
> ```
> a-vasenev@pg-dba-vm:~$ sudo docker run --help
> Usage:  docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
> ```
> Как выяснилось, для `docker run` важен порядок параметров: всё, что перечислено перед именем образа (в моём случае - `postgres:15`), трактуется как параметры запуска контейнера, а всё, что после него - как команды, запускаемые внутри этого контейнера. Таким образом, все параметры, которые я указывал при запуске, не применялись к самому контейнеру, а трактовались как команда и запускались внутри него. После перестановки имени образа в конец команды контейнер наконец запустился.

## Подключение к запущенному в контейнере кластеру

Теперь запустим контейнер с клиентом, присоединим его к той же сети и подключимся к серверу, работающему в запущенном ранее контейнере:
```
a-vasenev@pg-dba-vm:~$ sudo docker run --network pg-network --name pg-client --rm -it postgres:15 psql -h pg-server-15 -U postgres
Password for user postgres:
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

postgres=# \conninfo
You are connected to database "postgres" as user "postgres" on host "pg-server-15" (address "172.18.0.2") at port "5432".
postgres=#
```

Как видно, подключиться удалось. 🙂 Создадим базу данных, а в ней какую-нибудь таблицу с данными - например, таблицу из предыдущего домашнего задания:
```sql
postgres=# create database otus_docker;
CREATE DATABASE
postgres=# \c otus_docker;
You are now connected to database "otus_docker" as user "postgres".
otus_docker=# create table persons (id serial, first_name varchar(250), last_name varchar(250));
CREATE TABLE
otus_docker=# insert into persons (first_name, last_name) values ('Ivan', 'Ivanov'), ('Petr', 'Petrov'), ('Sergey', 'Sergeev'), ('Sveta', 'Svetova');
INSERT 0 4
otus_docker=# select * from persons;
 id | first_name | last_name
----+------------+-----------
  1 | Ivan       | Ivanov
  2 | Petr       | Petrov
  3 | Sergey     | Sergeev
  4 | Sveta      | Svetova
(4 rows)
```

Ради интереса можно подключиться к этому же кластеру прямо из ОС хостовой машины, на которой работает контейнер - проверим, что эти данные будут видны:
```
a-vasenev@pg-dba-vm:~$ psql -h localhost -p 5433 -U postgres
Password for user postgres:
psql (14.10 (Ubuntu 14.10-0ubuntu0.22.04.1), server 15.6 (Debian 15.6-1.pgdg120+2))
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
Type "help" for help.

postgres=# \l
                                  List of databases
    Name     |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-------------+----------+----------+------------+------------+-----------------------
 otus_docker | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 postgres    | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0   | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
             |          |          |            |            | postgres=CTc/postgres
 template1   | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
             |          |          |            |            | postgres=CTc/postgres
(4 rows)

postgres=# \c otus_docker
psql (14.10 (Ubuntu 14.10-0ubuntu0.22.04.1), server 15.6 (Debian 15.6-1.pgdg120+2))
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
You are now connected to database "otus_docker" as user "postgres".
otus_docker=# \d
               List of relations
 Schema |      Name      |   Type   |  Owner
--------+----------------+----------+----------
 public | persons        | table    | postgres
 public | persons_id_seq | sequence | postgres
(2 rows)

otus_docker=# select * from persons;
 id | first_name | last_name
----+------------+-----------
  1 | Ivan       | Ivanov
  2 | Petr       | Petrov
  3 | Sergey     | Sergeev
  4 | Sveta      | Svetova
(4 rows)
```

Мы видим ту же БД, а в ней - ту же таблицу, которые были созданы ранее, при подключении из запущенного в контейнере клиента. Из этого можно сделать вывод, что мы действительно подключились к тому же кластеру, и он действительно доступен на том порту машины-хоста, который мы ранее предоставили контейнеру с сервером PostgreSQL (5433).

Теперь попробуем подключиться к этому же кластеру с какого-нибудь внешнего компьютера:
```
PS C:\Users\a.vasenev> cd "C:\Program Files\pgAdmin 4\v7\runtime"
PS C:\Program Files\pgAdmin 4\v7\runtime> .\psql.exe -h (внешний_IP_виртуальной_машины) -p 5433 -U postgres
Password for user postgres:
psql (16.0, server 15.6 (Debian 15.6-1.pgdg120+2))
WARNING: Console code page (866) differs from Windows code page (1251)
         8-bit characters might not work correctly. See psql reference
         page "Notes for Windows users" for details.
Type "help" for help.

postgres=# \l
                                                       List of databases
    Name     |  Owner   | Encoding | Locale Provider |  Collate   |   Ctype    | ICU Locale | ICU Rules |   Access privileges
-------------+----------+----------+-----------------+------------+------------+------------+-----------+-----------------------
 otus_docker | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           |
 postgres    | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           |
 template0   | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           | =c/postgres          +
             |          |          |                 |            |            |            |           | postgres=CTc/postgres
 template1   | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           | =c/postgres          +
             |          |          |                 |            |            |            |           | postgres=CTc/postgres
(4 rows)

postgres=# \c otus_docker
psql (16.0, server 15.6 (Debian 15.6-1.pgdg120+2))
You are now connected to database "otus_docker" as user "postgres".
otus_docker=# \d
               List of relations
 Schema |      Name      |   Type   |  Owner
--------+----------------+----------+----------
 public | persons        | table    | postgres
 public | persons_id_seq | sequence | postgres
(2 rows)

otus_docker=# select * from persons;
 id | first_name | last_name
----+------------+-----------
  1 | Ivan       | Ivanov
  2 | Petr       | Petrov
  3 | Sergey     | Sergeev
  4 | Sveta      | Svetova
(4 rows)
```
Таким образом, подключиться к кластеру удалось и с внешнего хоста, расположенного за пределами виртуальной сети Docker и за пределами инфраструктуры Яндекс.Облака.

## Пересоздание контейнера с СУБД с сохранением данных

Теперь убедимся, что данные действительно сохранены на машине-хосте в директории, указанной при создании контейнера с сервером PostgreSQL. Проще всего это сделать, просто заглянув в эту директорию (ранее, сразу после создания, она была пуста):
```bash
a-vasenev@pg-dba-vm:~$ sudo ls -lA /var/lib/postgresql-docker/
total 124
drwx------ 6 lxd docker  4096 Feb 19 09:51 base
drwx------ 2 lxd docker  4096 Feb 19 09:51 global
drwx------ 2 lxd docker  4096 Feb 19 09:50 pg_commit_ts
drwx------ 2 lxd docker  4096 Feb 19 09:50 pg_dynshmem
-rw------- 1 lxd docker  4821 Feb 19 09:51 pg_hba.conf
-rw------- 1 lxd docker  1636 Feb 19 09:50 pg_ident.conf
drwx------ 4 lxd docker  4096 Feb 19 09:53 pg_logical
drwx------ 4 lxd docker  4096 Feb 19 09:50 pg_multixact
drwx------ 2 lxd docker  4096 Feb 19 09:50 pg_notify
drwx------ 2 lxd docker  4096 Feb 19 09:50 pg_replslot
drwx------ 2 lxd docker  4096 Feb 19 09:50 pg_serial
drwx------ 2 lxd docker  4096 Feb 19 09:50 pg_snapshots
drwx------ 2 lxd docker  4096 Feb 19 09:53 pg_stat
drwx------ 2 lxd docker  4096 Feb 19 09:50 pg_stat_tmp
drwx------ 2 lxd docker  4096 Feb 19 09:50 pg_subtrans
drwx------ 2 lxd docker  4096 Feb 19 09:50 pg_tblspc
drwx------ 2 lxd docker  4096 Feb 19 09:50 pg_twophase
-rw------- 1 lxd docker     3 Feb 19 09:50 PG_VERSION
drwx------ 3 lxd docker  4096 Feb 19 09:50 pg_wal
drwx------ 2 lxd docker  4096 Feb 19 09:50 pg_xact
-rw------- 1 lxd docker    88 Feb 19 09:50 postgresql.auto.conf
-rw------- 1 lxd docker 29525 Feb 19 09:50 postgresql.conf
-rw------- 1 lxd docker    36 Feb 19 09:51 postmaster.opts
```

Видно, что в ней есть файлы конфигурации и прочие данные, которые относятся к кластеру PostgreSQL из контейнера, а значит, он действительно сохранял свои данные в эту директорию.

Теперь можно попробовать более радикальный шаг и удалить контейнер с СУБД:
```bash
a-vasenev@pg-dba-vm:~$ sudo docker stop pg-server-15
pg-server-15
a-vasenev@pg-dba-vm:~$ sudo docker rm pg-server-15
pg-server-15
```
Убедимся, что кластер, к которому мы подключались ранее, больше не доступен ни из контейнера с клиентом...
```bash
a-vasenev@pg-dba-vm:~$ sudo docker run --network pg-network --name pg-client --rm -it postgres:15 psql -h pg-server-15 -U postgres
psql: error: could not translate host name "pg-server-15" to address: Temporary failure in name resolution
```
...ни с машины-хоста:
```bash
a-vasenev@pg-dba-vm:~$ psql -h localhost -p 5433 -U postgres
psql: error: connection to server at "localhost" (::1), port 5433 failed: Connection refused
        Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 5433 failed: Connection refused
        Is the server running on that host and accepting TCP/IP connections?
```
Снова создадим контейнер с сервером, используя те же параметры, и снова попробуем к нему подключиться:
```bash
a-vasenev@pg-dba-vm:~$ sudo docker run --name pg-server-15 --network pg-network -e POSTGRES_PASSWORD=postgres -p 5433:5432 -v /var/lib/postgresql-docker:/var/lib/postgresql/data -d postgres:15
b992e1ff3b2162834867c290b098a85bf33cd640b9dbca23bcce7b411ab96e8d
```
```bash
a-vasenev@pg-dba-vm:~$ sudo docker run --network pg-network --name pg-client --rm -it postgres:15 psql -h pg-server-15 -U postgres
Password for user postgres:
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

postgres=#
```

Итак, кластер снова работает. Убедимся, что данные по-прежнему на месте:
```sql
postgres=# \l
                                                 List of databases
    Name     |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-------------+----------+----------+------------+------------+------------+-----------------+-----------------------
 otus_docker | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 postgres    | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0   | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
             |          |          |            |            |            |                 | postgres=CTc/postgres
 template1   | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
             |          |          |            |            |            |                 | postgres=CTc/postgres
(4 rows)

postgres=# \c otus_docker
You are now connected to database "otus_docker" as user "postgres".
otus_docker=# \d
               List of relations
 Schema |      Name      |   Type   |  Owner
--------+----------------+----------+----------
 public | persons        | table    | postgres
 public | persons_id_seq | sequence | postgres
(2 rows)

otus_docker=# select * from persons;
 id | first_name | last_name
----+------------+-----------
  1 | Ivan       | Ivanov
  2 | Petr       | Petrov
  3 | Sergey     | Sergeev
  4 | Sveta      | Svetova
(4 rows)
```

Как видно, и созданная ранее БД, и данные в этой таблице по-прежнему доступны, несмотря на то, что контейнер с кластером PostgreSQL был удалён и создан заново. Причина этого в том, что данные этого кластера находились не внутри контейнера, а за его пределами - в примонтированной при создании контейнера директории `/var/lib/postgresql-docker`, которая была смонтирована в директорию `/var/lib/postgresql/data` контейнера. При удалении контейнера данные в этой директории остались в целости, и затем, когда мы пересоздали контейнер, сервер PostgreSQL в новом контейнере обнаружил эти данные. Поскольку мажорная версия сервера и в удалённом контейнере, и в заново созданном контейнере совпадает (в обоих случаях использовался один и тот же образ `postgresql:15`, который в данный момент содержит PostgreSQL 15.6), никакие дополнительные действия для обновления этих данных не потребовались - сервер PostgreSQL в новом созданном контейнере смог их загрузить и продолжил с ними работать.

> **Примечание**: В этом разделе домашнего задания я столкнулся ещё с одной тонкостью, которая мне показалась неочевидной. В своих первых попытках я пробовал создавать контейнер с сервером PostgreSQL командой вида:
> ```bash
> $ sudo docker run ... -v /var/lib/postgresql-docker:/var/lib/postgresql -d postgres:15 # несущественные параметры пропущены
> ```
> Здесь директория хоста связывается с директорией контейнера `/var/lib/postgresql`, а не `/var/lib/postgresql/data`. Казалось бы, `data` находится внутри монтируемой директории, и запись в неё тоже должна перенаправляться в директорию хоста. Тем не менее, при таком варианте в директории хоста `/var/lib/postgresql-docker` появлялась поддиректория `data`, но данные в неё не записывались - они оставались в локальной файловой системе контейнера и, как результат, терялись при пересоздании контейнера.
>
> Разгадка снова оказалась не совсем очевидной, но очень банальной:
> ```bash
> a-vasenev@pg-dba-vm:~$ sudo docker exec pg-server-15 mount | grep "/var/lib/postgresql"
> /dev/vda2 on /var/lib/postgresql type ext4 (rw,relatime)
> /dev/vda2 on /var/lib/postgresql/data type ext4 (rw,relatime)
> ```
> Здесь первая запись вызвана параметрами, переданными при создании контейнера, а вторая - [параметрами в Dockerfile образа PostgreSQL](https://github.com/docker-library/postgres/blob/34d4c14c235806e57fdd5eaf197f718fccee93b0/15/bookworm/Dockerfile#L189). Директория `/var/lib/postgresql/data` монтируется отдельно, поэтому всё, что в неё пишется, попадает не в поддиректорию `data` в примонтированной директории `/var/lib/postgresql`, а на отдельный раздел, примонтированный в `/var/lib/postgresql/data`.
>
> Если же пересоздать контейнер как положено, с монтированием директории хоста в `/var/lib/postgresql/data`:
> ```bash
> $ sudo docker run ... -v /var/lib/postgresql-docker:/var/lib/postgresql/data -d postgres:15
> ```
> ...то директория `/var/lib/postgresql/data` окажется связана с директорией хоста:
> ```bash
> a-vasenev@postgresql-dba-2024-1-vm:~$ sudo docker exec pg-server-15 mount | grep "/var/lib/postgresql"
> /dev/vda2 on /var/lib/postgresql/data type ext4 (rw,relatime)
> ```
> И теперь записываемые в неё данные действительно попадут на диск хоста и смогут жить независимо от времени жизни контейнера.
