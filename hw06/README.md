# Домашнее задание №6

На уже установленном pg с прошлого домашнего задания:

### 1) Под postgres заливаем данные

```
Объем порядка 6 млн.строк (600МБ):
wget https://storage.googleapis.com/thaibus/thai_small.tar.gz && tar -xf thai_small.tar.gz && psql < thai.sql
```

### 2) Список бд

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

### 3) Использование бд thai

```
postgres-# \c thai
You are now connected to database "thai" as user "postgres".
```

### 4) Выполняем сложный запрос без индексов, чтобы проверить время выполнения

```
thai=# EXPLAIN ANALYZE WITH all_place AS (
    SELECT count(s.id) as all_place, s.fkbus as fkbus
    FROM book.seat s
    group by s.fkbus
),
order_place AS (
    SELECT count(t.id) as order_place, t.fkride
    FROM book.tickets t
    group by t.fkride
)
SELECT r.id, r.startdate as depart_date, bs.city || ', ' || bs.name as busstation,
      t.order_place, st.all_place
FROM book.ride r
JOIN book.schedule as s
      on r.fkschedule = s.id
JOIN book.busroute br
      on s.fkroute = br.id
JOIN book.busstation bs
      on br.fkbusstationfrom = bs.id
JOIN order_place t
      on t.fkride = r.id
JOIN all_place st
      on r.fkbus = st.fkbus
GROUP BY r.id, r.startdate, bs.city || ', ' || bs.name, t.order_place,st.all_place
ORDER BY r.startdate
limit 10;
```

```
                                                                                                 QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=94730.09..94730.11 rows=10 width=56) (actual time=2576.496..2579.224 rows=10 loops=1)
   ->  Sort  (cost=94730.09..94730.59 rows=200 width=56) (actual time=2576.495..2579.222 rows=10 loops=1)
         Sort Key: r.startdate
         Sort Method: top-N heapsort  Memory: 25kB
         ->  Group  (cost=92362.60..94725.76 rows=200 width=56) (actual time=1327.234..2554.546 rows=144000 loops=1)
               Group Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
               ->  Incremental Sort  (cost=92362.60..94722.76 rows=200 width=56) (actual time=1327.232..2528.484 rows=144000 loops=1)
                     Sort Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
                     Presorted Key: r.id
                     Full-sort Groups: 4500  Sort Method: quicksort  Average Memory: 27kB  Peak Memory: 27kB
                     ->  Nested Loop  (cost=92350.77..94713.76 rows=200 width=56) (actual time=1326.824..2492.393 rows=144000 loops=1)
                           Join Filter: (r.fkbus = s_1.fkbus)
                           Rows Removed by Join Filter: 336602
                           ->  Nested Loop  (cost=92345.77..94106.23 rows=200 width=84) (actual time=1326.732..2407.453 rows=144000 loops=1)
                                 Join Filter: (br.fkbusstationfrom = bs.id)
                                 Rows Removed by Join Filter: 590400
                                 ->  Nested Loop  (cost=92345.77..94075.48 rows=200 width=24) (actual time=1326.724..2308.306 rows=144000 loops=1)
                                       Join Filter: (s.fkroute = br.id)
                                       Rows Removed by Join Filter: 4248000
                                       ->  Nested Loop  (cost=92345.77..93896.34 rows=200 width=24) (actual time=1326.709..1870.683 rows=144000 loops=1)
                                             ->  Nested Loop  (cost=92345.49..93837.24 rows=200 width=24) (actual time=1326.675..1702.108 rows=144000 loops=1)
                                                   ->  Finalize GroupAggregate  (cost=92345.07..92395.74 rows=200 width=12) (actual time=1326.606..1476.527 rows=144000 loops=1)
                                                         Group Key: t.fkride
                                                         ->  Gather Merge  (cost=92345.07..92391.74 rows=400 width=12) (actual time=1326.595..1412.955 rows=431987 loops=1)
                                                               Workers Planned: 2
                                                               Workers Launched: 2
                                                               ->  Sort  (cost=91345.05..91345.55 rows=200 width=12) (actual time=1294.129..1311.978 rows=143996 loops=3)
                                                                     Sort Key: t.fkride
                                                                     Sort Method: external merge  Disk: 3672kB
                                                                     Worker 0:  Sort Method: external merge  Disk: 3672kB
                                                                     Worker 1:  Sort Method: external merge  Disk: 3672kB
                                                                     ->  Partial HashAggregate  (cost=91335.41..91337.41 rows=200 width=12) (actual time=958.509..1227.332 rows=143996 loops=3)
                                                                           Group Key: t.fkride
                                                                           Batches: 5  Memory Usage: 8257kB  Disk Usage: 32288kB
                                                                           Worker 0:  Batches: 5  Memory Usage: 8257kB  Disk Usage: 27520kB
                                                                           Worker 1:  Batches: 5  Memory Usage: 8257kB  Disk Usage: 32320kB
                                                                           ->  Parallel Seq Scan on tickets t  (cost=0.00..80532.27 rows=2160627 width=12) (actual time=0.038..245.966 rows=1728502 loops=3)
                                                   ->  Index Scan using ride_pkey on ride r  (cost=0.42..7.20 rows=1 width=16) (actual time=0.001..0.001 rows=1 loops=144000)
                                                         Index Cond: (id = t.fkride)
                                             ->  Index Scan using schedule_pkey on schedule s  (cost=0.28..0.30 rows=1 width=8) (actual time=0.001..0.001 rows=1 loops=144000)
                                                   Index Cond: (id = r.fkschedule)
                                       ->  Materialize  (cost=0.00..1.90 rows=60 width=8) (actual time=0.000..0.001 rows=30 loops=144000)
                                             ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.010..0.019 rows=60 loops=1)
                                 ->  Materialize  (cost=0.00..1.15 rows=10 width=68) (actual time=0.000..0.000 rows=5 loops=144000)
                                       ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=68) (actual time=0.005..0.007 rows=10 loops=1)
                           ->  Materialize  (cost=5.00..10.00 rows=200 width=12) (actual time=0.000..0.000 rows=3 loops=144000)
                                 ->  HashAggregate  (cost=5.00..7.00 rows=200 width=12) (actual time=0.084..0.086 rows=5 loops=1)
                                       Group Key: s_1.fkbus
                                       Batches: 1  Memory Usage: 40kB
                                       ->  Seq Scan on seat s_1  (cost=0.00..4.00 rows=200 width=8) (actual time=0.012..0.031 rows=200 loops=1)
 Planning Time: 1.164 ms
 Execution Time: 2583.978 ms
(52 rows)
```

> время выполнения - 2583.978 ms

### 5) Навесить индексы на внешние ключи

Находим внешние ключи по таблицам в запросе по 

FROM book.ride r
JOIN book.schedule as s
      on r.fkschedule = s.id
JOIN book.busroute br
      on s.fkroute = br.id
JOIN book.busstation bs
      on br.fkbusstationfrom = bs.id
JOIN order_place t
      on t.fkride = r.id
JOIN all_place st
      on r.fkbus = st.fkbus

```
\d book.busroute
Foreign-key constraints:
    "busroute_fkbusstationfrom_fkey" FOREIGN KEY (fkbusstationfrom) REFERENCES book.busstation(id)

\d book.schedule
Foreign-key constraints:
    "schedule_fkroute_fkey" FOREIGN KEY (fkroute) REFERENCES book.busroute(id)
Referenced by:
    TABLE "book.ride" CONSTRAINT "ride_fkschedule_fkey" FOREIGN KEY (fkschedule) REFERENCES book.schedule(id)

\d book.ride
Foreign-key constraints:
    "ride_fkbus_fkey" FOREIGN KEY (fkbus) REFERENCES book.bus(id)
    "ride_fkschedule_fkey" FOREIGN KEY (fkschedule) REFERENCES book.schedule(id)

\d book.tickets
Foreign-key constraints:
    "tickets_fkride_fkey" FOREIGN KEY (fkride) REFERENCES book.ride(id)
```

```
thai=# CREATE INDEX idx_seat_fkbus ON book.seat(fkbus);
CREATE INDEX
thai=# CREATE INDEX idx_tickets_fkride ON book.tickets(fkride);
CREATE INDEX
thai=# CREATE INDEX idx_ride_fkschedule ON book.ride(fkschedule);
CREATE INDEX
thai=# CREATE INDEX idx_busroute_fkbusstationfrom ON book.busroute(fkbusstationfrom);
CREATE INDEX
thai=# CREATE INDEX idx_ride_fkroute ON book.schedule(fkroute);
CREATE INDEX
```

### 6) Время выполнения после добавления индексов

```
                                                                                                 QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=328278.18..328278.20 rows=10 width=56) (actual time=1761.816..1761.932 rows=10 loops=1)
   ->  Sort  (cost=328278.18..328629.02 rows=140338 width=56) (actual time=1740.825..1740.939 rows=10 loops=1)
         Sort Key: r.startdate
         Sort Method: top-N heapsort  Memory: 25kB
         ->  Group  (cost=322789.61..325245.52 rows=140338 width=56) (actual time=1677.721..1717.046 rows=144000 loops=1)
               Group Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
               ->  Sort  (cost=322789.61..323140.45 rows=140338 width=56) (actual time=1677.695..1691.242 rows=144000 loops=1)
                     Sort Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
                     Sort Method: external merge  Disk: 7888kB
                     ->  Hash Join  (cost=256901.07..305993.23 rows=140338 width=56) (actual time=1305.026..1642.424 rows=144000 loops=1)
                           Hash Cond: (r.fkbus = s_1.fkbus)
                           ->  Nested Loop  (cost=256895.91..304605.73 rows=140338 width=84) (actual time=1304.929..1604.074 rows=144000 loops=1)
                                 ->  Hash Join  (cost=256895.76..301265.82 rows=140338 width=24) (actual time=1304.839..1550.585 rows=144000 loops=1)
                                       Hash Cond: (s.fkroute = br.id)
                                       ->  Hash Join  (cost=256893.41..300869.06 rows=140338 width=24) (actual time=1304.807..1524.082 rows=144000 loops=1)
                                             Hash Cond: (r.fkschedule = s.id)
                                             ->  Merge Join  (cost=256850.01..300456.20 rows=140338 width=24) (actual time=1304.498..1497.644 rows=144000 loops=1)
                                                   Merge Cond: (r.id = t.fkride)
                                                   ->  Index Scan using ride_pkey on ride r  (cost=0.42..4534.42 rows=144000 width=16) (actual time=0.012..23.015 rows=144000 loops=1)
                                                   ->  Finalize GroupAggregate  (cost=256849.59..292404.17 rows=140338 width=12) (actual time=1304.474..1436.869 rows=144000 loops=1)
                                                         Group Key: t.fkride
                                                         ->  Gather Merge  (cost=256849.59..289597.41 rows=280676 width=12) (actual time=1304.432..1386.141 rows=432000 loops=1)
                                                               Workers Planned: 2
                                                               Workers Launched: 2
                                                               ->  Sort  (cost=255849.57..256200.41 rows=140338 width=12) (actual time=1239.248..1254.653 rows=144000 loops=3)
                                                                     Sort Key: t.fkride
                                                                     Sort Method: external merge  Disk: 3672kB
                                                                     Worker 0:  Sort Method: external merge  Disk: 3672kB
                                                                     Worker 1:  Sort Method: external merge  Disk: 3672kB
                                                                     ->  Partial HashAggregate  (cost=218947.44..241450.69 rows=140338 width=12) (actual time=922.067..1168.860 rows=144000 loops=3)
                                                                           Group Key: t.fkride
                                                                           Planned Partitions: 4  Batches: 5  Memory Usage: 8241kB  Disk Usage: 31832kB
                                                                           Worker 0:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 32240kB
                                                                           Worker 1:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 31664kB
                                                                           ->  Parallel Seq Scan on tickets t  (cost=0.00..80532.27 rows=2160627 width=12) (actual time=0.030..238.093 rows=1728502 loops=3)
                                             ->  Hash  (cost=25.40..25.40 rows=1440 width=8) (actual time=0.296..0.297 rows=1440 loops=1)
                                                   Buckets: 2048  Batches: 1  Memory Usage: 73kB
                                                   ->  Seq Scan on schedule s  (cost=0.00..25.40 rows=1440 width=8) (actual time=0.006..0.133 rows=1440 loops=1)
                                       ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.021..0.021 rows=60 loops=1)
                                             Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                             ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.007..0.011 rows=60 loops=1)
                                 ->  Memoize  (cost=0.15..0.36 rows=1 width=68) (actual time=0.000..0.000 rows=1 loops=144000)
                                       Cache Key: br.fkbusstationfrom
                                       Cache Mode: logical
                                       Hits: 143990  Misses: 10  Evictions: 0  Overflows: 0  Memory Usage: 2kB
                                       ->  Index Scan using busstation_pkey on busstation bs  (cost=0.14..0.35 rows=1 width=68) (actual time=0.007..0.007 rows=1 loops=10)
                                             Index Cond: (id = br.fkbusstationfrom)
                           ->  Hash  (cost=5.10..5.10 rows=5 width=12) (actual time=0.082..0.083 rows=5 loops=1)
                                 Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                 ->  HashAggregate  (cost=5.00..5.05 rows=5 width=12) (actual time=0.076..0.077 rows=5 loops=1)
                                       Group Key: s_1.fkbus
                                       Batches: 1  Memory Usage: 24kB
                                       ->  Seq Scan on seat s_1  (cost=0.00..4.00 rows=200 width=8) (actual time=0.018..0.031 rows=200 loops=1)
 Planning Time: 1.795 ms
 JIT:
   Functions: 84
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 7.870 ms, Inlining 0.000 ms, Optimization 2.006 ms, Emission 47.258 ms, Total 57.134 ms
 Execution Time: 1788.081 ms
(59 rows)
```

> время выполнения - 1788.081 ms, видим, что время сократилось, навешивание индексов помогло
