# Домашнее задание №5

## 1. Установка postgres на debian 12 на 2 инстансах

Характеристика сервера: 2 cpu, 6 ram, 15g disk.

### 1) Обновление списка пакетов
`sudo apt update && sudo apt upgrade`

### 2) Установка пакета из репозитория Debian с утилитой apt
`sudo apt install postgresql postgresql-contrib`

### 3) Проверка работы сервиса postgresql

`sudo systemctl status postgresql`

```
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; preset: enabled)
     Active: active (exited) since Wed 2024-10-02 21:22:12 MSK; 6s ago
   Main PID: 28162 (code=exited, status=0/SUCCESS)
        CPU: 1ms
```

### 4) sudo su - postgres

мастер:

```
postgres@test1:~$ psql
psql (15.8 (Debian 15.8-0+deb12u1))
Type "help" for help.

postgres=# \q
```

```
postgres=# SELECT version();
                                                      version
-------------------------------------------------------------------------------------------------------------------
 PostgreSQL 15.8 (Debian 15.8-0+deb12u1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14) 12.2.0, 64-bit
(1 row)
```

реплика:
```
postgres@test2:~$ psql
psql (15.8 (Debian 15.8-0+deb12u1))
Type "help" for help.

postgres=# \q
```

```
postgres=# SELECT version();
                                                      version
-------------------------------------------------------------------------------------------------------------------
 PostgreSQL 15.8 (Debian 15.8-0+deb12u1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14) 12.2.0, 64-bit
(1 row)
```

## 2. Получение ip-адресов

мастер - `192.168.2.2`

реплика - `192.168.1.2`

## 3. Подготовка мастера для репликации

### 1) Добавление в конфигурацию postgres listen_addresses

/etc/postgresql/15/main/postgresql.conf:

```
cat >> /etc/postgresql/15/main/postgresql.conf << EOL
listen_addresses = '192.168.2.2'
EOL
```

### 2) Добавление в pg_hba пользователя для репликации и пароль в scram-sha-256
/etc/postgresql/15/main/pg_hba.conf:

```
cat >> /etc/postgresql/15/main/pg_hba.conf << EOL
host replication replicator 0.0.0.0/0 scram-sha-256
EOL
```

### 2) Рестарт кластера pg под postgres

```
pg_ctlcluster 15 main stop && pg_ctlcluster 15 main start
```
```
postgres@test1:/home/mongush.baylak3$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

### 3) Зальем данными мастер

```
wget https://storage.googleapis.com/thaibus/thai_small.tar.gz && tar -xf thai_small.tar.gz && psql < thai.sql

postgres@test1:/home/mongush.baylak3$ psql
psql (15.8 (Debian 15.8-0+deb12u1))
Type "help" for help.

postgres=# \l
                                             List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+---------+---------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
 thai      | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
(4 rows)
```

### 4) Создадим пользователя для репликации
```
postgres@test1:~$ psql -c "CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'secret123';"
CREATE ROLE

postgres=# \du
                                    List of roles
 Role name  |                         Attributes                         | Member of
------------+------------------------------------------------------------+-----------
 postgres   | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 replicator | Replication                                                | {}
```

### 5) Создадим слот репликации

```
postgres@test1:~$ psql -c "SELECT pg_create_physical_replication_slot('test');"
 pg_create_physical_replication_slot
-------------------------------------
 (test,)
(1 row)
```

## 3. Подготовка реплики

### 1) Пропишем в файле пароль, чтобы не требовал пароль

```
cat >> ~/.pgpass << EOL
test1.pg-course-mongush.vm.nb.clv2:5432:*:replicator:secret123
EOL

chmod 0600 ~/.pgpass
```

### 2) Остановим кластер

```
pg_ctlcluster 15 main stop
```

### 3) Очистим от лишних данных кластер

```
rm -rf /var/lib/postgresql/15/main
```

### 4) Зальем при помощи бэкапирования pg_basebackup данными из мастера

```
pg_basebackup -h test1.pg-course-mongush.vm.nb.clv2 -p 5432 -U replicator -R -S test -D /var/lib/postgresql/15/main
```

### 5) Проверяем статус кластера, убедившись, что статус recovery
```
postgres@test2:/home/mongush.baylak3$ pg_lsclusters
Ver Cluster Port Status        Owner    Data directory              Log file
15  main    5432 down,recovery postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

## 4. Тестирование standalone

### 1) Создаем файл с данными для тестирования записи на мастере standalone

```
postgres@test1:/home/mongush.baylak3$ cat > ~/workload2.sql << EOL
INSERT INTO book.tickets (fkRide, fio, contact, fkSeat)
VALUES (
        ceil(random()*100)
        , (array(SELECT fam FROM book.fam))[ceil(random()*110)]::text || ' ' ||
    (array(SELECT nam FROM book.nam))[ceil(random()*110)]::text
    ,('{"phone":"+7' || (1000000000::bigint + floor(random()*9000000000)::bigint)::text || '"}')::jsonb
    , ceil(random()*100));

EOL
```

### 2) Тестируем запись на мастере standalone при помощи pgbench

```
postgres@test1:/home/mongush.baylak3$ /usr/lib/postgresql/15/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
pgbench (15.8 (Debian 15.8-0+deb12u1))
transaction type: /var/lib/postgresql/workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 53501
number of failed transactions: 0 (0.000%)
latency average = 1.495 ms
initial connection time = 13.141 ms
tps = 5351.004855 (without initial connection time)
```

### 3) Создаем файл с данными для тестирования чтения на мастере standalone

```
postgres@test1:/home/mongush.baylak3$ cat > ~/workload.sql << EOL

\set r random(1, 5000000)
SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r;

EOL
```

### 4) Тестируем чтение на мастере standalone при помощи pgbench

```
postgres@test1:/home/mongush.baylak3$ /usr/lib/postgresql/15/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
pgbench (15.8 (Debian 15.8-0+deb12u1))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 249342
number of failed transactions: 0 (0.000%)
latency average = 0.320 ms
initial connection time = 17.750 ms
tps = 24967.436652 (without initial connection time)
```

## 4. Тестирование кластера с 1 асинк репликой

### 1) Стартуем реплику

```
postgres@test2:/home/mongush.baylak3$ pg_lsclusters
Ver Cluster Port Status          Owner    Data directory              Log file
15  main    5432 online,recovery postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

### 2) Проверяем pg_is_in_recovery

```
postgres@test2:/home/mongush.baylak3$ psql -d thai -c "select pg_is_in_recovery();"
 pg_is_in_recovery
-------------------
 t
(1 row)
```

### 3) Сверяем количество данных с мастером

Количество поездок:

Мастер:
```
postgres@test1:/home/mongush.baylak3$ psql -d thai -c "select count(*) from book.tickets;"
  count
---------
 5239006
(1 row)
```

Реплика:
```
postgres@test2:/home/mongush.baylak3$ psql -d thai -c "select count(*) from book.tickets;"
  count
---------
 5239006
(1 row)
```


### 4) Тестируем запись на мастере с включенной асинк репликой при помощи pgbench

мастер:
```
postgres@test1:/home/mongush.baylak3$ /usr/lib/postgresql/15/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
pgbench (15.8 (Debian 15.8-0+deb12u1))
transaction type: /var/lib/postgresql/workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 43139
number of failed transactions: 0 (0.000%)
latency average = 1.856 ms
initial connection time = 13.668 ms
tps = 4311.191279 (without initial connection time)
```

> Видим, что запись стала чуть быстрее

### 5) Тестируем чтение на мастере с включенной асинк репликой при помощи pgbench

postgres@test1:/home/mongush.baylak3$ /usr/lib/postgresql/15/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
pgbench (15.8 (Debian 15.8-0+deb12u1))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 251350
number of failed transactions: 0 (0.000%)
latency average = 0.318 ms
initial connection time = 8.783 ms
tps = 25155.748461 (without initial connection time)

> Видим, что чтение стало чуть медленнее

### 6) Тестируем чтение на реплике при помощи pgbench, создав предварительно файл с данными

```
cat > ~/workload.sql << EOL

\set r random(1, 5000000)
SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r;

EOL
```

```
postgres@test2:/home/mongush.baylak3$ /usr/lib/postgresql/15/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
pgbench (15.8 (Debian 15.8-0+deb12u1))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 249731
number of failed transactions: 0 (0.000%)
latency average = 0.320 ms
initial connection time = 9.378 ms
tps = 24978.175565 (without initial connection time)
```

> Видим, что чтение стало чуть быстрее