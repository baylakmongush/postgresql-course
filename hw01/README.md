# Домашнее задание №1

Установка postgres на debian 12

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

### 5) Под postgres заливаем данные

```
Объем порядка 6 млн.строк (600МБ):
wget https://storage.googleapis.com/thaibus/thai_small.tar.gz && tar -xf thai_small.tar.gz && psql < thai.sql
```

### 6) Список бд

```
postgres-# \l
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

### 7) Использование бд thai

```
postgres-# \c thai
You are now connected to database "thai" as user "postgres".
```

### 8) Список schemas

```
thai-# \dn
      List of schemas
  Name  |       Owner
--------+-------------------
 book   | postgres
 public | pg_database_owner
(2 rows)
```

### 9) Список таблиц

```
thai-# \dt book.*
            List of relations
 Schema |     Name     | Type  |  Owner
--------+--------------+-------+----------
 book   | bus          | table | postgres
 book   | busroute     | table | postgres
 book   | busstation   | table | postgres
 book   | fam          | table | postgres
 book   | nam          | table | postgres
 book   | ride         | table | postgres
 book   | schedule     | table | postgres
 book   | seat         | table | postgres
 book   | seatcategory | table | postgres
 book   | tickets      | table | postgres
(10 rows)
```

### 10) Посчитать количество поездок:

```
thai=# select count(*) from book.tickets;
  count
---------
 5185505
(1 row)
```