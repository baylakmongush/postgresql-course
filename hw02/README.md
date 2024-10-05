# Домашнее задание №2

На уже установленном pg с прошлого домашнего задания:

## 1. Создаем базу данных iso и подключаемся к ней в 2 сессиях:

### `Сессия 1`
```
postgres=# CREATE DATABASE iso;
CREATE DATABASE
postgres=# \c iso
You are now connected to database "iso" as user "postgres".
```

### `Сессия 2`
```
postgres=# \c iso
You are now connected to database "iso" as user "postgres".
```

## 2. Создаем таблицу и наполняем ее данными

### `Сессия 1`
```
iso=# CREATE TABLE products (
    product_no integer,
    name text,
    price numeric
);
CREATE TABLE
iso=# INSERT INTO products (product_no, name, price) VALUES
    (1, 'Cheese', 9.99),
    (2, 'Bread', 1.99),
    (3, 'Milk', 2.99),
    (4, 'Water', 0.99),
    (5, 'Eggs', 2.99),
    (6, 'Sugar', 1.99);
INSERT 0 6
iso=# select * from products;
 product_no |  name  | price
------------+--------+-------
          1 | Cheese |  9.99
          2 | Bread  |  1.99
          3 | Milk   |  2.99
          4 | Water  |  0.99
          5 | Eggs   |  2.99
          6 | Sugar  |  1.99
(6 rows)
```

### `Сессия 2`
```
iso=# \dt
          List of relations
 Schema |   Name   | Type  |  Owner
--------+----------+-------+----------
 public | products | table | postgres
(1 row)
iso=# select * from products;
 product_no |  name  | price
------------+--------+-------
          1 | Cheese |  9.99
          2 | Bread  |  1.99
          3 | Milk   |  2.99
          4 | Water  |  0.99
          5 | Eggs   |  2.99
          6 | Sugar  |  1.99
(6 rows)
```

## 3. Проверим текущий уровень изоляции

### `Сессия 1`
```
iso=# select current_setting('transaction_isolation');
 current_setting
-----------------
 read committed
(1 row)
```

### `Сессия 2`
```
iso=# select current_setting('transaction_isolation');
 current_setting
-----------------
 read committed
(1 row)
```
> Используется дефолтный уровень изоляции `read committed`.

## 4. Начнем транзакцию в обеих сессиях

### `Сессия 1`
```
iso=# begin;
BEGIN
iso=*#
```

### `Сессия 2`
```
iso=# begin;
BEGIN
iso=*#
```
> 

## 5. Добавим новую запись во 1 сессии, а во второй сделаем запрос на выбор всех записей

### `Сессия 1`
```
iso=*# INSERT INTO products (name, price, product_no) VALUES ('Flour', 0.99, 7);
INSERT 0 1
```

### `Сессия 2`
```
iso=*# select * from products;
 product_no |  name  | price
------------+--------+-------
          1 | Cheese |  9.99
          2 | Bread  |  1.99
          3 | Milk   |  2.99
          4 | Water  |  0.99
          5 | Eggs   |  2.99
          6 | Sugar  |  1.99
(6 rows)
```
> Во второй сессии в транзакции не видно новой записи, потому что первая транзакция не завершилась и не подтвердилась, в случае изоляции `read committed` во второй сессии в транзакции изменения появлятся только в случае завершения транзакции в первой сессии.

## 6. Завершаем транзакцию в 1 сессии, а во второй сделаем запрос на выбор всех записей

### `Сессия 1`
```
iso=*# commit;
COMMIT
```

### `Сессия 2`
```
iso=*# select * from products;
 product_no |  name  | price
------------+--------+-------
          1 | Cheese |  9.99
          2 | Bread  |  1.99
          3 | Milk   |  2.99
          4 | Water  |  0.99
          5 | Eggs   |  2.99
          6 | Sugar  |  1.99
          7 | Flour  |  0.99
(7 rows)
```
> Во второй сессии в транзакции запись появилась из-за того, что первая транзакция подтверждена commit`ом или завершена. И эта аномалия называется - неповторяющее чтение. В большинстве случаев и кейсов считается нормой, но иногда может вызвать ошибки, если ожидаете неизменность данных. Для решения данной проблемы можно использовать Reapetable read иои Serializable уровни изоляции.

## 7. Во второй сессии завершаем транзакцию

### `Сессия 2`
```
iso=*# commit;
COMMIT
```

## 8. Начинаем новые транзакции с `repeatable read`.

### `Сессия 1`
```
iso=# begin transaction isolation level repeatable read;
BEGIN
iso=*# select current_setting('transaction_isolation');
 current_setting
-----------------
 repeatable read
(1 row)
```

### `Сессия 2`
```
iso=# begin transaction isolation level repeatable read;
BEGIN
iso=*# select current_setting('transaction_isolation');
 current_setting
-----------------
 repeatable read
(1 row)
```

## 9. Добавляем запись в первой сессии в транзакции, а во второй сделаем запрос на выбор всех записей

### `Сессия 1`
```
iso=*# INSERT INTO products (name, price, product_no) VALUES ('Potato', 1.99, 8);
INSERT 0 1
```

### `Сессия 2`
```
iso=*# select * from products;
 product_no |  name  | price
------------+--------+-------
          1 | Cheese |  9.99
          2 | Bread  |  1.99
          3 | Milk   |  2.99
          4 | Water  |  0.99
          5 | Eggs   |  2.99
          6 | Sugar  |  1.99
          7 | Flour  |  0.99
(7 rows)
```

> Новую запись во второй транзакции не видно, потому что данная транзакция обеспечивает неизменность данных в рамках текущей транзакции, как писала ранее, для решения проблем с неповторяющимися данными.

## 10. Завершаем транзакцию в 1 сессии, а во второй сделаем запрос на выбор всех записей

### `Сессия 1`
```
iso=*# commit;
COMMIT
```

### `Сессия 2`
```
iso=*# select * from products;
 product_no |  name  | price
------------+--------+-------
          1 | Cheese |  9.99
          2 | Bread  |  1.99
          3 | Milk   |  2.99
          4 | Water  |  0.99
          5 | Eggs   |  2.99
          6 | Sugar  |  1.99
          7 | Flour  |  0.99
(7 rows)
```
> Новую запись во второй транзакции не видно из-за того, что транзакция показывает одни и те же данные с начала работы, даже если другая транзакция завершила и сохранила изменения. В данном случае наблюдается аномалия `фантомного чтения`.