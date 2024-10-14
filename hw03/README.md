# Домашнее задание №3

На уже установленном pg с прошлого домашнего задания:

## 1.Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1 млн строк

Создадим базу данных vac и подключаемся к ней:

```
postgres=# CREATE DATABASE vac;
CREATE DATABASE
postgres=# \c vac
You are now connected to database "vac" as user "postgres".
```

Создадим таблицу с текстовым полем и заполним её случайными данными в размере 1 миллион строк.

```
vac=# CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT
);
CREATE TABLE
vac=# INSERT INTO products (name) SELECT md5(random()::text) FROM generate_series(1, 1000000);
INSERT 0 1000000
```

## 2. Посмотреть размер файла с таблицей

Смотрим размер таблицы при помощи функции pg_size_pretty для читабельности, а также pg_total_relation_size возвращает общий объём, который занимает на диске таблица `products`.

```
vac=# SELECT pg_size_pretty(pg_total_relation_size('products'));
 pg_size_pretty
----------------
 87 MB
(1 row)
```

## 3. 5 раз обновить все строчки и добавить к каждой строчке любой символ
Повторяем 5 раз
```
vac=# UPDATE products SET name = CONCAT(name,'s');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name,'s');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name,'s');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name,'s');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name,'s');
UPDATE 1000000
```

## 4. Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум

Последний автовакуум приходит 2 минуты назад, количество мертвых строк - 4000000
```
vac=# SELECT n_dead_tup, last_autovacuum FROM pg_stat_user_tables WHERE relname = 'products';
 n_dead_tup |        last_autovacuum
------------+-------------------------------
    5000000 | 2024-10-14 20:33:47.805443+03
(1 row)
```

Пороговые значения сработали для автовакуума:

1) от 50 мертвых строк
2) если количество мертвых строк превышает на 20% количество строк в таблице.
 
```
vac=# SHOW autovacuum_vacuum_threshold;
 autovacuum_vacuum_threshold
-----------------------------
 50
(1 row)
vac=# SHOW autovacuum_vacuum_scale_factor;
 autovacuum_vacuum_scale_factor
--------------------------------
 0.2
(1 row)

```


## 5. Подождать некоторое время, проверяя, пришел ли автовакуум

После ожидания автовакуум пришел и мертных строк стало 0.
```
vac=# SELECT n_dead_tup, last_autovacuum FROM pg_stat_user_tables WHERE relname = 'products';
 n_dead_tup |        last_autovacuum
------------+-------------------------------
          0 | 2024-10-14 20:35:02.239462+03
(1 row)
```

## 6. 5 раз обновить все строчки и добавить к каждой строчке любой символ

Повторяем 5 раз
```
vac=# UPDATE products SET name = CONCAT(name,'s');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name,'s');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name,'s');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name,'s');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name,'s');
UPDATE 1000000
```

## 7. Посмотреть размер файла с таблицей

```
vac=# SELECT pg_size_pretty(pg_total_relation_size('products'));
 pg_size_pretty
----------------
 524 MB
(1 row)
```

## 8. Отключить Автовакуум на конкретной таблице

```
vac=# ALTER TABLE products SET (autovacuum_enabled = off);
ALTER TABLE
```

## 9. 10 раз обновить все строчки и добавить к каждой строчке любой символ

```
vac=# UPDATE products SET name = CONCAT(name,'n');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name,'n');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name,'n');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name,'n');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name,'n');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name,'n');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name,'n');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name,'n');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name,'n');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name,'n');
UPDATE 1000000
```

## 10. Посмотреть размер файла с таблицей

```
vac=# SELECT pg_size_pretty(pg_total_relation_size('products'));
 pg_size_pretty
----------------
 1008 MB
(1 row)
```

## 11. Объясните полученный результат

Когда мы добавили символ к строке, это привело к созданию еще одной строки с символом, а старая при этом не используется, поэтому стала мертвой. А мертвые строки подчищаются автовакуумом, а на этот раз мы его отключили.

```
vac=# SELECT n_dead_tup, last_autovacuum FROM pg_stat_user_tables WHERE relname = 'products';
 n_dead_tup |        last_autovacuum
------------+-------------------------------
   10000000 | 2024-10-14 20:39:11.032505+03
(1 row)
```

## 12. Не забудьте включить автовакуум)

Включаем обратно автовакуум.

```
vac=# ALTER TABLE products SET (autovacuum_enabled = on);
ALTER TABLE
```

Инициировала запуск вакуума

```
vac=# VACUUM products;
VACUUM
vac=# SELECT n_dead_tup, last_autovacuum FROM pg_stat_user_tables WHERE relname = 'products';
 n_dead_tup |        last_autovacuum
------------+-------------------------------
          0 | 2024-10-14 20:39:11.032505+03
(1 row)


После автовакуума таблица не почистилась
```
vac=#  SELECT pg_size_pretty(pg_total_relation_size('products'));
 pg_size_pretty
----------------
 1008 MB
(1 row)
```

После автовакуума размер таблицы не поменялся, а мертвые строки очистились. Это значит, что процесс очистки autovacuum, или команда VACUUM, пробегает по изменённым страницам и помечает такое место как свободное, после чего новые записи могут спокойно записываться в это место - размер файла таблицы физически не уменьшается.

Добавим еще 5 строк к таблице и проверим размер

```
vac=#  SELECT pg_size_pretty(pg_total_relation_size('products'));
 pg_size_pretty
----------------
 1008 MB
(1 row)

vac=# UPDATE products SET name = CONCAT(name, '%');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name, '%');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name, '%');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name, '%');
UPDATE 1000000
vac=# UPDATE products SET name = CONCAT(name, '%');
UPDATE 1000000

vac=#  SELECT pg_size_pretty(pg_total_relation_size('products'));
 pg_size_pretty
----------------
 1008 MB
(1 row)
```

Заметили, что новые символы добавились в пустое пространство и поэтому размер таблицы не поменялся

Чтобы почистить таблицу максимально, можно запустить VACUUM FULL

```
vac=# VACUUM FULL ANALYZE products;
VACUUM
vac=#  SELECT pg_size_pretty(pg_total_relation_size('products'));
 pg_size_pretty
----------------
 110 MB
(1 row)
```