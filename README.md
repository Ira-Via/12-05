# 12-05
12-05 «Индексы» Васяева Ирина
# Задание 1
Для получения процентного отношения общего размера всех индексов к общему размеру всех таблиц в учебной базе данных, вам нужно будет использовать системные представления, которые предоставляют информацию о размерах таблиц и индексов. 
### Пример запроса:
```sql
DECLARE @total_table_size BIGINT;
DECLARE @total_index_size BIGINT;

SELECT @total_table_size = SUM(reserved_page_count) * 8
FROM sys.dm_db_partition_stats;

SELECT @total_index_size = SUM(a.total_pages) * 8
FROM sys.indexes AS i
JOIN sys.partitions AS p ON i.object_id = p.object_id AND i.index_id = p.index_id
JOIN sys.allocation_units AS a ON p.partition_id = a.container_id
WHERE i.type > 0; -- Индексы (не кластерные)

SELECT 
    (@total_index_size * 100.0 / @total_table_size) AS процентное_отношение
WHERE @total_table_size > 0; -- Избегаем деления на ноль
```
### Объяснение запроса:
1. **DECLARE**: Объявляем переменные для хранения общего размера таблиц и индексов.
2. **SELECT @total_table_size**: Получаем общий размер всех таблиц, используя системное представление `sys.dm_db_partition_stats`, которое содержит информацию о страницах, использованных каждой таблицей.
3. **SELECT @total_index_size**: Получаем общий размер всех индексов, используя представления `sys.indexes`, `sys.partitions` и `sys.allocation_units`.
4. **WHERE i.type > 0**: Фильтруем только индексы (кроме кластерных).
5. **SELECT (@total_index_size * 100.0 / @total_table_size)**: Вычисляем процентное отношение размера индексов к размеру таблиц.
6. **WHERE @total_table_size > 0**: Проверяем, чтобы избежать деления на ноль.
# Задание 2
### 1: Выполнение `EXPLAIN ANALYZE`
```sql
EXPLAIN ANALYZE
SELECT DISTINCT CONCAT(c.last_name, ' ', c.first_name), SUM(p.amount) OVER (PARTITION BY c.customer_id, f.title)
FROM payment p
JOIN rental r ON p.payment_date = r.rental_date
JOIN customer c ON r.customer_id = c.customer_id
JOIN inventory i ON i.inventory_id = r.inventory_id
JOIN film f ON f.film_id = i.film_id
WHERE DATE(p.payment_date) = '2005-07-30';
```
### 2: Анализ результатов `EXPLAIN ANALYZE`
1. **Полное сканирование таблиц**: Если запрос выполняет полное сканирование таблиц (например, `payment`, `rental`), это может указывать на отсутствие индексов.
2. **Высокая стоимость операций**: Обратите внимание на стоимость операций (например, `Seq Scan`, `Hash Join`, `Nested Loop`), которые могут указывать на неэффективные способы соединения таблиц.
3. **Количество возвращаемых строк**: Если количество возвращаемых строк значительно превышает ожидаемое, это может указывать на необходимость более строгих условий фильтрации или индексов.
### 3: Оптимизация запроса
1. **Использование явных соединений**: Вместо использования старого синтаксиса `FROM table1, table2`, лучше использовать явные `JOIN` операторы. Это улучшает читаемость и может помочь оптимизатору.
2. **Добавление индексов**: Убедитесь, что у вас есть индексы на столбцах, которые используются в условиях соединения и фильтрации. Например:
   - Индекс на `payment(payment_date)`
   - Индекс на `rental(rental_date, customer_id, inventory_id)`
   - Индекс на `customer(customer_id)`
   - Индекс на `inventory(inventory_id, film_id)`
   - Индекс на `film(film_id)`
3. **Фильтрация данных**: Убедитесь, что фильтрация по дате выполняется на самом начале, чтобы уменьшить объем данных, обрабатываемых в последующих операциях.
4. **Избегание `DISTINCT`**: Если возможно, избегайте использования `DISTINCT`, так как это может быть дорогостоящей операцией. Если вы можете гарантировать уникальность строк, это улучшит производительность.
### Оптимизированный запрос
```sql
SELECT CONCAT(c.last_name, ' ', c.first_name) AS full_name, 
       SUM(p.amount) OVER (PARTITION BY c.customer_id, f.title) AS total_amount
FROM payment p
JOIN rental r ON p.payment_date = r.rental_date
JOIN customer c ON r.customer_id = c.customer_id
JOIN inventory i ON i.inventory_id = r.inventory_id
JOIN film f ON f.film_id = i.film_id
WHERE p.payment_date = '2005-07-30' -- Убираем DATE() для повышения производительности
GROUP BY c.customer_id, f.title, c.last_name, c.first_name;
```
# Задание 3*
В PostgreSQL существует несколько типов индексов, которые обеспечивают гибкость и мощные возможности для оптимизации запросов. Некоторые из этих индексов уникальны для PostgreSQL и не имеют прямых аналогов в MySQL. Вот основные типы индексов, используемые в PostgreSQL, которые отсутствуют в MySQL:

1. **B-tree индексы**: Это наиболее распространенный тип индекса, который используется по умолчанию. Он поддерживает операции равенства и диапазона, и его можно использовать для сортировки. Хотя B-tree индексы есть и в MySQL, их реализация и возможности могут различаться.

2. **Hash индексы**: Эти индексы оптимизированы для операций равенства. В PostgreSQL они могут быть полезны для оптимизации запросов, где используются операции `=`. MySQL также поддерживает хеш-индексы, но они имеют ограничения и не так часто используются.

3. **GiST (Generalized Search Tree) индексы**: Этот тип индекса позволяет создавать индексы для более сложных типов данных, таких как геометрические данные и полнотекстовые поисковые запросы. GiST индексы обеспечивают возможность создания пользовательских типов индексов, что делает их очень мощными. MySQL не имеет аналогичных индексов.

4. **GIN (Generalized Inverted Index) индексы**: GIN индексы особенно полезны для работы с массивами и полнотекстовыми поисковыми запросами. Они позволяют эффективно индексировать данные, которые могут содержать несколько значений, такие как массивы или JSONB. MySQL не имеет GIN индексов.

5. **SP-GiST (Space-partitioned Generalized Search Tree) индексы**: Этот тип индекса подходит для работы с данными, которые имеют пространственные или иерархические структуры. SP-GiST индексы могут использоваться для оптимизации запросов на основе пространственных данных, чего MySQL не поддерживает так же эффективно.

6. **BRIN (Block Range INdexes) индексы**: BRIN индексы предназначены для работы с очень большими таблицами, где данные имеют физическую последовательность. Они хранят минимальные и максимальные значения для блоков данных, что позволяет быстро находить нужные строки. MySQL не имеет аналогичных индексов.

7. **Пользовательские индексы**: PostgreSQL позволяет создавать пользовательские индексы, используя расширения и специфические функции, что дает разработчикам возможность оптимизировать запросы под свои нужды. MySQL имеет более ограниченные возможности для создания пользовательских индексов.
