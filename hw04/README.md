# Домашнее задание №4

На уже установленном pg с прошлого домашнего задания:

## 1.Создать таблицу accounts(id integer, amount numeric)

Создадим базу данных locks и подключаемся к ней:

```
postgres=# CREATE DATABASE locks;
CREATE DATABASE
postgres=# \c locks
You are now connected to database "locks" as user "postgres".
```

Создадим таблицу accounts(id integer, amount numeric);

```
postgres=# \c locks
You are now connected to database "locks" as user "postgres".
locks=# CREATE TABLE accounts (
    id integer,
    amount numeric
);
CREATE TABLE
locks=# \dt
          List of relations
 Schema |   Name   | Type  |  Owner
--------+----------+-------+----------
 public | accounts | table | postgres
(1 row)
```

## 2. Добавить несколько записей и подключившись через 2 терминала добиться ситуации взаимоблокировки (deadlock).

```
locks=# INSERT INTO accounts (id, amount) VALUES (1, 10.00);
INSERT 0 1
locks=# INSERT INTO accounts (id, amount) VALUES (2, 20.00);
INSERT 0 1
locks=# INSERT INTO accounts (id, amount) VALUES (3, 30.00);
INSERT 0 1
```

откроем 2 сессии

1 сессия - начнем транзакцию и попытаемся обновить amount с id = 1
```
locks=# begin;
BEGIN
locks=*# UPDATE accounts SET amount = amount + 1 WHERE id = 1;
UPDATE 1
```

2 сессия - начнем транзакцию и попытаемся обновить amount с id = 2
```
locks=# begin;
BEGIN
locks=*# UPDATE accounts SET amount = amount + 1 WHERE id = 2;
UPDATE 1
```

Проверим pid у 2 сессий:

1 сессия
```
locks=# select * from pg_backend_pid();
 pg_backend_pid
----------------
         231582
(1 row)
```
2 сессия;

```
locks=# select * from pg_backend_pid();
 pg_backend_pid
----------------
         231583
(1 row)
```


Затем переходим снова в 1 сессию и пытаемся обновить amount с id = 2, как и во второй транзакции, видимо, что зависло
```
locks=*# UPDATE accounts SET amount = amount + 1 WHERE id = 2;
|
```

Посмотрим в логах, там никакой информации, включим логирование всех блокировок:
```
postgres=# ALTER SYSTEM SET log_lock_waits = 'on';
ALTER SYSTEM SET deadlock_timeout = '1s';
ALTER SYSTEM
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# SHOW log_lock_waits;
SHOW deadlock_timeout;
 log_lock_waits
----------------
 on
(1 row)

 deadlock_timeout
------------------
 1s
(1 row)
```

После включения логгирования:
```
2024-10-19 15:06:29.542 MSK [231582] postgres@locks LOG:  process 231582 still waiting for ShareLock on transaction 1196 after 1000.060 ms
2024-10-19 15:06:29.542 MSK [231582] postgres@locks DETAIL:  Process holding the lock: 231583. Wait queue: 231582.
2024-10-19 15:06:29.542 MSK [231582] postgres@locks CONTEXT:  while updating tuple (0,2) in relation "accounts"
2024-10-19 15:06:29.542 MSK [231582] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 1 WHERE id = 2;
```

ShareLock означает, что транзакция 2-ой сессии (pid 231583) удерживает блокировку, что другие транзакции (231582 - это 1-ая сессия) не могут изменять этот объект, чтобы избежать конфликтов при чтении и модификации одновремено.


Теперь во второй сессии пытаемся обновить amount с id = 1, как и в первой транзакции:

```
locks=*# UPDATE accounts SET amount = amount + 1 WHERE id = 1;
ERROR:  deadlock detected
DETAIL:  Process 231583 waits for ShareLock on transaction 1195; blocked by process 231582.
Process 231582 waits for ShareLock on transaction 1196; blocked by process 231583.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "accounts"
```

И видим ошибку!

## 3. Посмотреть логи и убедиться, что информация о дедлоке туда попала.

В логах видим информацию, что `ERROR:  deadlock detected`

```
2024-10-19 15:13:22.065 MSK [231583] postgres@locks LOG:  process 231583 detected deadlock while waiting for ShareLock on transaction 1195 after 1000.096 ms
2024-10-19 15:13:22.065 MSK [231583] postgres@locks DETAIL:  Process holding the lock: 231582. Wait queue: .
2024-10-19 15:13:22.065 MSK [231583] postgres@locks CONTEXT:  while updating tuple (0,1) in relation "accounts"
2024-10-19 15:13:22.065 MSK [231583] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 1 WHERE id = 1;
2024-10-19 15:13:22.065 MSK [231583] postgres@locks ERROR:  deadlock detected
2024-10-19 15:13:22.065 MSK [231583] postgres@locks DETAIL:  Process 231583 waits for ShareLock on transaction 1195; blocked by process 231582.
        Process 231582 waits for ShareLock on transaction 1196; blocked by process 231583.
        Process 231583: UPDATE accounts SET amount = amount + 1 WHERE id = 1;
        Process 231582: UPDATE accounts SET amount = amount + 1 WHERE id = 2;
2024-10-19 15:13:22.065 MSK [231583] postgres@locks HINT:  See server log for query details.
2024-10-19 15:13:22.065 MSK [231583] postgres@locks CONTEXT:  while updating tuple (0,1) in relation "accounts"
2024-10-19 15:13:22.065 MSK [231583] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 1 WHERE id = 1;
2024-10-19 15:13:22.065 MSK [231582] postgres@locks LOG:  process 231582 acquired ShareLock on transaction 1196 after 413522.677 ms
2024-10-19 15:13:22.065 MSK [231582] postgres@locks CONTEXT:  while updating tuple (0,2) in relation "accounts"
2024-10-19 15:13:22.065 MSK [231582] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 1 WHERE id = 2;
```
