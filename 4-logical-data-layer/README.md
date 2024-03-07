# Логический уровень данных в PostgreSQL

## Подготовка окружения
Для выполнения задания будет использоваться уже существующая виртуальная машина, на которой в предыдущих заданиях уже был развёрнут кластер PostgreSQL 14.
```
a-vasenev@pg-dba-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory                  Log file
14  main    5432 down   postgres /mnt/pg-dba-vm-external/14/main /var/log/postgresql/postgresql-14-main.log
```

Данные этого кластера не представляют никакой ценности, поэтому, возможно, для текущего задания его будет проще пересоздать:
```
a-vasenev@pg-dba-vm:~$ sudo pg_dropcluster 14 main
a-vasenev@pg-dba-vm:~$ sudo pg_createcluster 14 main

... вывод статусной информации о создании кластера ...

a-vasenev@pg-dba-vm:~$ sudo pg_ctlcluster 14 main start
a-vasenev@pg-dba-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```

Подключимся к этому кластеру:
```
a-vasenev@pg-dba-vm:~$ sudo su - postgres
postgres@pg-dba-vm:~$ psql
psql (14.11 (Ubuntu 14.11-0ubuntu0.22.04.1))
Type "help" for help.

postgres=#
```

Создадим на нём тестовые данные для выполнения домашнего задания - схему, таблицу, пользователя и роль для чтения данных. Для простоты также можно добавить в приглашение `psql` имя текущего пользователя - так будет проще ориентироваться в листингах дальше.
```sql
postgres=# create database testdb;
CREATE DATABASE
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=> \set PROMPT1 '%n@%/%R%x%# '
postgres@testdb=# create schema testnm;
CREATE SCHEMA
postgres@testdb=# create table t1 (c1 integer);
CREATE TABLE
postgres@testdb=# insert into t1 (c1) values (1);
INSERT 0 1
postgres@testdb=# create role readonly;
CREATE ROLE
postgres@testdb=# drop table t1;
DROP TABLE
postgres@testdb=# create table testnm.t1 (c1 integer);
CREATE TABLE
postgres@testdb=# insert into testnm.t1 (c1) values (1);
INSERT 0 1
postgres@testdb=# grant connect on database testdb to readonly;
GRANT
postgres@testdb=# grant usage on schema testnm to readonly;
GRANT
postgres@testdb=# grant select on all tables in schema testnm to readonly;
GRANT
postgres@testdb=# create user testread with password 'test123';
CREATE ROLE
postgres@testdb=# grant readonly to testread;
GRANT ROLE
```

## Эксперименты с правами на чтение данных
Откроем ещё одну SSH-сессию, подключимся к СУБД от имени созданного пользователя и попробуем запросить данные:
```sql
a-vasenev@pg-dba-vm:~$ psql -h localhost -U testread -d testdb
Password for user testread:
psql (14.11 (Ubuntu 14.11-0ubuntu0.22.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> \set PROMPT1 '%n@%/%R%x%# '
testread@testdb=> select * from t1;
ERROR:  relation "t1" does not exist
LINE 1: select * from t1;
                      ^
testread@testdb=> show search_path;
   search_path
-----------------
 "$user", public
(1 row)

testread@testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)
```
Как видно, в первый раз запрос не сработал - я не указал схему явно, и СУБД попыталась найти таблицу в схеме `public` (это видно по значению `search_path`). Там, разумеется, этой таблицы нет (она была создана в схеме `testnm`), и попытка запроса завершилась ошибкой. Но после явного указания схемы запрос сработал, и пользователь смог прочитать данные из таблицы.

> **Примечание**: На этом этапе я заглянул в запросы в шпаргалке и увидел, что запрос данных из таблицы должен был сработать по-другому. Причина в том, что в запросе в шпаргалке не указана схема, и из-за этого таблица создастся в схеме `public`. Попробуем повторить это точно так, как этого требовала шпаргалка (но поменяем имя таблицы, чтобы явно отличать её от таблицы, созданной в схеме `testnm`):
> ```sql
> postgres@testdb=# create table t1_public (c1 integer);
> CREATE TABLE
> postgres@testdb=# insert into t1_public (c1) values (2);
> INSERT 0 1
> postgres@testdb=# select * from t1_public;
>  c1
> ----
>   2
> (1 row)
> ```
> Снова вернёмся в окно с клиентом, запущенным от имени пользователя `testread`, и попробуем прочитать данные из этой таблицы:
> ```sql
> testread@testdb=> select * from t1_public;
> ERROR:  permission denied for table t1_public
> ```
> Теперь запрос завершился ошибкой, так как выданная пользователю `testread` роль `readonly` разрешает чтение только из схемы `testnm`. Таблица же, которую мы создали, находится в схеме `public`, на которую у пользователя разрешения нет.

## Эксперименты с правами на запись данных
Попробуем создать от имени пользователя `testread` таблицу и записать в неё какие-нибудь данные:
```sql
testread@testdb=> create table t2 (c2 integer);
CREATE TABLE
testread@testdb=> insert into t2 (c2) values (3);
INSERT 0 1
testread@testdb=> select * from t2;
 c2
----
  3
(1 row)
```
Видно, что таблица успешно создалась, и данные в неё были успешно записаны. Причина этого в том, что PostgreSQL 14 по умолчанию разрешает создание отношений в схеме `public` (право на это есть у роли `PUBLIC`, в которой состоят все пользователи, в том числе и `testread`). После создания же пользователь имеет полные права на созданные им отношения.

> **Примечание**: Здесь есть один момент, который мне не даёт покоя. В предыдущем разделе мы создали таблицу в схеме `public` от имени пользователя `postgres`, а затем попытались прочитать из неё данные от имени пользователя `testread`, и эта попытка завершилась неудачей. Однако в этом разделе мы увидели, что пользователь `testread` всё-таки смог прочитать данные из таблицы в схеме `public`, созданной им же самим. Почему же поведение отличается, ведь в обоих случаях таблицы находятся в одной и той же схеме? В чём принципиальная разница между этими двумя сценариями? Внятного ответа на этот вопрос у меня нет, но есть гипотеза:
> ```sql
> testread@testdb=> \d
>            List of relations
>  Schema |   Name    | Type  |  Owner
> --------+-----------+-------+----------
>  public | t1_public | table | postgres
>  public | t2        | table | testread
> (2 rows)
> 
> testread@testdb=> select * from information_schema.role_table_grants where grantee in ('testread', 'readonly');
>  grantor  | grantee  | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
> ----------+----------+---------------+--------------+------------+----------------+--------------+----------------
>  postgres | readonly | testdb        | testnm       | t1         | SELECT         | NO           | YES
>  testread | testread | testdb        | public       | t2         | INSERT         | YES          | NO
>  testread | testread | testdb        | public       | t2         | SELECT         | YES          | YES
>  testread | testread | testdb        | public       | t2         | UPDATE         | YES          | NO
>  testread | testread | testdb        | public       | t2         | DELETE         | YES          | NO
>  testread | testread | testdb        | public       | t2         | TRUNCATE       | YES          | NO
>  testread | testread | testdb        | public       | t2         | REFERENCES     | YES          | NO
>  testread | testread | testdb        | public       | t2         | TRIGGER        | YES          | NO
> (8 rows)
> ```
> По выводу команды `\d` видно, что PostgreSQL хранит информацию о пользователе, создавшем объект, а по выводу второго запроса видно, что у пользователя `testdb` действительно есть полные права на работу с таблицей `public.t2`. Поэтому мне кажется, что разрешения по умолчанию в PostgreSQL 14 разрешают в схеме `public` работу (в т.ч. чтение) только с таблицами, созданными текущим пользователем, но не с таблицами, созданными другими пользователями. Но я пока не знаю какого-то способа, которым это можно было бы подтвердить или опровергнуть.

Запретить создание отношений в схеме `public` можно, явно отобрав права на это у роли `PUBLIC`:
```sql
postgres@testdb=# revoke CREATE on schema public from PUBLIC;
REVOKE
postgres@testdb=# revoke ALL on database testdb from PUBLIC;
REVOKE
```
```sql
testread@testdb=> create table t3 (c3 integer);
ERROR:  permission denied for schema public
LINE 1: create table t3 (c3 integer);
                     ^
```
Теперь пользователь `testread` больше не может создавать таблицы.