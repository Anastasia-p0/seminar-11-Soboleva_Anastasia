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
   Созданный BRIN индекс ускоряет выполнение запроса, так как отсекает область поиска, выделяя блок с NULL значением category

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
   BRIN индекс по автору не использовался. Планировщик выбрал BRIN индекс по category для первичной фильтрации, так как посчитал его более эффективным, а дальше уже после получения строк из блоков, определённых по category, выполнил фильтрацию по author

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
   При выполнении запроса индексы не были использованы, так как BRIN индексы неэффективны для получения уникальных значений. Для выполнения DISTINCT планировщику всё равно нужно обработать каждую строку, чтобы проверить их уникальность.

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
   При выполнении данного запроса индексы не были использованы, так как хранят только минимальные и максимальные значения для блоков страниц, а не информацию о каждой строке. Так что если в блоке есть хотя бы одна строка, начинающаяся на "S", то весь блок будет включён в результат, даже если остальные строки не подходят. К тому же из-за того что данные не отсортированы, эффективность BRIN индексов сильно снижается.

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
   ```sql
   "Aggregate  (cost=3475.88..3475.89 rows=1 width=8) (actual time=127.899..127.901 rows=1 loops=1)"
   "  ->  Seq Scan on t_books  (cost=0.00..3474.00 rows=750 width=0) (actual time=127.888..127.891 rows=1 loops=1)"
   "        Filter: (lower((title)::text) ~~ 'o%'::text)"
   "        Rows Removed by Filter: 149999"
   "Planning Time: 0.507 ms"
   "Execution Time: 127.938 ms"
   ```
   
   *Объясните результат:*
   Индекс не использовался, так как сначала должен пройти по всем строкам и привести их к нижнему регисту, а это не ускоряет поиск

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
   ```sql
   "Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=2.803..2.804 rows=0 loops=1)"
   "  Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))"
   "  Rows Removed by Index Recheck: 8818"
   "  Heap Blocks: lossy=72"
   "  ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.041..0.042 rows=720 loops=1)"
   "        Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))"
   "Planning Time: 0.335 ms"
   "Execution Time: 2.869 ms"
   ```
   
   *Объясните результат:*
   Составной индекс применился и ускорил запрос, так как он определяет диапазон значений по category и по author для одних и тех же блоков и таким образом помогает ускорить запросы, комбинирующие фильтрацию по category и author. Однако данный индекс не супер эффективный, так как плохо подходит для точного поиска и для поиска на неупорядоченых данных
