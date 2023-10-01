# Домашнее задание к занятию "Индексы" - Наталья Мартынова (Пономарева)

### Задание 1.

`Набросок:`
```
SELECT 
table_schema as `Database name`, 
table_name AS `Table name`, 
(select data_length) as `Size data`, 
(select index_length) as `Size index`, 
(select INDEX_LENGTH + DATA_LENGTH) as `Sum length`, 
(select (`size index` * 100)/ `Sum length`) as `value %`
FROM information_schema.TABLES 
WHERE TABLE_SCHEMA like 'sakila';
```
`Итог:`
```
select table_schema as `Database name`, sum(index_length) * 100 / sum(data_length)
FROM information_schema.TABLES 
WHERE TABLE_SCHEMA = 'sakila'
```
### Задание 2.

`Факторы влияющие на скорость выполнения запросов:`
- `индексы таблиц;`
- `условие WHERE(и использования внутренних функций MySQL, например, таких как IF или DATE);`
- `сортировка по ORDER BY;`
- `частое повторение одинаковых запросов;`
- `тип механизма хранения данных (InnoDB, MyISAM, Memory, Blackhole);`
- `не использование версии Percona;`
- `конфигурации сервера ( my.cnf / my.ini );`
- `большие выдачи данных (более 1000 строк);`
- `нестойкое соединение;`
- `распределенная или кластерная конфигурация;`
- `слабое проектирование таблиц.`

`Что бросается в глаза:`
1. `Работа с таблицей`
```
Table scan on <temporary>  (cost=2.50..2.50 rows=0) (actual time=12890.902..12890.965 rows=200 loops=1)
    -> Temporary table with deduplication  (cost=0.00..0.00 rows=0) (actual time=12890.897..12890.897 rows=391 loops=1)
```
 2. `Обрабатываются 642000 строк`
```
"sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=6313.444..12369.150 rows=642000 loops=1)"
```
`и тут`
```
-> Sort: c.customer_id, f.title  (actual time=6313.395..6501.155 rows=642000 loops=1)
```
3. `Запросы имеют очень высокую стоимость и обрабатывают большое количество строк`
```
Stream results  (cost=10794647.09 rows=16703446) (actual time=0.678..4783.897 rows=642000 loops=1)
    -> Nested loop inner join  (cost=10794647.09 rows=16703446) (actual time=0.670..4094.256 rows=642000 loops=1)
        -> Nested loop inner join  (cost=9120126.64 rows=16703446) (actual time=0.665..3675.354 rows=642000 loops=1)
            -> Nested loop inner join  (cost=7445606.18 rows=16703446) (actual time=0.656..3216.399 rows=642000 loops=1)
            -> Inner hash join (no condition)  (cost=1650174.80 rows=16500000) (actual time=0.636..154.644 rows=634000 loops=1)
                -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1.72 rows=16500) (actual time=0.048..18.424 rows=634 loops=1)
                    -> Table scan on p  (cost=1.72 rows=16500) (actual time=0.032..12.412 rows=16044 loops=1)
```
4. `Очень много циклов`
```
-> Covering index scan on f using idx_title  (cost=103.00 rows=1000) (actual time=0.061..0.448 rows=1000 loops=1)
    -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.25 rows=1) (actual time=0.003..0.005 rows=1 loops=634000)
        -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.00 rows=1) (actual time=0.000..0.000 rows=1 loops=642000)
            -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.00 rows=1) (actual time=0.000..0.000 rows=1 loops=642000)
```

`Возможные решения:`
- `Первый шаг, немного меняем запрос`
```
select concat(c.last_name, ' ', c.first_name) as `ФИО`, sum(p.amount) as amount #over (partition by c.customer_id, f.film_id)
from payment p, rental r, customer c, inventory i
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id 
group by `ФИО`
```
- `Как видно, без оконной функции и дедупликации время запроса на порядок сократилось`
```
-> Limit: 200 row(s)  (actual time=2338.327..2338.416 rows=200 loops=1)
    -> Table scan on <temporary>  (actual time=2338.326..2338.406 rows=200 loops=1)
        -> Aggregate using temporary table  (actual time=2338.324..2338.324 rows=391 loops=1)
            -> Nested loop inner join  (cost=10022515.88 rows=15415581) (actual time=0.452..1802.770 rows=642000 loops=1)
                -> Nested loop inner join  (cost=8477103.92 rows=15415581) (actual time=0.439..1611.551 rows=642000 loops=1)
                    -> Nested loop inner join  (cost=6931691.96 rows=15415581) (actual time=0.425..1414.450 rows=642000 loops=1)
                        -> Inner hash join (no condition)  (cost=1540127.25 rows=15400000) (actual time=0.398..50.634 rows=634000 loops=1)
                            -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1.56 rows=15400) (actual time=0.044..7.846 rows=634 loops=1)
                                -> Table scan on p  (cost=1.56 rows=15400) (actual time=0.029..5.570 rows=16044 loops=1)
                            -> Hash
                                -> Covering index scan on f using idx_fk_language_id  (cost=103.00 rows=1000) (actual time=0.030..0.246 rows=1000 loops=1)
                        -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.25 rows=1) (actual time=0.001..0.002 rows=1 loops=634000)
                    -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.00 rows=1) (actual time=0.000..0.000 rows=1 loops=642000)
                -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.00 rows=1) (actual time=0.000..0.000 rows=1 loops=642000)

                            
```
- `Добавим индекс и ещё немного сократим время выполнения`
```
CREATE index test_payment_in ON payment(payment_date);
```
```
-> Limit: 200 row(s)  (actual time=634.966..634.990 rows=200 loops=1)
    -> Table scan on <temporary>  (actual time=634.963..634.982 rows=200 loops=1)
        -> Aggregate using temporary table  (actual time=634.961..634.961 rows=391 loops=1)
            -> Inner hash join (no condition)  (cost=1599856.46 rows=15831000) (actual time=68.229..174.517 rows=642000 loops=1)
                -> Covering index scan on f using idx_fk_language_id  (cost=0.01 rows=1000) (actual time=0.018..0.773 rows=1000 loops=1)
                -> Hash
                    -> Nested loop inner join  (cost=16683.70 rows=15831) (actual time=0.381..67.969 rows=642 loops=1)
                        -> Nested loop inner join  (cost=11142.85 rows=15831) (actual time=0.106..37.454 rows=16044 loops=1)
                            -> Nested loop inner join  (cost=5602.00 rows=15831) (actual time=0.095..21.406 rows=16044 loops=1)
                                -> Table scan on c  (cost=61.15 rows=599) (actual time=0.040..0.310 rows=599 loops=1)
                                -> Index lookup on r using idx_fk_customer_id (customer_id=c.customer_id)  (cost=6.61 rows=26) (actual time=0.028..0.034 rows=27 loops=599)
                            -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.25 rows=1) (actual time=0.001..0.001 rows=1 loops=16044)
                        -> Index lookup on p using test_payment_in (payment_date=r.rental_date), with index condition: (cast(p.payment_date as date) = '2005-07-30')  (cost=0.25 rows=1) (actual time=0.002..0.002 rows=0 loops=16044)
```
- `Результат не тот, что ожидался, поэтому меняем условие, вместо date используем date_add, а вместо where and - inner join`
```
select concat(c.last_name, ' ', c.first_name) as ФИО, sum(p.amount) as amount
from payment p
inner join  rental r on p.payment_date = r.rental_date
inner join customer c on r.customer_id = c.customer_id
inner join inventory i on i.inventory_id = r.inventory_id
where p.payment_date >= '2005-07-30' and p.payment_date < DATE_ADD('2005-07-30', INTERVAL 1 DAY)
group by ФИО
```
- `Итог:`
```
-> Limit: 200 row(s)  (actual time=5.562..5.592 rows=200 loops=1)
    -> Table scan on <temporary>  (actual time=5.559..5.582 rows=200 loops=1)
        -> Aggregate using temporary table  (actual time=5.558..5.558 rows=391 loops=1)
            -> Nested loop inner join  (cost=793.03 rows=634) (actual time=0.067..4.655 rows=642 loops=1)
                -> Nested loop inner join  (cost=571.13 rows=634) (actual time=0.062..3.790 rows=642 loops=1)
                    -> Nested loop inner join  (cost=349.23 rows=634) (actual time=0.043..1.391 rows=634 loops=1)
                        -> Filter: ((r.rental_date >= TIMESTAMP'2005-07-30 00:00:00') and (r.rental_date < <cache>(('2005-07-30' + interval 1 day))))  (cost=127.33 rows=634) (actual time=0.029..0.469 rows=634 loops=1)
                            -> Covering index range scan on r using rental_date over ('2005-07-30 00:00:00' <= rental_date < '2005-07-31 00:00:00')  (cost=127.33 rows=634) (actual time=0.026..0.336 rows=634 loops=1)
                        -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.25 rows=1) (actual time=0.001..0.001 rows=1 loops=634)
                    -> Index lookup on p using test_payment_in (payment_date=r.rental_date)  (cost=0.25 rows=1) (actual time=0.003..0.004 rows=1 loops=634)
                -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.25 rows=1) (actual time=0.001..0.001 rows=1 loops=642)
```
