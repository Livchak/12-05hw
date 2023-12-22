# Домашнее задание к занятию "`12-05hw`" - `Ливчак Сергей`


---

### Задание 1

`Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.`

1. `Из инормационной схемы берём сумму байт индексов и данных и через пропорцию находим процентное отношение размера индекса к данным`

```
SELECT ROUND((100 * SUM(index_length))/ SUM(data_length), 1) AS 'index-to-database in pct'
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'sakila';

```

1. **12-05hw-1** 
<img src = "img/12-05hw-1-1.png" width = 80%>


---

### Задание 2

`Выполните explain analyze следующего запроса:

select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id

1) перечислите узкие места;
2) оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.`

<details>
  <summary>explain analyze запроса из задания 2</summary>

```
CREATE INDEX
-> Limit: 200 row(s)  (cost=0..0 rows=0) (actual time=30450..30450 rows=200 loops=1)
    -> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=30450..30450 rows=200 loops=1)
        -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=30450..30450 rows=391 loops=1)
            -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY customer.customer_id,film.title )   (actual time=12157..29320 rows=642000 loops=1)
                -> Sort: customer.customer_id, film.title  (actual time=12157..12614 rows=642000 loops=1)
                    -> Stream results  (cost=22e+6 rows=16.1e+6) (actual time=1.62..9130 rows=642000 loops=1)
                        -> Nested loop inner join  (cost=22e+6 rows=16.1e+6) (actual time=1.6..7620 rows=642000 loops=1)
                            -> Nested loop inner join  (cost=20.4e+6 rows=16.1e+6) (actual time=1.59..6740 rows=642000 loops=1)
                                -> Nested loop inner join  (cost=18.8e+6 rows=16.1e+6) (actual time=1.57..5857 rows=642000 loops=1)
                                    -> Inner hash join (no condition)  (cost=1.61e+6 rows=16.1e+6) (actual time=1.54..310 rows=634000 loops=1)
                                        -> Filter: (cast(payment.payment_date as date) = '2005-07-30')  (cost=1.68 rows=16086) (actual time=0.114..61.3 rows=634 loops=1)
                                            -> Table scan on payment  (cost=1.68 rows=16086) (actual time=0.0803..51 rows=16044 loops=1)
                                        -> Hash
                                            -> Covering index scan on film using idx_title  (cost=112 rows=1000) (actual time=0.0958..1.03 rows=1000 loops=1)
                                    -> Covering index lookup on rental using rental_date (rental_date=payment.payment_date)  (cost=0.969 rows=1) (actual time=0.00564..0.00814 rows=1.01 loops=634000)
                                -> Single-row index lookup on customer using PRIMARY (customer_id=rental.customer_id)  (cost=250e-6 rows=1) (actual time=671e-6..761e-6 rows=1 loops=642000)
                            -> Single-row covering index lookup on inventory using PRIMARY (inventory_id=rental.inventory_id)  (cost=250e-6 1)1)rows=1) (actual time=556e-6..651e-6 rows=1 loops=642000)

```
</details>

1. `Читаем explain analyze построчно снизу вверх начиная с вложенных операций`
2. `Начинается всё с join по индексам таблиц inventory, customer и поиск в таблице rental по индексу rental_date, который соответствует payment_date из таблицы payment. Это всё быстрые операции.`
3. `Так же из-за того что мы явно не указали в join какие столцы с какими, он видимо использовал join по хеш функции. Достаточно быстрая операция`
4. `Далее mysql в Nested loop inner join выполняет поиск соответствующих строк между таблицами. Этот способ затратен если база данных достаточо большая и мы видим ччто он потратил почти 7 секунд, а затем из-за большого количества данных он почти две секунды передаёт данные. с 7620 по 9130 милисекунду передаёт данные. Если мы явно зададим соответствие строк по таблицам, он не будет сравнивать кажду строку таблицы с соответсвующей строкой друго таблицы и это может стать точкой оптимизации.`
5. `Далее у нас идёт сортировка результатов перед применением оконной агрегации. Быстрая операция 12157..12614 мс`
6. `И самая тяжелая операция - оконная агрегация с буфером. Суммирование платежей по каждому customer.customer_id и film.title 12157..29320 Тут он выполняет много лишней работы. Это тоже может стать точкой оптимизации, потому как нам достаточно лишь посчитать на запрошенную нами дату, а не всё подряд`

7. `В нашем запросе в функции оконной агрегации идёт условие   "where date(payment.payment_date) = '2005-07-30'" и оно связывается с "sum(payment.amount)", значит для ускорения мы можем в таблице payment создать индекс "payment_date_payment_amount"  `

```
CREATE INDEX idx_payment_date_payment_amount ON payment(payment_date, amount);
```
<details>
  <summary>explain analyze после создания индекса в таблице payment</summary>
  
  ```
  -> Limit: 200 row(s)  (cost=0..0 rows=0) (actual time=21849..21850 rows=200 loops=1)
    -> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=21849..21850 rows=200 loops=1)
        -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=21849..21849 rows=391 loops=1)
            -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY customer.customer_id,film.title )   (actual time=4299..20841 rows=642000 loops=1)
                -> Sort: customer.customer_id, film.title  (actual time=4299..4716 rows=642000 loops=1)
                    -> Stream results  (cost=1.6e+6 rows=15.7e+6) (actual time=573..1917 rows=642000 loops=1)
                        -> Inner hash join (no condition)  (cost=1.6e+6 rows=15.7e+6) (actual time=573..988 rows=642000 loops=1)
                            -> Covering index scan on film using idx_title  (cost=0.011 rows=1000) (actual time=0.0532..5.29 rows=1000 loops=1)
                            -> Hash
                                -> Nested loop inner join  (cost=27121 rows=15684) (actual time=1.74..571 rows=642 loops=1)
                                    -> Nested loop inner join  (cost=10855 rows=15420) (actual time=0.454..256 rows=16044 loops=1)
                                        -> Nested loop inner join  (cost=5458 rows=15420) (actual time=0.435..127 rows=16044 loops=1)
                                            -> Table scan on customer  (cost=61.2 rows=599) (actual time=0.139..1.64 rows=599 loops=1)
                                            -> Index lookup on rental using idx_fk_customer_id (customer_id=customer.customer_id)  (cost=6.44 rows=25.7) (actual time=0.146..0.204 rows=26.8 loops=599)
                                        -> Single-row covering index lookup on inventory using PRIMARY (inventory_id=rental.inventory_id)  (cost=0.25 rows=1) (actual time=0.00691..0.00698 rows=1 loops=16044)
                                    -> Filter: (cast(payment.payment_date as date) = '2005-07-30')  (cost=0.953 rows=1.02) (actual time=0.0192..0.0193 rows=0.04 loops=16044)
                                        -> Covering index lookup on payment using idx_payment_date_payment_amount (payment_date=rental.rental_date)  (cost=0.953 rows=1.02) (actual time=0.0102..0.0172 rows=3.06 loops=16044)

  ```

</details>

`При повторном выполнении explain analyze мы так же видим что получили не только ускорение оконной агрегации на 4 секунды, но и Nested loop inner join, по скольку там идёт поиск соответсвий между таблицами `


8.`Можно изменить запрос и сделать запросы join явными, задать условие по дате 2005-07-30 и отсортировать вывод. Тогда на выполнение запроса нужны будут доли секунды`

```
SELECT CONCAT(customer.last_name, ' ', customer.first_name), SUM(payment.amount)
FROM payment
INNER JOIN rental ON payment.rental_id = rental.rental_id
INNER JOIN customer ON rental.customer_id = customer.customer_id
WHERE DATE(payment.payment_date) = '2005-07-30'
GROUP BY customer.customer_id, customer.last_name, customer.first_name;
```

<details>
  <summary>explain analyze после изменения запроса с явным заданием join</summary>

-> Limit: 200 row(s)  (actual time=289..289 rows=200 loops=1)
    -> Table scan on <temporary>  (actual time=289..289 rows=200 loops=1)
        -> Aggregate using temporary table  (actual time=289..289 rows=391 loops=1)
            -> Nested loop inner join  (cost=7167 rows=15460) (actual time=0.7..281 rows=634 loops=1)
                -> Nested loop inner join  (cost=1756 rows=15420) (actual time=0.134..24.2 rows=16044 loops=1)
                    -> Table scan on customer  (cost=61.2 rows=599) (actual time=0.0898..1.28 rows=599 loops=1)
                    -> Covering index lookup on rental using idx_fk_customer_id (customer_id=customer.customer_id)  (cost=0.259 rows=25.7) (actual time=0.0177..0.0337 rows=26.8 loops=599)
                -> Filter: (cast(payment.payment_date as date) = '2005-07-30')  (cost=0.251 rows=1) (actual time=0.0153..0.0154 rows=0.0395 loops=16044)
                    -> Index lookup on payment using fk_payment_rental (rental_id=rental.rental_id)  (cost=0.251 rows=1) (actual time=0.012..0.0146 rows=1 loops=16044)


</details>

---
