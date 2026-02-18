## 1. Зачем профилировать запросы?

Профилирование — это диагностика хода работы запроса с целью его улучшения.

Существует два основных подхода к работе с запросами:

1.  **Классический:**
    1.  Необходимо разработать запрос
    2.  Пишем
    3.  Работает медленно
    4.  **Профилируем**
    5.  Оптимизируем
    6.  Выводим в production

2.  **Production-based подход:**
    1.  Создаём (пишем, отлаживаем) запрос
    2.  Выводим в production
    3.  Эксплуатируем
    4.  Запрос деградирует (данных становится больше)
    5.  **Профилируем**
    6.  Оптимизируем
    7.  Выводим в production

## 2. Текстовый лог запроса

Самый детализированный способ диагностики, доступный интерактивно в `clickhouse-client`.

**Как включить:**
- Для всей сессии: `SET send_logs_level = 'trace';`
- Для конкретного запроса: `SELECT ... SETTINGS send_logs_level = 'trace';`
- Чтобы получить только отладку без результата: `SELECT ... SETTINGS send_logs_level = 'trace' FORMAT Null;`

### 2.1. Структура лога

Каждая запись в логе содержит стандартные атрибуты:
- **Timestamp:** Когда произошло событие.
  ```
  [86ef3191d0b8] 2025.12.15 00:12:37.404724 ...
  ```
- **ID потока (Thread ID):** В каком потоке произошло событие.
  ```
  ... [ 85 ] {92975248-...} <Debug> ...
  ```
- **Идентификатор запроса (Query ID):** В каком запросе произошло событие.
  ```
  ... {92975248-37e0-4ec7-ab47-199813358880} <Debug> ...
  ```
- **Уровень логирования:** `<Debug>`, `<Trace>` и т.д.

### 2.2. Разбор лога: план сканирования (`SelectExecutor`)

Ключевой блок для понимания того, как будут читаться данные. Пример лога:
```
(SelectExecutor): Key condition: (column 1 in [18779, 18779])
(SelectExecutor): MinMax index condition: (column 0 in [18779, 18779])
...
(SelectExecutor): Selected 8/203 parts by partition key, 8 parts by primary key, 168/168 marks by primary key, 168 marks to read from 8 ranges
```
Здесь видно:
- Использовался первичный ключ (`Key condition`).
- Использовалось партиционирование (`MinMax index condition`, `partition key`).
- Какая часть данных была отсеяна на каждом этапе.

### 2.3. Пример оптимизации с индексом

1.  Находим популярное значение `uploader_id`:
    ```sql
    SELECT topK(uploader_id)[1] FROM youtube WHERE upload_date = '2021-06-01';
    -- Результат: UCOSz58wVDMYJ0vpb2T8S5FA
    ```

2.  Выполняем запрос поиска без эффективного использования ключа (просто по дате).
    *(В конспекте не показан лог, но подразумевается, что сканируется много данных)*

3.  Выполняем запрос, используя первичный ключ `(uploader_id, upload_date)`:
    ```sql
    SELECT count() FROM default.youtube
    WHERE (upload_date = '2021-06-01') AND (uploader_id = 'UCOSz58wVDMYJ0vpb2T8S5FA')
    ```
    Лог покажет значительное улучшение:
    ```
    Selected 8/203 parts by partition key, 1 parts by primary key, 1/168 marks by primary key, 1 marks to read from 1 ranges
    ```
    Количество читаемых `parts` и `marks` резко сократилось.

### 2.4. На что обратить внимание в логе

1.  **Время разбора запроса (начало лога):** Обычно незначительное. Если в запросе много данных в `IN (...)`, это может тормозить разбор. Решение — переделать на временную таблицу.
2.  **Фазы запроса:** Разбор -> Сканирование -> Агрегация.
3.  **Оптимизация сканирования:**
    - **Использование партиций (Partition pruning):** Классическая оптимизация БД.
    - **Использование первичного ключа (PRIMARY KEY):** Основная техника оптимизации ClickHouse. Смотрим на данные `parts`, `granules`.
    - **Использование data skipping indexes:** Оптимизируют ряд условий (сравнение, `IN` и т.д.).
4.  **Перекос нагрузки (для Distributed tables):** Лог содержит данные со всех хостов. Важно обращать внимание, не выполняют ли какие-то хосты работу дольше других.

## 3. Встроенный профайлер

Позволяет увидеть, на каких внутренних функциях (символах) проводится больше всего времени.

**Важно:** Снятие профиля — CPU-intensive операция. Использование на production-сервере вызывает существенную деградацию производительности.

### 3.1. Хранение данных

Данные профилирования хранятся в системной таблице:
**`system.trace_log`**

Пример структуры записи:
- `hostname`, `event_date`, `event_time`
- `trace_type`: `Real` (реальное время) или `CPU`
- `thread_id`, `thread_name`: `TCPHandler`, `QueryPipelineEx`
- `query_id`
- `trace`: Массив адресов в памяти.
- `symbols`: Массив имён функций (если включена символизация).
- `lines`: Массив имен бинарных файлов.

### 3.2. Получение читаемых имен функций

Чтобы вместо адресов памяти увидеть имена функций, нужно:
1.  Включить `<symbolize>1</symbolize>` в конфигурации для `trace_log`.
2.  Использовать функцию `demangle` в запросе:
    ```sql
    SELECT arrayMap(x -> demangle(addressToSymbol(x)), trace) from system.trace_log
    ```

### 3.3. Диагностика запроса

Можно анализировать, какие агрегатные функции потребляют больше всего ресурсов.

1.  Выполняем проблемный запрос и получаем его `query_id`.
2.  Делаем выборку из `system.trace_log` для этого `query_id`, фильтруя по символам интересующих функций (`topK`, `uniq`):
    ```sql
    SELECT
        count(1) AS count,
        thread_name,
        symbol
    FROM (
        SELECT
            thread_name,
            arrayJoin(symbols) AS symbol
        FROM system.trace_log
        WHERE (query_id = 'b9e83c0b-...')
          AND (trace_type = 'CPU')
          AND (symbol LIKE '%AggregateFunction%')
          AND ((symbol LIKE '%topK%') OR (symbol LIKE '%uniq%'))
          AND (thread_name = 'QueryPipelineEx')
    )
    GROUP BY 2, 3
    ORDER BY count DESC;
    ```
    Результат покажет, какая функция вызывалась чаще (например, `AggregateFunctionTopKGeneric` против `AggregateFunctionUnique`).

### 3.4. Визуализация

- **Каноничный инструмент:** `flamegraph.pl` от Brendan Gregg.
- **Современный инструмент:** **speedscope.app**

  Для подготовки данных нужно выполнить запрос:
  ```sql
  clickhouse client --query "
  SELECT arrayJoin(flameGraph(arrayReverse(trace)))
  FROM system.trace_log
  WHERE trace_type = 'CPU' AND query_id = '<query_id>'
  SETTINGS allow_introspection_functions=1" > 1.flame
  ```
  Полученный файл `1.flame` можно загрузить в приложение speedscope.app для визуального анализа.

## 4. План запроса (EXPLAIN)

Команда для получения плана выполнения запроса без его фактического запуска.

**Синтаксис:**
`EXPLAIN [AST | SYNTAX | PLAN | PIPELINE] [setting = value, ...] SELECT ...`

### 4.1. Виды EXPLAIN

- **`EXPLAIN <по умолчанию>`** (или `PLAN`)
  Показывает ступенчатый план запроса.
  ```
  Expression (Project names + Projection))
  Aggregating
  Expression (Before GROUP BY)
  Filter (WHERE ...)
  ReadFromMergeTree (default.youtube)
  ```

- **`EXPLAIN indexes=1`**
  Расширенная версия плана, показывающая, как использовались индексы (партиционирование, первичный ключ).
  - `MinMax`: Использование партиций.
  - `Partition`: Отбор по ключу партиционирования.
  - `PrimaryKey`: Отбор по первичному ключу (`Parts`, `Granules`, `Ranges`).
  ```
  ReadFromMergeTree (default.youtube)
  Indexes:
    MinMax
      Condition: (upload_date in [18779, 18779])
      Parts: 8/185
      Granules: 168/2933
    Partition
      Condition: (toStartOfMonth(upload_date) in [18779, 18779])
      Parts: 8/8
      Granules: 168/168
    PrimaryKey
      Condition: (upload_date in [18779, 18779])
      Parts: 8/8
      Granules: 168/168
      Ranges: 8
  ```

- **`EXPLAIN AST`**
  Показывает разбор синтаксиса запроса на токены и узлы (Abstract Syntax Tree). Полезно для отладки парсера.

- **`EXPLAIN SYNTAX`**
  Показывает SQL-вид запроса после обработки встроенным оптимизатором ClickHouse.

- **`EXPLAIN ESTIMATE`**
  Показывает, сколько строк, гранул и партов будет прочитано. **Не показывает ожидаемое время выполнения.**

- **`EXPLAIN PIPELINE`**
  Показывает, сколько и каких тредов будет использовано для выполнения запроса.
```
