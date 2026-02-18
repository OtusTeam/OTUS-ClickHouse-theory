## 1. Классический логгер (файловый)

Это стандартный механизм логирования, который включается в конфигурации сервера секцией `<logger>...</logger>`.

**Возможности:**
- Логирование в text-лог и error-лог.
- Ротация лог-файлов.
- Настройка уровней логирования: `none`, `fatal`, `critical`, `error`, `warning`, `notice`, `information`, `debug`, `trace`, `test`.
- Логирование в `stdout`/`stderr` и `syslog`.

### Простая конфигурация
```xml
<logger>
    <level>trace</level>
    <log>/var/log/clickhouse-server/clickhouse-server.log</log>
    <errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>
    <size>1000M</size>
    <count>10</count>
</logger>
```
**Параметры:**
- `level`: уровень логирования.
- `log`: путь к файлу text-лога.
- `errorlog`: путь к файлу error-лога.
- `size`: размер, при достижении которого начинается новый лог-файл (старый переименовывается).
- `count`: количество хранимых файлов логов (более старые удаляются).

### Продвинутые конфигурации (шаблонные имена файлов)
В именах файлов можно использовать шаблонные подстановки:
- `%F` - дата в формате YYYY-MM-DD (например, 2023-07-06).
- `%T` - время в формате HH:MM:SS (например, 18:32:07).
- `%H` - час.
- `%M` - минута.
- `%S` - секунда.

Пример:
```xml
<log>/var/log/clickhouse-server/clickhouse-server-%F-%T.log</log>
<errorlog>/var/log/clickhouse-server/clickhouse-server-%F-%T.err.log</errorlog>
```

## 2. Табличный логгер

Логи записываются не в файлы, а в специальные таблицы ClickHouse. Включается секциями конфигурации вида `<тип_лога_log> ... </тип_лога_log>`.

### Наиболее часто используемые:
- `query_log`
- `text_log`
- `part_log`

### Полный список системных таблиц для логов (зависит от версии):
- `asynchronous_insert_log`
- `asynchronous_metric_log`
- `backup_log`
- `blob_storage_log`
- `crash_log` (логи падений, если используется watchdog)
- `error_log` (аналог error-лога в виде таблицы)
- `metric_log`
- `opentelemetry_span_log`
- `part_log` (история фоновых операций над MergeTree-таблицами: Merge, TTL, мутации)
- `processors_profile_log`
- `query_log` (лог запросов, если `log_queries=1`)
- `query_metric_log`
- `query_thread_log` (лог тредов, если `log_query_threads=1`)
- `query_views_log` (лог запросов от view, если `log_query_views=1`)
- `s3queue_log`
- `text_log` (аналог текстового лога в виде таблицы)
- `trace_log` (трейсы от профайлера)
- `zookeeper_log`

### Настройка табличного логгера (на примере `query_log`)
```xml
<query_log>
    <database>system</database>
    <table>query_log</table>
    <engine>
        Engine = MergeTree
        PARTITION BY event_date
        ORDER BY event_time
        TTL event_date + INTERVAL 30 day
        SETTINGS index_granularity = 8192, ttl_only_drop_parts = 1
    </engine>
    <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    <max_size_rows>1048576</max_size_rows>
    <flush_on_crash>false</flush_on_crash>
</query_log>
```
**Параметры:**
- `database`: база данных для таблицы (обычно `system`).
- `table`: имя таблицы (обычно одноименное с типом лога).
- `engine`: движок таблицы (обычно `MergeTree` с настройками партиционирования, сортировки и TTL).
- `flush_interval_milliseconds`: интервал сброса событий в таблицу.
- `max_size_rows`: принудительный сброс при накоплении указанного количества событий.
- `flush_on_crash`: попытка записать события при падении.

### Часто используемые поля в `query_log`
- `event_time`, `event_date`: время и дата события.
- `query_id`: ID запроса (используется для поиска по другим логам).
- `exception_code`: код ошибки (0 — успех).
- `exception`: текст ошибки.
- `ProfileEvents`: счетчики затраченных ресурсов.
- `type`: тип события (`QueryStart`, `QueryFinish`, `ExceptionBeforeStart`, `ExceptionWhileProcessing`).

## 3. Встроенный профайлер и `trace_log`

В ClickHouse встроен семплирующий профайлер, который снимает трейсы со всех тредов запросов.

**Включение и настройка:**
1.  Необходимо объявить секцию `<trace_log>` в конфигурации.
2.  Настройка периода семплирования через `settings`:
    ```sql
    SET query_profiler_cpu_time_period_ns = 1000000; -- каждые N нс процессорного времени
    SET query_profiler_real_time_period_ns = 1000000; -- каждые N нс реального времени
    ```
    (В новых версиях также есть `memory_profiler_%`).
3.  Для сбора и чтения `trace_log` должен быть установлен пакет `clickhouse-common-static-dbg`.

**Анализ данных:**
1.  Включить функции интроспекции: `SET allow_introspection_functions = 1`.
2.  Распарсить колонку `trace` с помощью функций:
    ```sql
    SELECT arrayStringConcat(arrayMap(x -> demangle(addressToSymbol(x)), trace), '\n')
    FROM system.trace_log LIMIT 1 FORMAT Vertical
    ```

**Применение:**
- Выявление самых долгих мест выполнения запроса (часто встречающиеся строки в трейсе).
- Анализ группы однотипных запросов.
- Сопоставление `query_id` и `thread_id` с `query_log` и `thread_log`.
- Использование сторонних инструментов для визуализации (например, `clickhouse-flamegraph`).

**⚠️ Внимание!** Снятие трейсов — CPU-интенсивная операция. Частый профайлинг на нагруженном инстансе приведет к деградации скорости запросов.

## 4. Получение логов на один запрос

### Логи в консоли клиента
В `clickhouse-client` можно включить вывод лога для конкретного запроса:
```sql
SET send_logs_level = 'trace';
SELECT ... FORMAT Null; -- FORMAT Null удобен, если результат не нужен
```

### Анализ ключевых моментов в логе запроса
На примере запроса `SELECT count() FROM ... WHERE upload_date = '...'`:

**1. Начальная стадия (интерпретация):**
- Принят запрос, проверены права, определены колонки для чтения.
- *Что искать:* если эта стадия слишком долгая, запрос тяжел для парсера (стоит переписать длинные `IN (...)`).

**2. Работа с индексом:**
- Поиск по индексу, определение количества частей (`parts`) и засечек (`marks`) для чтения.
- *Что искать:*
    - `Selected ... parts by primary key, ... marks to read`: показывает, насколько эффективно сработал индекс.
    - Если этой строки нет или читается 100% засечек — происходит полное сканирование таблицы (fullscan).

**3. Выборка и агрегация:**
- Чтение данных в несколько потоков, агрегация, вывод итоговых счетчиков.
- *Что искать:*
    - **Метки времени:** каждая строка лога имеет метку. Разница во времени между этапами показывает, сколько заняла каждая операция.
    - **Медленные реплики:** при работе с `Distributed`-таблицами, лог приходит со всех реплик (идентификатор хостнейма в логе). Сравнение меток времени помогает найти медленные реплики.

## 5. Встроенный дашборд

Доступен по URL: `http://clickhouse-server:port/dashboard` (порт по умолчанию: 8123).

- Отображает графики на основе запросов, возвращающих `DateTime` и числовое значение.
- Содержит встроенный пресет запросов из `system.dashboards`.
- Можно добавлять и менять запросы прямо в веб-интерфейсе (кодируются в URL, не сохраняются).

**Примеры доступных метрик (зависят от версии):**
- Queries/second
- CPU Usage (user/system, wait)
- Queries Running
- Merges Running
- Selected/Inserted Rows/ Bytes per second
- Read From Disk/FS
- IO Wait
- Memory usage
- Total parts / Max Parts for Partition

## 6. Интеграция с системами хранения метрик

ClickHouse поддерживает сбор метрик внешними системами, такими как Graphite и Prometheus. Данные забираются из таблиц `system.metrics`, `system.events` и `system.asynchronous_metrics`.

### Конфигурация для Graphite
```xml
<graphite>
    <host>localhost</host>
    <port>42000</port>
    <timeout>0.1</timeout>
    <interval>60</interval>
    <root_path>one_min</root_path>
    <metrics>true</metrics>
    <events>true</events>
    <events_cumulative>false</events_cumulative>
    <asynchronous_metrics>true</asynchronous_metrics>
</graphite>
```

### Конфигурация для Prometheus
```xml
<prometheus>
    <endpoint>/metrics</endpoint>
    <port>9363</port>
    <metrics>true</metrics>
    <events>true</events>
    <asynchronous_metrics>true</asynchronous_metrics>
    <errors>true</errors>
</prometheus>
```

### Системные таблицы метрик
- `system.metrics`: текущие значения метрик (name, value, description).
- `system.events`: счетчики произошедших событий (name, value, description).
- `system.asynchronous_metrics`: асинхронно обновляемые метрики (name, value, description).

### Персонализация мониторинга для Prometheus
Можно создать свой endpoint, который будет отдавать произвольные метрики в формате Prometheus.

1.  **Настройка HTTP-хендлера** в конфиге:
    ```xml
    <http_handlers>
        <rule>
            <url>/custom_metrics</url>
            <methods>POST,GET</methods>
            <handler>
                <type>predefined_query_handler</type>
                <query>SELECT * FROM system.custom_prom_metrics FORMAT Prometheus</query>
                <content_type>text/plain; charset=utf-8</content_type>
            </handler>
        </rule>
    </http_handlers>
    ```
2.  **Создание источника данных:**
    - База данных: `CREATE DATABASE custom_prom_metrics;`
    - Представления (`VIEW`) с метриками, например:
        ```sql
        CREATE VIEW custom_prom_metrics.merges AS
        SELECT
            'merges' AS name,
            count() AS value,
            'active merges' AS help,
            map('hostname', hostName()) AS labels,
            'gauge' AS type
        FROM system.merges;
        ```
    - Финишная таблица, объединяющая все VIEW: `CREATE TABLE system.custom_prom_metrics ENGINE = merge(custom_prom_metrics, '.*');`

### Готовый дашборд для Grafana
Доступен по ссылке: https://grafana.com/grafana/dashboards/14192-clickhouse/
```
