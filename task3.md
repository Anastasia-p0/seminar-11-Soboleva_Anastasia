## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```sql
    "Bitmap Heap Scan on test_cluster  (cost=5579.04..20218.88 rows=500467 width=39) (actual time=25.775..135.163 rows=500615 loops=1)"
    "  Recheck Cond: (category = 'A'::text)"
    "  Heap Blocks: exact=8334"
    "  ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5453.93 rows=500467 width=0) (actual time=24.212..24.213 rows=500615       loops=1)"
    "        Index Cond: (category = 'A'::text)"
    "Planning Time: 0.360 ms"
    "Execution Time: 163.274 ms"
    ```
    
    *Объясните результат:*
    Несмотря на то что используется индекс, общее время выполнения относительно высоко из-за большого объема данных и количества строк, соответствующих условию

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    ```sql
    CLUSTER
    Query returned successfully in 1 secs 670 msec.
    ```

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```"Bitmap Heap Scan on test_cluster  (cost=5579.04..20168.88 rows=500467 width=39) (actual time=20.870..103.842 rows=500615             loops=1)"
    "  Recheck Cond: (category = 'A'::text)"
    "  Heap Blocks: exact=4172"
    "  ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5453.93 rows=500467 width=0) (actual time=20.152..20.153 rows=500615         loops=1)"
    "        Index Cond: (category = 'A'::text)"
    "Planning Time: 0.130 ms"
    "Execution Time: 130.826 ms"
    ```
    
    *Объясните результат:*
    Также был применён индекс для ускорения выполнения запроса, но всё ещё скорость выполнения достаточно невысокая, потому что очень много строк, соответствующих условию

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    После кластеризации производительность запроса хоть и не сильно, но улучшилась, так как кластеризации позволила сократить количество блоков, которые нужно было обрабатывать при выполнении запроса. 
