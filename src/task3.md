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
    ```
                                                                  QUERY PLAN                                                               
    ----------------------------------------------------------------------------------------------------------------------------------------
    Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=13.857..85.077 rows=500104 loops=1)
   Recheck Cond: (category = 'A'::text)
   Heap Blocks: exact=8334
   ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=13.021..13.021 rows=500104 loops=1)
         Index Cond: (category = 'A'::text)
    Planning Time: 0.586 ms
    Execution Time: 97.996 ms
    (7 rows)
   ```
    
    *Объясните результат:*
    ```
       Оптимизатор думал, что будет только 5000 строк (0.5%)

    Фактически строк в 100 раз больше (50%)

    Для 50% строк Seq Scan мог бы быть эффективнее 
   ```

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    ```
   workshop=#     CLUSTER test_cluster USING test_cluster_cat_idx;

    CLUSTER 
   ```

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```
                                                                     QUERY PLAN                                                                 
    --------------------------------------------------------------------------------------------------------------------------------------------
    Bitmap Heap Scan on test_cluster  (cost=5537.07..20078.58 rows=496600 width=39) (actual time=14.928..86.291 rows=500104 loops=1)
   Recheck Cond: (category = 'A'::text)
   Heap Blocks: exact=4168
   ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5412.93 rows=496600 width=0) (actual time=14.273..14.274 rows=500104 loops=1)
         Index Cond: (category = 'A'::text)
    Planning Time: 1.023 ms
    Execution Time: 102.611 ms
    (7 rows)
    ```
    
    *Объясните результат:*
    ```
       Heap Blocks: exact=4168 - В 2 раза меньше блоков нужно читать! (было 8334, стало 4168)

    Статистика улучшилась: ожидаемые строки rows=496600 (ближе к реальным 500104)

    Время выполнения: 102.6 ms vs 98.0 ms (незначительное увеличение)
   
   ```

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    ```
   Улучшения после кластеризации:

    В 2 раза меньше блоков данных (4168 вместо 8334) - главное преимущество!

    Более точная статистика (496600 vs 5000 ожидаемых строк)

    Данные физически упорядочены - все 'A' находятся в первых ~50% блоков

    Недостатки/нейтрально:

    Время выполнения не уменьшилось (102.6 ms vs 98.0 ms)

    Время планирования увеличилось (1.023 ms vs 0.586 ms)

    Почему время не уменьшилось, хотя блоков в 2 раза меньше?

    Bitmap Index Scan все еще нужен - для определения, какие строки выбирать

    Чтение с диска может быть буферизировано - данные могли быть в кэше

    Накладные расходы на поддержание порядка после кластеризации

    Реальное преимущество кластеризации проявится при:

    Диапазонных запросах по кластеризованному полю

    Последовательном чтении большого объема данных

    Уменьшении IO на холодных данных (не в кэше)
   ```