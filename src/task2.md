## Задание 2

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

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
    ```
                                                           QUERY PLAN                                                         
    ---------------------------------------------------------------------------------------------------------------------------
    Bitmap Heap Scan on t_books  (cost=21.03..1336.08 rows=750 width=33) (actual time=1.610..1.611 rows=1 loops=1)
   Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=1.560..1.560 rows=1 loops=1)
         Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
    Planning Time: 1.449 ms
    Execution Time: 1.792 ms
    (7 rows)
   ```
    
    *Объясните результат:*
    ```
       Bitmap Index Scan on t_books_fts_idx - Используется GIN индекс для полнотекстового поиска

    Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery) - Поиск слова 'expert' в индексе

    Bitmap Heap Scan - Получение данных из таблицы по найденным позициям

    Heap Blocks: exact=1 - Точное указание: нужно прочитать всего 1 блок данных

    rows=1 - Найдена 1 книга (как и ожидалось - 'Expert PostgreSQL Architecture')
   ```

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
    ```
                                                          QUERY PLAN                                                       
    -----------------------------------------------------------------------------------------------------------------------
    Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.050..0.050 rows=1 loops=1)
    Index Cond: ((item_key)::text = '0000000455'::text)
    Planning Time: 0.553 ms
    Execution Time: 0.142 ms
    (4 rows)
    ```
     
     *Объясните результат:*
     ```
        Index Scan using t_lookup_pk - Используется индекс по первичному ключу (B-tree)

    Index Cond: ((item_key)::text = '0000000455'::text) - Поиск конкретного значения в индексе

    rows=1 - Найдена ровно 1 запись

    Execution Time: 0.142 ms - Очень быстрое выполнение
    ```

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```
                                                                     QUERY PLAN                                                                  
    ---------------------------------------------------------------------------------------------------------------------------------------------
    Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.194..0.195 rows=1 loops=1)
    Index Cond: ((item_key)::text = '0000000455'::text)
    Planning Time: 1.277 ms
    Execution Time: 0.334 ms
    (4 rows)
    ```
     
     *Объясните результат:*
     ```
        Для поиска по первичному ключу B-tree индекс и так эффективен в обеих таблицах

    CLUSTER переписывает таблицу в порядке индекса, но для точечного поиска это не дает преимущества

    Дополнительные накладные расходы на поддержание кластеризованного порядка
    ```

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
                                                              QUERY PLAN                                                          
    ------------------------------------------------------------------------------------------------------------------------------
    Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.075..0.076 rows=0 loops=1)
    Index Cond: ((item_value)::text = 'T_BOOKS'::text)
    Planning Time: 0.642 ms
    Execution Time: 0.130 ms
    (4 rows)
    ```
     
     *Объясните результат:*
     ```
        Index Scan using t_lookup_value_idx - Используется индекс по значению (B-tree)

    Index Cond: ((item_value)::text = 'T_BOOKS'::text) - Поиск 'T_BOOKS' в индексе

    rows=0 - Не найдено, так как значения начинаются с 'Value_'

    Execution Time: 0.130 ms - Быстрый поиск даже для несуществующего значения
    ```

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
                                                                        QUERY PLAN                                                                    
    --------------------------------------------------------------------------------------------------------------------------------------------------
    Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.170..0.171 rows=0 loops=1)
    Index Cond: ((item_value)::text = 'T_BOOKS'::text)
    Planning Time: 1.409 ms
    Execution Time: 0.257 ms
    (4 rows)
    ```
     
     *Объясните результат:*
     ```
        CLUSTER физически переписывает таблицу в порядке первичного ключа

    Для поиска по другим полям (item_value) это не помогает

    Индекс по item_value в обеих таблицах работает одинаково

    Дополнительные расходы на доступ к кластеризованной таблице
    ```

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     ```
           Для поиска по индексированному полю (item_value) кластеризация НЕ дает преимущества

    Кластеризованная таблица всегда медленнее в нашем тесте:

        Поиск по PK: 0.142 ms vs 0.334 ms

        Поиск по значению: 0.130 ms vs 0.257 ms
    ```