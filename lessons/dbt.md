## 1. Мотивация и место dbt в архитектуре данных

-   **Проблематика:** Традиционный подход ETL (извлечение-трансформация-загрузка) подразумевает написание сложного процедурного кода (пример с `CREATE PROCEDURE`).
-   **ETL-инструменты:** Пример кода на Python для Apache Airflow показывает, как можно оркестрировать задачи ClickHouse, но это все еще описание процессов, а не самих трансформаций.
    ```python
    # Пример задачи в Airflow для ClickHouse
    task1 = ClickhouseOperator(task_id='get_data', sql='select * from table', dag=dag)
    task2 = ClickhouseOperator(task_id='insert_data', sql='insert into table values()', retries=3, dag=dag)
    task1 >> task2
    ```
-   **Роль dbt:** Data Build Tool реализует букву **T (Transform)** в парадигме **ELT**. Он фокусируется именно на трансформации данных внутри хранилища.

## 2. Основные концепции dbt

-   **Модели:** Основной строительный блок в dbt. **Модель — это ничто иное как SELECT-запрос**. dbt берет этот запрос и материализует его в базе данных в виде таблицы или представления.
-   **SQL + Jinja:** dbt позволяет использовать шаблонизатор Jinja вместе с SQL, делая код динамичным и модульным.
-   **Графы исполнения (DAGs):** dbt автоматически строит граф зависимостей между моделями, что позволяет выполнять только нужные части пайплайна и в правильном порядке.
-   **Синтаксис выбора узлов:** Предоставляет гибкие возможности для запуска только определенных моделей или их групп.
    -   Примеры команд:
        -   `dbt run --exclude cars_positions_zones tag:dq` (запустить все, кроме моделей с тегом `dq`)
        -   `dbt run -m tag:dq --full-refresh` (запустить только модели с тегом `dq` с полным обновлением)

## 3. Структура и конфигурация проекта dbt

-   **Файл проекта (`dbt_project.yml`):** Содержит метаданные о проекте.
    -   `name: acme_corp`
    -   `profile: acme_corp` (ссылка на профиль подключения к БД)
    -   `version: '1.0'`
    -   `require-dbt-version: ">=0.14.0"`
    -   **Пути к каталогам:** Определяет, где dbt будет искать модели, аналитические запросы, тесты, макросы и т.д.
        -   `source-paths: ["models"]`
        -   `analysis-paths: ["analysis"]`
        -   `test-paths: ["tests"]`
        -   `data-paths: ["data"]`
        -   `macro-paths: ["macros"]`

## 4. Материализация моделей

-   **Полная и инкрементальная загрузка:** Конфигурация модели задается в блоке `config`.
    -   **`materialized='incremental'`** — указывает, что модель должна обновляться инкрементально, а не полностью перестраиваться.
    -   **`unique_key='id'`** — ключ для идентификации уникальных записей при инкрементальном обновлении.
    -   В примере также показаны параметры `dist` и `sort`, характерные для СУБД (например, Greenplum), но не для ClickHouse.
    ```sql
    {{
        config (
            materialized='incremental',
            sql_where='true',
            unique_key='id',
            dist="call_id",
            sort="min_event_ts_msk",
        )
    }}
    ```

## 5. Использование CTE (Common Table Expressions)

-   **Определение:** CTE (обобщённое табличное выражение) — это временный результирующий набор, на который можно ссылаться внутри SQL-запроса.
    ```sql
    WITH cte_name (column1, column2, ..., columnN) AS (
        -- Query definition goes here
    )
    SELECT column1, column2, ..., columnN
    FROM cte_name
    -- Additional query operations go here
    ```
-   **Назначение:** Упрощают сложные запросы, делают их более читаемыми и модульными, разбивая на логические блоки.

## 6. Уровни моделей (Пример)

-   **Staging (Подготовка):** Прямое отображение данных из источников (например, `stg_books`, `stg_authors`). Выполняются легкие трансформации (переименование, приведение типов).
-   **Intermediate (Промежуточные):** Модели, которые объединяют данные из разных staging-моделей для создания единого набора данных. Служат строительными блоками для итоговых витрин.
    ```sql
    -- int_book_authors.sql
    WITH
        books AS (SELECT * FROM {{ ref('stg_books') }}),
        authors AS (SELECT * FROM {{ ref('stg_authors') }})
    SELECT
        b.book_id,
        b.title,
        a.author_id,
        a.author_name
    FROM books b
    JOIN authors a ON b.author_id = a.author_id
    ```
-   **Mart (Витрины):** Итоговые модели, агрегирующие данные для конкретных задач или отчетов. Часто материализуются как таблицы.
    ```sql
    -- mart_book_authors.sql
    {{
        config( materialized='table', unique_key='author_id', sort='author_id' )
    }}
    WITH book_counts AS (
        SELECT
            author_id,
            COUNT(*) AS total_books
        FROM {{ ref('int_book_authors') }}
        GROUP BY author_id
    )
    SELECT
        author_id,
        total_books
    FROM book_counts
    ```

## 7. Специфичная конфигурация для ClickHouse (dbt-clickhouse)

-   **Материализации:**
    -   `view` (Да)
    -   `table` (Да)
    -   `incremental` (Да)
    -   `ephemeral` (Да, создает CTE)
-   **Экспериментальные материализации:**
    -   Materialized View (Да, Экспериментально)
    -   Distributed table (Да, Экспериментально)
    -   Distributed incremental (Да, Экспериментально)
    -   Dictionary (Да, Экспериментально)

-   **Конфигурация таблиц:**
    -   `engine` (Опционально) - Движок таблицы. По умолчанию: `MergeTree()`.
    -   `order_by` (Опционально) - Ключ сортировки (для разреженного индекса). По умолчанию: `tuple()`.
    -   `partition_by` (Опционально) - Ключ партиционирования.

-   **Конфигурация инкрементальных таблиц:**
    -   `unique_key` (Обязательно) - Кортеж колонок для уникальной идентификации строк.
    -   `engine`, `order_by`, `partition_by` - аналогично таблицам.
    -   `incremental_strategy` (Опционально) - Стратегия инкрементальной загрузки. Поддерживаются: `delete+insert`, `append` и `insert_overwrite` (экспериментально). По умолчанию: `'default'`.
    -   `incremental_predicates` (Опционально) - Предикаты для стратегии `delete+insert`.

-   **Поддерживаемые движки таблиц:**
    -   **Основные:** MergeTree (по умолчанию), HDFS, MaterializedPostgreSQL, S3, EmbeddedRocksDB, Hive.
    -   **Экспериментальные:** Distributed Table, Dictionary.

## 8. Материалы для изучения

В презентации приведен список рекомендуемых материалов:
1.  dbt Getting Started Tutorial
2.  dbt Documentation
3.  dbt FAQ
4.  How we structure our dbt projects
5.  The Modern Data Stack: Past, Present, and Future
6.  Five principles that will keep your data warehouse organized
7.  The Analytics Engineering Guide
```
