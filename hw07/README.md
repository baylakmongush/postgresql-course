# Домашнее задание №7

На уже установленном pg с прошлого домашнего задания:

### 1) Создаем таблицу sales с продажами

```
postgres=# CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    date DATE,
    amount NUMERIC(10, 2),
    product VARCHAR(255)
);
CREATE TABLE
```

```
postgres=# \dt
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | sales | table | postgres
(1 row)
```

### 2) Реализуем функцию выбора трети года((1-4 месяц - первая треть, 5-8 - вторая и тд) с обработкой NULL

Функция get_third принимает дату date и возвращает номер трети года (1, 2 или 3). Есть также проверка на NULL, при котором функция также возвращает NULL.

```
postgres=# CREATE OR replace FUNCTION get_third(date DATE) RETURNS INTEGER AS $$
BEGIN
    RETURN CASE
        WHEN date IS NULL THEN NULL
        WHEN EXTRACT(MONTH FROM date) BETWEEN 1 AND 4 THEN 1
        WHEN EXTRACT(MONTH FROM date) BETWEEN 5 AND 8 THEN 2
        WHEN EXTRACT(MONTH FROM date) BETWEEN 9 AND 12 THEN 3
        ELSE NULL
    END;
END;
$$ LANGUAGE plpgsql;
CREATE FUNCTION
```

### 3) Сделаем проверку, что данные корректно вставляются и функция верно отрабатывает

```
postgres=# INSERT INTO sales (date, amount, product) VALUES (NULL, 100, 'Example Product');
INSERT 0 1
postgres=# SELECT id,
       date,
       amount,
       product,
       get_third(date) AS third_of_year
FROM sales;
 id | date | amount |     product     | third_of_year
----+------+--------+-----------------+------------
  1 |      | 100.00 | Example Product |
(1 row)

postgres=# INSERT INTO sales (date, amount, product) VALUES ('2024-11-11', 100, 'Example Product');
INSERT 0 1
postgres=# INSERT INTO sales (date, amount, product) VALUES ('2024-11-12', 100, 'Example Product');
INSERT 0 1
postgres=# INSERT INTO sales (date, amount, product) VALUES ('2024-11-13', 100, 'Example Product');
INSERT 0 1
postgres=# INSERT INTO sales (date, amount, product) VALUES ('2024-01-13', 100, 'Example Product');
INSERT 0 1
postgres=# INSERT INTO sales (date, amount, product) VALUES ('2024-05-13', 100, 'Example Product');
INSERT 0 1
postgres=# SELECT id,
       date,
       amount,
       product,
       get_third(date) AS third_of_year
FROM sales;
 id |    date    | amount |     product     | third_of_year
----+------------+--------+-----------------+------------
  1 |            | 100.00 | Example Product |
  2 | 2024-11-11 | 100.00 | Example Product |          3
  3 | 2024-11-12 | 100.00 | Example Product |          3
  4 | 2024-11-13 | 100.00 | Example Product |          3
  5 | 2024-01-13 | 100.00 | Example Product |          1
  6 | 2024-05-13 | 100.00 | Example Product |          2
(6 rows)
```

> видим, что месяцы корректно определяются по трети года и есть проверка на NULL