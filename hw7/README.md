Запрос
```
EXPLAIN ANALYZE
WITH all_place AS (
    SELECT
        COUNT(s.id) AS all_place,
        s.fkbus     AS fkbus
    FROM book.seat s
    GROUP BY s.fkbus
),
order_place AS (
    SELECT
        COUNT(t.id) AS order_place,
        t.fkride    AS fkride
    FROM book.tickets t
    GROUP BY t.fkride
)
SELECT
    r.id,
    r.startdate                AS depart_date,
    bs.city || ', ' || bs.name AS busstation,
    t.order_place,
    st.all_place
FROM book.ride r
JOIN book.schedule AS s
    ON r.**fkschedule** = s.id
JOIN book.busroute br
    ON s.**fkroute** = br.id
JOIN book.busstation bs
    ON br.**fkbusstationfrom** = bs.id
JOIN order_place t
    ON t.**fkride** = r.id
JOIN all_place st
    ON r.**fkbus** = st.fkbus
GROUP BY
    r.id,
    r.startdate,
    bs.city || ', ' || bs.name,
    t.order_place,
    st.all_place
ORDER BY
    r.startdate
LIMIT 10;


Изначальный план выполнения

                                                                                           QUERY PLAN                                                                             
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=346683.94..**346683.97** rows=10 width=56) (actual time=1077.011..1077.109 rows=10 loops=1)
   ->  Sort  (cost=346683.94..347045.96 rows=144808 width=56) (actual time=1055.077..1055.173 rows=10 loops=1)
         Sort Key: r.startdate
         Sort Method: top-N heapsort  Memory: 25kB
         ->  Group  (cost=341020.55..343554.69 rows=144808 width=56) (actual time=995.084..1032.643 rows=144000 loops=1)
               Group Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
               ->  Sort  (cost=341020.55..341382.57 rows=144808 width=56) (actual time=995.060..1007.885 rows=144000 loops=1)
                     Sort Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
                     Sort Method: external merge  Disk: 7864kB
                     ->  Hash Join  (cost=276825.64..323655.27 rows=144808 width=56) (actual time=730.169..962.457 rows=144000 loops=1)
                           Hash Cond: (r.fkbus = s_1.fkbus)
                           ->  Hash Join  (cost=276820.53..322223.79 rows=144808 width=84) (actual time=729.992..928.632 rows=144000 loops=1)
                                 Hash Cond: (r.fkschedule = s.id)
                                 ->  Merge Join  (cost=276764.13..320176.28 rows=144808 width=24) (actual time=728.960..901.167 rows=144000 loops=1)
                                       Merge Cond: (r.id = t.fkride)
                                       ->  Index Scan using ride_pkey on ride r  (cost=0.42..4555.42 rows=144000 width=16) (actual time=0.059..25.581 rows=144000 loops=1)
                                       ->  Finalize GroupAggregate  (cost=276763.71..313450.76 rows=144808 width=12) (actual time=728.887..841.779 rows=144000 loops=1)
                                             Group Key: t.fkride
                                             ->  Gather Merge  (cost=276763.71..310554.60 rows=289616 width=12) (actual time=728.849..794.546 rows=432000 loops=1)
                                                   Workers Planned: 2
                                                   Workers Launched: 2
                                                   ->  Sort  (cost=275763.68..276125.70 rows=144808 width=12) (actual time=714.112..726.107 rows=144000 loops=3)
                                                         Sort Key: t.fkride
                                                         Sort Method: external merge  Disk: 3672kB
                                                         Worker 0:  Sort Method: external merge  Disk: 3672kB
                                                         Worker 1:  Sort Method: external merge  Disk: 3672kB
                                                         ->  Partial HashAggregate  (cost=236625.68..260872.90 rows=144808 width=12) (actual time=556.612..673.866 rows=144000 loops=3)
                                                               Group Key: t.fkride
                                                               Planned Partitions: 4  Batches: 5  Memory Usage: 8241kB  Disk Usage: 27808kB
                                                               Worker 0:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 27864kB
                                                               Worker 1:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 27840kB
                                                               ->  Parallel Seq Scan on tickets t  (cost=0.00..87063.32 rows=2334632 width=12) (actual time=0.038..157.973 rows=1868718 loops=3)
                                 ->  Hash  (cost=38.40..38.40 rows=1440 width=68) (actual time=1.017..1.022 rows=1440 loops=1)
                                       Buckets: 2048  Batches: 1  Memory Usage: 91kB
                                       ->  Hash Join  (cost=3.58..38.40 rows=1440 width=68) (actual time=0.094..0.731 rows=1440 loops=1)
                                             Hash Cond: (br.fkbusstationfrom = bs.id)
                                             ->  Hash Join  (cost=2.35..31.80 rows=1440 width=8) (actual time=0.057..0.455 rows=1440 loops=1)
                                                   Hash Cond: (s.fkroute = br.id)
                                                   ->  Seq Scan on schedule s  (cost=0.00..25.40 rows=1440 width=8) (actual time=0.014..0.149 rows=1440 loops=1)
                                                   ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.031..0.033 rows=60 loops=1)
                                                         Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                                         ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.010..0.017 rows=60 loops=1)
                                             ->  Hash  (cost=1.10..1.10 rows=10 width=68) (actual time=0.027..0.027 rows=10 loops=1)

JOIN all_place st
    ON r.fkbus = st.fkbus

**Создадим индекс на fkbus:**

thai=# \d book.seat;
                                      Table "book.seat"
     Column     |     Type     | Collation | Nullable |                Default
----------------+--------------+-----------+----------+---------------------------------------
 id             | integer      |           | not null | nextval('book.seat_id_seq'::regclass)
 fkbus          | integer      |           |          |
 place          | character(3) |           |          |
 fkseatcategory | integer      |           |          |
Indexes:
    "seat_pkey" PRIMARY KEY, btree (id)
    **"seat_fkbus_idx" btree (fkbus)**
Foreign-key constraints:
    "seat_fkbus_fkey" FOREIGN KEY (fkbus) REFERENCES book.bus(id)
    "seat_fkseatcategory_fkey" FOREIGN KEY (fkseatcategory) REFERENCES book.seatcategory(id)
Referenced by:
    TABLE "book.tickets" CONSTRAINT "tickets_fkseat_fkey" FOREIGN KEY (fkseat) REFERENCES book.seat(id)

                                                                                              QUERY PLAN                                                                          
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=327292.65..**327292.68** rows=10 width=56) (actual time=1120.879..1120.974 rows=10 loops=1)
   ->  Sort  (cost=327292.65..327558.78 rows=106451 width=56) (actual time=1103.764..1103.858 rows=10 loops=1)
         Sort Key: r.startdate
         Sort Method: top-N heapsort  Memory: 25kB
         ->  Group  (cost=323129.39..324992.29 rows=106451 width=56) (actual time=1042.745..1081.033 rows=144000 loops=1)
               Group Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
               ->  Sort  (cost=323129.39..323395.52 rows=106451 width=56) (actual time=1042.722..1055.703 rows=144000 loops=1)
                     Sort Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
                     Sort Method: external merge  Disk: 7864kB
                     ->  Hash Join  (cost=272360.45..310600.83 rows=106451 width=56) (actual time=728.936..1009.416 rows=144000 loops=1)
                           Hash Cond: (r.fkbus = s_1.fkbus)
                           ->  Nested Loop  (cost=272355.33..309547.17 rows=106451 width=36) (actual time=728.849..974.247 rows=144000 loops=1)
                                 ->  Hash Join  (cost=272355.19..307012.85 rows=106451 width=24) (actual time=728.809..925.633 rows=144000 loops=1)
                                       Hash Cond: (r.fkschedule = s.id)
                                       ->  Merge Join  (cost=272305.39..305499.35 rows=106451 width=24) (actual time=728.319..900.673 rows=144000 loops=1)
                                             Merge Cond: (r.id = t.fkride)
                                             ->  Index Scan using ride_pkey on ride r  (cost=0.42..4534.42 rows=144000 width=16) (actual time=0.012..21.508 rows=144000 loops=1)
                                             ->  Finalize GroupAggregate  (cost=272304.97..299274.29 rows=106451 width=12) (actual time=728.295..844.768 rows=144000 loops=1)
                                                   Group Key: t.fkride
                                                   ->  Gather Merge  (cost=272304.97..297145.27 rows=212902 width=12) (actual time=728.246..796.073 rows=432000 loops=1)
                                                         Workers Planned: 2
                                                         Workers Launched: 2
                                                         ->  Sort  (cost=271304.95..271571.07 rows=106451 width=12) (actual time=707.721..719.742 rows=144000 loops=3)
                                                               Sort Key: t.fkride
                                                               Sort Method: external merge  Disk: 3672kB
                                                               Worker 0:  Sort Method: external merge  Disk: 3672kB
                                                               Worker 1:  Sort Method: external merge  Disk: 3672kB
                                                               ->  Partial HashAggregate  (cost=236720.26..260596.38 rows=106451 width=12) (actual time=546.087..667.561 rows=144000 loops=3)
                                                                     Group Key: t.fkride
                                                                     Batches: 5  Memory Usage: 8241kB  Disk Usage: 27904kB
                                                                     Worker 0:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 28488kB
                                                                     Worker 1:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 27952kB
                                                                     ->  Parallel Seq Scan on tickets t  (cost=0.00..87076.09 rows=2335909 width=12) (actual time=0.030..137.756 rows=1868718 loops=3)
                                       ->  Hash  (cost=31.80..31.80 rows=1440 width=8) (actual time=0.478..0.480 rows=1440 loops=1)
                                             Buckets: 2048  Batches: 1  Memory Usage: 73kB
                                             ->  Hash Join  (cost=2.35..31.80 rows=1440 width=8) (actual time=0.033..0.320 rows=1440 loops=1)
                                                   Hash Cond: (s.fkroute = br.id)
                                                   ->  Seq Scan on schedule s  (cost=0.00..25.40 rows=1440 width=8) (actual time=0.004..0.081 rows=1440 loops=1)
                                                   ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.020..0.021 rows=60 loops=1)
                                                         Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                                         ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.006..0.011 rows=60 loops=1)
                                 ->  Memoize  (cost=0.15..0.36 rows=1 width=20) (actual time=0.000..0.000 rows=1 loops=144000)
                                       Cache Key: br.fkbusstationfrom

JOIN order_place t
    ON t.fkride = r.id

**Создадим индекс на fkbus:**
thai=# \d book.tickets;
                                Table "book.tickets"
 Column  |  Type   | Collation | Nullable |                 Default
---------+---------+-----------+----------+------------------------------------------
 id      | bigint  |           | not null | nextval('book.tickets_id_seq'::regclass)
 fkride  | integer |           |          |
 fio     | text    |           |          |
 contact | jsonb   |           |          |
 fkseat  | integer |           |          |
Indexes:
    "tickets_pkey" PRIMARY KEY, btree (id)
    **"tickets_fkride_idx" btree (fkride)**
Foreign-key constraints:
    "tickets_fkride_fkey" FOREIGN KEY (fkride) REFERENCES book.ride(id)
    "tickets_fkseat_fkey" FOREIGN KEY (fkseat) REFERENCES book.seat(id)


                                                                                              QUERY PLAN                                                                          
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=328274.78..**328274.80** rows=10 width=56) (actual time=1105.567..1105.658 rows=10 loops=1)
   ->  Sort  (cost=328274.78..328544.99 rows=108084 width=56) (actual time=1089.982..1090.072 rows=10 loops=1)
         Sort Key: r.startdate
         Sort Method: top-N heapsort  Memory: 25kB
         ->  Group  (cost=324047.65..325939.12 rows=108084 width=56) (actual time=1029.670..1067.626 rows=144000 loops=1)
               Group Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
               ->  Sort  (cost=324047.65..324317.86 rows=108084 width=56) (actual time=1029.644..1042.467 rows=144000 loops=1)
                     Sort Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
                     Sort Method: external merge  Disk: 7864kB
                     ->  Hash Join  (cost=272562.98..311314.86 rows=108084 width=56) (actual time=713.814..998.727 rows=144000 loops=1)
                           Hash Cond: (r.fkbus = s_1.fkbus)
                           ->  Nested Loop  (cost=272557.86..310245.12 rows=108084 width=36) (actual time=713.727..962.881 rows=144000 loops=1)
                                 ->  Hash Join  (cost=272557.72..307671.96 rows=108084 width=24) (actual time=713.691..913.471 rows=144000 loops=1)
                                       Hash Cond: (r.fkschedule = s.id)
                                       ->  Merge Join  (cost=272507.92..306136.01 rows=108084 width=24) (actual time=713.199..888.382 rows=144000 loops=1)
                                             Merge Cond: (r.id = t.fkride)
                                             ->  Index Scan using ride_pkey on ride r  (cost=0.42..4534.42 rows=144000 width=16) (actual time=0.013..21.842 rows=144000 loops=1)
                                             ->  Finalize GroupAggregate  (cost=272507.50..299890.54 rows=108084 width=12) (actual time=713.175..831.219 rows=144000 loops=1)
                                                   Group Key: t.fkride
                                                   ->  Gather Merge  (cost=272507.50..297728.86 rows=216168 width=12) (actual time=713.138..782.128 rows=431992 loops=1)
                                                         Workers Planned: 2
                                                         Workers Launched: 2
                                                         ->  Sort  (cost=271507.48..271777.69 rows=108084 width=12) (actual time=692.118..704.847 rows=143997 loops=3)
                                                               Sort Key: t.fkride
                                                               Sort Method: external merge  Disk: 3672kB
                                                               Worker 0:  Sort Method: external merge  Disk: 3672kB
                                                               Worker 1:  Sort Method: external merge  Disk: 3672kB
                                                               ->  Partial HashAggregate  (cost=236729.07..260622.69 rows=108084 width=12) (actual time=528.702..652.617 rows=143997 loops=3)
                                                                     Group Key: t.fkride
                                                                     Batches: 5  Memory Usage: 8241kB  Disk Usage: 27848kB
                                                                     Worker 0:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 30648kB
                                                                     Worker 1:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 28200kB
                                                                     ->  Parallel Seq Scan on tickets t  (cost=0.00..87077.28 rows=2336028 width=12) (actual time=0.025..136.272 rows=1868718 loops=3)
                                       ->  Hash  (cost=31.80..31.80 rows=1440 width=8) (actual time=0.480..0.482 rows=1440 loops=1)
                                             Buckets: 2048  Batches: 1  Memory Usage: 73kB
                                             ->  Hash Join  (cost=2.35..31.80 rows=1440 width=8) (actual time=0.034..0.322 rows=1440 loops=1)
                                                   Hash Cond: (s.fkroute = br.id)
                                                   ->  Seq Scan on schedule s  (cost=0.00..25.40 rows=1440 width=8) (actual time=0.004..0.080 rows=1440 loops=1)
                                                   ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.021..0.022 rows=60 loops=1)
                                                         Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                                         ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.007..0.011 rows=60 loops=1)
                                 ->  Memoize  (cost=0.15..0.36 rows=1 width=20) (actual time=0.000..0.000 rows=1 loops=144000)
                                       Cache Key: br.fkbusstationfrom

Планы значимо не изменились после внедрения индексов на внешние ключи таблиц из CTE, скорее всего это из-за того что все строки попадают в агрегаты (COUNT(...)). В результате планировщик не использует недавно созданные индексы — они не дают прироста при сканировании всей таблицы целиком.

Создадим индексы по другим JOIN-столбцам

CREATE INDEX ON book.ride (fkschedule);
CREATE INDEX ON book.schedule (fkroute);
CREATE INDEX ON book.busroute (fkbusstationfrom);

thai=# \d book.ride
                                  Table "book.ride"
   Column   |  Type   | Collation | Nullable |                Default
------------+---------+-----------+----------+---------------------------------------
 id         | integer |           | not null | nextval('book.ride_id_seq'::regclass)
 startdate  | date    |           |          |
 fkbus      | integer |           |          |
 fkschedule | integer |           |          |
Indexes:
    "ride_pkey" PRIMARY KEY, btree (id)
    **"ride_fkschedule_idx" btree (fkschedule)**
Foreign-key constraints:
    "ride_fkbus_fkey" FOREIGN KEY (fkbus) REFERENCES book.bus(id)
    "ride_fkschedule_fkey" FOREIGN KEY (fkschedule) REFERENCES book.schedule(id)
Referenced by:
    TABLE "book.tickets" CONSTRAINT "tickets_fkride_fkey" FOREIGN KEY (fkride) REFERENCES book.ride(id)

thai=# \d book.schedule
                                         Table "book.schedule"
  Column   |          Type          | Collation | Nullable |                  Default
-----------+------------------------+-----------+----------+-------------------------------------------
 id        | integer                |           | not null | nextval('book.schedule_id_seq'::regclass)
 fkroute   | integer                |           |          |
 starttime | time without time zone |           |          |
 price     | numeric(17,2)          |           |          |
 validfrom | date                   |           |          |
 validto   | date                   |           |          |
Indexes:
    "schedule_pkey" PRIMARY KEY, btree (id)
    **"schedule_fkroute_idx" btree (fkroute)**
Foreign-key constraints:
    "schedule_fkroute_fkey" FOREIGN KEY (fkroute) REFERENCES book.busroute(id)
Referenced by:
    TABLE "book.ride" CONSTRAINT "ride_fkschedule_fkey" FOREIGN KEY (fkschedule) REFERENCES book.schedule(id)

thai=# \d book.busroute
                                     Table "book.busroute"
      Column      |   Type   | Collation | Nullable |                  Default
------------------+----------+-----------+----------+-------------------------------------------
 id               | integer  |           | not null | nextval('book.busroute_id_seq'::regclass)
 fkbusstationfrom | integer  |           |          |
 fkbusstationto   | integer  |           |          |
 distance         | integer  |           |          |
 duration         | interval |           |          |
Indexes:
    "busroute_pkey" PRIMARY KEY, btree (id)
    **"busroute_fkbusstationfrom_idx" btree (fkbusstationfrom)**
Foreign-key constraints:
    "busroute_fkbusstationfrom_fkey" FOREIGN KEY (fkbusstationfrom) REFERENCES book.busstation(id)
    "busroute_fkbusstationto_fkey" FOREIGN KEY (fkbusstationto) REFERENCES book.busstation(id)
Referenced by:
    TABLE "book.schedule" CONSTRAINT "schedule_fkroute_fkey" FOREIGN KEY (fkroute) REFERENCES book.busroute(id)


thai=# VACUUM ANALYZE ;
VACUUM
thai=#


                                                                                              QUERY PLAN                                                                          
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (**cost=326537.03..326537.06** rows=10 width=56) (actual time=1103.492..1103.584 rows=10 loops=1)
   ->  Sort  (cost=326537.03..326800.03 rows=105200 width=56) (actual time=1087.115..1087.206 rows=10 loops=1)
         Sort Key: r.startdate
         Sort Method: top-N heapsort  Memory: 25kB
         ->  Group  (cost=322422.70..324263.70 rows=105200 width=56) (actual time=1026.140..1064.381 rows=144000 loops=1)
               Group Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
               ->  Sort  (cost=322422.70..322685.70 rows=105200 width=56) (actual time=1026.116..1039.060 rows=144000 loops=1)
                     Sort Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
                     Sort Method: external merge  Disk: 7864kB
                     ->  Hash Join  (cost=272201.02..310049.56 rows=105200 width=56) (actual time=714.784..995.070 rows=144000 loops=1)
                           Hash Cond: (r.fkbus = s_1.fkbus)
                           ->  Nested Loop  (cost=272195.91..309008.23 rows=105200 width=36) (actual time=714.694..959.977 rows=144000 loops=1)
                                 ->  Hash Join  (cost=272195.76..306503.64 rows=105200 width=24) (actual time=714.657..911.599 rows=144000 loops=1)
                                       Hash Cond: (r.fkschedule = s.id)
                                       ->  Merge Join  (cost=272145.96..305007.35 rows=105200 width=24) (actual time=714.167..886.650 rows=144000 loops=1)
                                             Merge Cond: (r.id = t.fkride)
                                             ->  Index Scan using ride_pkey on ride r  (cost=0.42..4534.42 rows=144000 width=16) (actual time=0.014..21.208 rows=144000 loops=1)
                                             ->  Finalize GroupAggregate  (cost=272145.54..298797.93 rows=105200 width=12) (actual time=714.141..830.800 rows=144000 loops=1)
                                                   Group Key: t.fkride
                                                   ->  Gather Merge  (cost=272145.54..296693.93 rows=210400 width=12) (actual time=714.104..782.346 rows=431995 loops=1)
                                                         Workers Planned: 2
                                                         Workers Launched: 2
                                                         ->  Sort  (cost=271145.52..271408.52 rows=105200 width=12) (actual time=694.105..706.964 rows=143998 loops=3)
                                                               Sort Key: t.fkride
                                                               Sort Method: external merge  Disk: 3672kB
                                                               Worker 0:  Sort Method: external merge  Disk: 3672kB
                                                               Worker 1:  Sort Method: external merge  Disk: 3672kB
                                                               ->  Partial HashAggregate  (cost=236709.23..260571.38 rows=105200 width=12) (actual time=534.603..654.874 rows=143998 loops=3)
                                                                     Group Key: t.fkride
                                                                     Batches: 5  Memory Usage: 8241kB  Disk Usage: 28032kB
                                                                     Worker 0:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 26592kB
                                                                     Worker 1:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 28200kB
                                                                     ->  Parallel Seq Scan on tickets t  (cost=0.00..87074.60 rows=2335760 width=12) (actual time=0.028..139.259 rows=1868718 loops=3)
                                       ->  Hash  (cost=31.80..31.80 rows=1440 width=8) (actual time=0.478..0.480 rows=1440 loops=1)
                                             Buckets: 2048  Batches: 1  Memory Usage: 73kB
                                             ->  Hash Join  (cost=2.35..31.80 rows=1440 width=8) (actual time=0.035..0.321 rows=1440 loops=1)
                                                   Hash Cond: (s.fkroute = br.id)
                                                   ->  Seq Scan on schedule s  (cost=0.00..25.40 rows=1440 width=8) (actual time=0.003..0.080 rows=1440 loops=1)
                                                   ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.021..0.021 rows=60 loops=1)
                                                         Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                                         ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.006..0.012 rows=60 loops=1)
                                 ->  Memoize  (cost=0.15..0.36 rows=1 width=20) (actual time=0.000..0.000 rows=1 loops=144000)
                                       Cache Key: br.fkbusstationfrom

Построение индексов по внешним ключам не повлияло на план выполнение запроса, видно что запрос использует Seq, веротяно это из-за того что в запросе не используется селективная фильтрация и нужно считать агрегаты по большим объёмам данных, планировщик остаётся на последовательном сканировании и хеш-агрегатах

thai=# SET work_mem = '512MB';
SET
thai=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

thai=# SHOW work_mem ;
 work_mem
----------
 512MB
(1 row)

thai=#
                                                                                              QUERY PLAN                                                                          
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=135081.86..135081.89 rows=10 width=56) (actual time=965.320..965.421 rows=10 loops=1)
   ->  Sort  (cost=135081.86..135346.99 rows=106049 width=56) (actual time=938.409..938.508 rows=10 loops=1)
         Sort Key: r.startdate
         Sort Method: top-N heapsort  Memory: 26kB
         ->  HashAggregate  (cost=131199.45..132790.18 rows=106049 width=56) (actual time=890.120..917.032 rows=144000 loops=1)
               Group Key: r.id, ((bs.city || ', '::text) || bs.name), (count(t.id)), (count(s_1.id))
               Batches: 1  Memory Usage: 22545kB
               ->  Hash Join  (cost=125523.76..130138.96 rows=106049 width=56) (actual time=683.215..837.732 rows=144000 loops=1)
                     Hash Cond: (r.fkbus = s_1.fkbus)
                     ->  Hash Join  (cost=125518.65..129089.26 rows=106049 width=36) (actual time=683.074..807.139 rows=144000 loops=1)
                           Hash Cond: (br.fkbusstationfrom = bs.id)
                           ->  Hash Join  (cost=125517.43..128691.68 rows=106049 width=24) (actual time=683.044..785.598 rows=144000 loops=1)
                                 Hash Cond: (s.fkroute = br.id)
                                 ->  Hash Join  (cost=125515.08..128391.29 rows=106049 width=24) (actual time=683.001..764.630 rows=144000 loops=1)
                                       Hash Cond: (r.fkschedule = s.id)
                                       ->  Hash Join  (cost=125471.68..128068.69 rows=106049 width=24) (actual time=682.473..742.228 rows=144000 loops=1)
                                             Hash Cond: (r.id = t.fkride)
                                             ->  Seq Scan on ride r  (cost=0.00..2219.00 rows=144000 width=16) (actual time=0.008..9.631 rows=144000 loops=1)
                                             ->  Hash  (cost=124146.06..124146.06 rows=106049 width=12) (actual time=682.351..682.443 rows=144000 loops=1)
                                                   Buckets: 262144 (originally 131072)  Batches: 1 (originally 1)  Memory Usage: 8236kB
                                                   ->  Finalize HashAggregate  (cost=123085.57..124146.06 rows=106049 width=12) (actual time=638.778..660.978 rows=144000 loops=1)
                                                         Group Key: t.fkride
                                                         Batches: 1  Memory Usage: 22545kB
                                                         ->  Gather  (cost=99754.79..122025.08 rows=212098 width=12) (actual time=485.041..534.669 rows=432000 loops=1)
                                                               Workers Planned: 2
                                                               Workers Launched: 2
                                                               ->  Partial HashAggregate  (cost=98754.79..99815.28 rows=106049 width=12) (actual time=470.002..494.097 rows=144000 loops=3)
                                                                     Group Key: t.fkride
                                                                     Batches: 1  Memory Usage: 22545kB
                                                                     Worker 0:  Batches: 1  Memory Usage: 22545kB
                                                                     Worker 1:  Batches: 1  Memory Usage: 22545kB
                                                                     ->  Parallel Seq Scan on tickets t  (cost=0.00..87075.53 rows=2335853 width=12) (actual time=0.022..141.806 rows=1868718 loops=3)
                                       ->  Hash  (cost=25.40..25.40 rows=1440 width=8) (actual time=0.515..0.515 rows=1440 loops=1)
                                             Buckets: 2048  Batches: 1  Memory Usage: 73kB
                                             ->  Seq Scan on schedule s  (cost=0.00..25.40 rows=1440 width=8) (actual time=0.008..0.204 rows=1440 loops=1)
                                 ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.033..0.033 rows=60 loops=1)
                                       Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                       ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.007..0.016 rows=60 loops=1)
                           ->  Hash  (cost=1.10..1.10 rows=10 width=20) (actual time=0.018..0.019 rows=10 loops=1)
                                 Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                 ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=20) (actual time=0.009..0.011 rows=10 loops=1)
                     ->  Hash  (cost=5.05..5.05 rows=5 width=12) (actual time=0.124..0.125 rows=5 loops=1)
                           Buckets: 1024  Batches: 1  Memory Usage: 9kB

Увеличение work_mem помогло оптимизирвоать запрос но планировщик также не выбирает индексы по внешним ключам

ПОпробуем добавить условия и посмотерть план

EXPLAIN ANALYZE
WITH all_place AS (
    SELECT
        COUNT(s.id) AS all_place,
        s.fkbus     AS fkbus
    FROM book.seat s
	WHERE s.fkbus = 1
    GROUP BY s.fkbus
),
order_place AS (
    SELECT
        COUNT(t.id) AS order_place,
        t.fkride    AS fkride
    FROM book.tickets t
    GROUP BY t.fkride
)
SELECT
    r.id,
    r.startdate                AS depart_date,
    bs.city || ', ' || bs.name AS busstation,
    t.order_place,
    st.all_place
FROM book.ride r
JOIN book.schedule AS s
    ON r.fkschedule = s.id
JOIN book.busroute br
    ON s.fkroute = br.id
JOIN book.busstation bs
    ON br.fkbusstationfrom = bs.id
JOIN order_place t
    ON t.fkride = r.id
JOIN all_place st
    ON r.fkbus = st.fkbus
WHERE r.fkbus = 1
GROUP BY
    r.id,
    r.startdate,
    bs.city || ', ' || bs.name,
    t.order_place,
    st.all_place
ORDER BY
    r.startdate
LIMIT 10;


                                                                                              QUERY PLAN                                                                          
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=130747.73..130747.76 rows=10 width=56) (actual time=768.444..768.564 rows=10 loops=1)
   ->  Sort  (cost=130747.73..130836.68 rows=35579 width=56) (actual time=751.847..751.967 rows=10 loops=1)
         Sort Key: r.startdate
         Sort Method: top-N heapsort  Memory: 26kB
         ->  HashAggregate  (cost=129445.20..129978.88 rows=35579 width=56) (actual time=736.662..744.849 rows=48017 loops=1)
               Group Key: r.id, ((bs.city || ', '::text) || bs.name), (count(t.id)), (count(s_1.id))
               Batches: 1  Memory Usage: 5649kB
               ->  Nested Loop  (cost=125518.65..129089.41 rows=35579 width=56) (actual time=666.951..721.334 rows=48017 loops=1)
                     ->  GroupAggregate  (cost=0.00..4.61 rows=1 width=12) (actual time=0.044..0.045 rows=1 loops=1)
                           ->  Seq Scan on seat s_1  (cost=0.00..4.50 rows=40 width=8) (actual time=0.023..0.034 rows=40 loops=1)
                                 Filter: (fkbus = 1)
                                 Rows Removed by Filter: 160
                     ->  Hash Join  (cost=125518.65..128551.11 rows=35579 width=36) (actual time=666.892..713.506 rows=48017 loops=1)
                           Hash Cond: (br.fkbusstationfrom = bs.id)
                           ->  Hash Join  (cost=125517.43..128416.91 rows=35579 width=24) (actual time=666.864..706.028 rows=48017 loops=1)
                                 Hash Cond: (s.fkroute = br.id)
                                 ->  Hash Join  (cost=125515.08..128314.57 rows=35579 width=24) (actual time=666.834..698.817 rows=48017 loops=1)
                                       Hash Cond: (r.fkschedule = s.id)
                                       ->  Hash Join  (cost=125471.68..128177.50 rows=35579 width=24) (actual time=666.546..691.042 rows=48017 loops=1)
                                             Hash Cond: (r.id = t.fkride)
                                             ->  Seq Scan on ride r  (cost=0.00..2579.00 rows=48312 width=16) (actual time=0.006..10.680 rows=48017 loops=1)
                                                   Filter: (fkbus = 1)
                                                   Rows Removed by Filter: 95983
                                             ->  Hash  (cost=124146.06..124146.06 rows=106049 width=12) (actual time=666.434..666.546 rows=144000 loops=1)
                                                   Buckets: 262144 (originally 131072)  Batches: 1 (originally 1)  Memory Usage: 8236kB
                                                   ->  Finalize HashAggregate  (cost=123085.57..124146.06 rows=106049 width=12) (actual time=623.494..646.738 rows=144000 loops=1)
                                                         Group Key: t.fkride
                                                         Batches: 1  Memory Usage: 22545kB
                                                         ->  Gather  (cost=99754.79..122025.08 rows=212098 width=12) (actual time=471.049..521.289 rows=432000 loops=1)
                                                               Workers Planned: 2
                                                               Workers Launched: 2
                                                               ->  Partial HashAggregate  (cost=98754.79..99815.28 rows=106049 width=12) (actual time=456.728..480.991 rows=144000 loops=3)
                                                                     Group Key: t.fkride
                                                                     Batches: 1  Memory Usage: 22545kB
                                                                     Worker 0:  Batches: 1  Memory Usage: 22545kB
                                                                     Worker 1:  Batches: 1  Memory Usage: 22545kB
                                                                     ->  Parallel Seq Scan on tickets t  (cost=0.00..87075.53 rows=2335853 width=12) (actual time=0.031..139.278 rows=1868718 loops=3)
                                       ->  Hash  (cost=25.40..25.40 rows=1440 width=8) (actual time=0.278..0.278 rows=1440 loops=1)
                                             Buckets: 2048  Batches: 1  Memory Usage: 73kB
                                             ->  Seq Scan on schedule s  (cost=0.00..25.40 rows=1440 width=8) (actual time=0.006..0.111 rows=1440 loops=1)
                                 ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.021..0.021 rows=60 loops=1)
                                       Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                       ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.006..0.011 rows=60 loops=1)
```
