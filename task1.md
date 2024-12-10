# Задание 1: BRIN индексы и bitmap-сканирование

1. Удалите старую базу данных, если есть:
   ```shell
   docker compose down
   ```

2. Поднимите базу данных из src/docker-compose.yml:
   ```shell
   docker compose down && docker compose up -d
   ```

3. Обновите статистику:
   ```sql
   ANALYZE t_books;
   ```

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   ```sql
   "Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.028..0.028 rows=0 loops=1)"
   "  Recheck Cond: (category IS NULL)"
   "  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.022..0.022 rows=0 loops=1)"
   "        Index Cond: (category IS NULL)"
   "Planning Time: 0.126 ms"
   "Execution Time: 0.053 ms"
   ```
   
   *Объясните результат:*
   [Ваше объяснение]

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   ```sql
   "Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=31.741..31.742 rows=0 loops=1)"
   "  Recheck Cond: ((category)::text = 'INDEX'::text)"
   "  Rows Removed by Index Recheck: 150000"
   "  Filter: ((author)::text = 'SYSTEM'::text)"
   "  Heap Blocks: lossy=1224"
   "  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.074..0.075 rows=12240 loops=1)"
   "        Index Cond: ((category)::text = 'INDEX'::text)"
   "Planning Time: 0.336 ms"
   "Execution Time: 31.790 ms"
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   [Ваше объяснение]

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```sql
   "Sort  (cost=3099.11..3099.12 rows=5 width=7) (actual time=74.715..74.718 rows=6 loops=1)"
   "  Sort Key: category"
   "  Sort Method: quicksort  Memory: 25kB"
   "  ->  HashAggregate  (cost=3099.00..3099.05 rows=5 width=7) (actual time=74.694..74.697 rows=6 loops=1)"
   "        Group Key: category"
   "        Batches: 1  Memory Usage: 24kB"
   "        ->  Seq Scan on t_books  (cost=0.00..2724.00 rows=150000 width=7) (actual time=0.019..18.568 rows=150000 loops=1)"
   "Planning Time: 0.085 ms"
   "Execution Time: 74.787 ms"
   ```
   
   *Объясните результат:*
   [Ваше объяснение]

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```sql
   "Aggregate  (cost=3099.04..3099.05 rows=1 width=8) (actual time=55.373..55.375 rows=1 loops=1)"
   "  ->  Seq Scan on t_books  (cost=0.00..3099.00 rows=15 width=0) (actual time=55.338..55.339 rows=0 loops=1)"
   "        Filter: ((author)::text ~~ 'S%'::text)"
   "        Rows Removed by Filter: 150000"
   "Planning Time: 0.332 ms"
   "Execution Time: 55.420 ms"
   ```
   
   *Объясните результат:*
   [Ваше объяснение]

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   [Вставьте план выполнения]
   
   *Объясните результат:*
   [Ваше объяснение]

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   [Вставьте план выполнения]
   
   *Объясните результат:*
   [Ваше объяснение]
