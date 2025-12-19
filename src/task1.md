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
   ```
                                                              QUERY PLAN                                                          
    ------------------------------------------------------------------------------------------------------------------------------
    Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.027..0.028 rows=0 loops=1)
   Recheck Cond: (category IS NULL)
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.024..0.024 rows=0 loops=1)
         Index Cond: (category IS NULL)
    Planning Time: 0.611 ms
    Execution Time: 0.132 ms
    (6 rows)
    ```
   
   *Объясните результат:*
   ```
       Данные были сгенерированы так, что категория всегда заполнена (нет NULL)

    BRIN индекс быстро определил, что нет блоков с NULL значениями

    Запрос выполнился почти мгновенно

    ```

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
   ```
                                                               QUERY PLAN                                                            
    ----------------------------------------------------------------------------------------------------------------------------------
    Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=13.499..13.499 rows=0 loops=1)
   Recheck Cond: ((category)::text = 'INDEX'::text)
   Rows Removed by Index Recheck: 150000
   Filter: ((author)::text = 'SYSTEM'::text)
   Heap Blocks: lossy=1225
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.102..0.102 rows=12250 loops=1)
         Index Cond: ((category)::text = 'INDEX'::text)
    Planning Time: 0.416 ms
    Execution Time: 13.558 ms
    (9 rows)
    ```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   ```
        Категории в вашей таблице: 'Fiction', 'Science', 'History', 'Technology', 'Art', 'Databases'

    Категории 'INDEX' не существует в таблице!

    Когда BRIN ищет несуществующее значение, он может вернуть много ложных срабатываний
    ```

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```
                                                            QUERY PLAN                                                        
    --------------------------------------------------------------------------------------------------------------------------
    Sort  (cost=3100.11..3100.12 rows=5 width=7) (actual time=29.251..29.252 rows=6 loops=1)
   Sort Key: category
   Sort Method: quicksort  Memory: 25kB
   ->  HashAggregate  (cost=3100.00..3100.05 rows=5 width=7) (actual time=29.220..29.221 rows=6 loops=1)
         Group Key: category
         Batches: 1  Memory Usage: 24kB
         ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.042..9.986 rows=150000 loops=1)
    Planning Time: 0.572 ms
    Execution Time: 29.447 ms
    (9 rows)
    ```
   
   *Объясните результат:*
   ```
       Для получения всех уникальных значений нужно прочитать все строки

    BRIN не хранит все значения, только диапазоны (min/max) для блоков

    Оптимизатор правильно выбрал полное сканирование
   ```

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```
                                                     QUERY PLAN                                                  
    -------------------------------------------------------------------------------------------------------------
    Aggregate  (cost=3100.04..3100.05 rows=1 width=8) (actual time=20.002..20.003 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=0) (actual time=19.995..19.995 rows=0 loops=1)
         Filter: ((author)::text ~~ 'S%'::text)
         Rows Removed by Filter: 150000
    Planning Time: 1.238 ms
    Execution Time: 20.266 ms
    (6 rows)
    ```
   
   *Объясните результат:*
   ```
        BRIN индекс по автору НЕ используется для префиксного поиска (LIKE 'S%')

    Причина: BRIN хранит только min/max значения для блоков

        Например, блок содержит авторов от 'Author_100' до 'Author_200'

        'S%' не входит в этот диапазон, но PostgreSQL не может быть уверен

        Поэтому проверяет весь блок
    ```

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
   ```
                                                      QUERY PLAN                                                  
    --------------------------------------------------------------------------------------------------------------
    Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=41.712..41.713 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=41.706..41.707 rows=1 loops=1)
         Filter: (lower((title)::text) ~~ 'o%'::text)
         Rows Removed by Filter: 149999
    Planning Time: 0.732 ms
    Execution Time: 41.777 ms
    (6 rows)
    ```
   
   *Объясните результат:*
   ```
       Seq Scan on t_books - Полное сканирование таблицы вместо использования индекса!

    Filter: (lower((title)::text) ~~ 'o%'::text) - Фильтр применяется к каждой строке

    Rows Removed by Filter: 149999 - Только 1 строка прошла фильтр

    rows=1 - Найдена всего 1 книга, начинающаяся на 'o'
   ```

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
   ```
                                                                QUERY PLAN                                                              
    -------------------------------------------------------------------------------------------------------------------------------------
    Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=2.210..2.211 rows=0 loops=1)
    Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Rows Removed by Index Recheck: 8861
   Heap Blocks: lossy=73
   ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.058..0.059 rows=730 loops=1)
         Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
    Planning Time: 0.719 ms
    Execution Time: 2.382 ms
    (8 rows)
   ```
   
   *Объясните результат:*
   ```
    Bitmap Index Scan on t_books_brin_cat_auth_idx - Используется составной BRIN индекс

    Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text)) - Поиск по двум полям

    Heap Blocks: lossy=73 - BRIN вернул 73 блока для проверки

    Rows Removed by Index Recheck: 8861 - 8861 строк проверено, ни одна не подошла

    rows=0 - Не найдено книг с category='INDEX' AND author='SYSTEM'
    ```