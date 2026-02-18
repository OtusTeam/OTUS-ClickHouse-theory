## 1. Шардирование

### 1.1. Что такое шардирование?
**Шардирование** — это расширение доступного места под данные за счет размещения данных на нескольких экземплярах приложения с независимым хранением. Как правило, это реализуется путем размещения экземпляров приложения на разных серверах.

### 1.2. Engine=Distributed
`Distributed` — это виртуальная таблица, которая выступает в роли «агрегирующего прокси». Её функции:
- Принимает запросы.
- Повторяет запросы в таблицы с данными согласно топологии кластера.
- Объединяет результаты с шардов.
- Возвращает объединенный результат.

**Синтаксис создания:**
```sql
CREATE TABLE имя_таблицы ( колонки ) Engine=Distributed( аргументы )
```

### 1.3. Аргументы Engine=Distributed
`Distributed( cluster_name, database, table, sharding_key )`

*   **`cluster_name`**: Имя кластера, описанное в конфигурации в секции `<remote_servers>`. Можно выбрать любое имя и использовать разные топологии с разными Distributed-таблицами.
*   **`database`, `table`**: Имя базы данных и таблицы, в которую будет повторен запрос (на одной реплике каждого шарда).
*   **`sharding_key`**: Необязательный параметр для `SELECT`, но необходимый для `INSERT`.
    *   Должен быть числового типа.
    *   Можно использовать хеширующие функции от других колонок.
    *   Данные нарезаются на шарды по номеру шарда, получаемому как остаток от деления ключа шардирования на количество шардов.

### 1.4. Описание топологии кластера (конфигурация)
Топология описывается в конфигурационном файле ClickHouse (на серверах с Distributed-таблицами) в секции `<remote_servers>`.

**Пример структуры:**
```xml
<remote_servers>
    <cluster_name>
        <shard>
            <replica>
                <host>ch-server-1.zone</host>
                <port>9000</port>
            </replica>
            <!-- replica 2 ... -->
            <!-- replica 3 ... -->
        </shard>
        <!-- one more shard ... -->
        <!-- more shards ... -->
    </cluster_name>
    <!-- more clusters ... -->
</remote_servers>
```

### 1.5. Доступы и права пользователей
*   Если Distributed-таблица находится на отдельном сервере (не там, где хранятся данные), дублировать пользователей на серверы с данными не обязательно.
*   Distributed-таблица порождает запросы к репликам от имени пользователя `default` или пользователя, указанного в секции `<user>` конфигурации реплики.
*   Для работы `quotas-policy` пользователь все равно должен существовать, так как он передается как `initial_user` на реплики.

### 1.6. Удаление и добавление шардов/реплик
Процесс достигается редактированием конфигурационного файла на всех серверах с Distributed-таблицами.
**Важно:** Целевые таблицы на новых серверах должны быть созданы **заранее**. ClickHouse не создаст их автоматически. В противном случае будет ошибка `DB::Exception: нет таблицы на remote реплика`.

### 1.7. Замена отказавшей реплики (алгоритм)
1.  В `system.replicas` живой реплики подсмотреть имя мертвой реплики и список таблиц.
2.  Выполнить запрос `SYSTEM DROP REPLICA 'отказавшая_реплика' FROM table таблица` для каждой таблицы.
3.  Ввести в эксплуатацию новую реплику на замену старой, создав на ней все необходимые таблицы.
4.  Дождаться репликации таблиц.
5.  Заменить старую реплику на новую в секции `remote_servers` конфигурации.

## 2. Распределенные запросы

### 2.1. GLOBAL IN / GLOBAL JOIN
*   **Проблема:** Запрос вида `SELECT ... FROM distributed_table WHERE IN ( SELECT ... )` будет передан на **каждый шард**. Подзапрос `(SELECT ...)` выполнится многократно (по числу шардов).
*   **Решение:** Модификатор `GLOBAL IN ( подзапрос )` (и аналогично `GLOBAL JOIN`) сначала выполняет подзапрос на инициаторе, а затем передает его результат на все шарды как временную таблицу для переиспользования.

### 2.2. Настройка distributed_product_mode
Меняет поведение по умолчанию для `IN`/`JOIN` запросов.
*   **`deny`** (по умолчанию): Возвращает исключение при попытке использовать `GLOBAL`.
*   **`local`**: Запрещает `GLOBAL`, но в подзапросах на шардах заменяет distributed-таблицы на их локальные (`local`) таблицы.
*   **`global`**: Заменяет обычные `IN`/`JOIN` на `GLOBAL IN`/`GLOBAL JOIN`.
*   **`allow`**: Разрешает пользователю выбирать самостоятельно.

### 2.3. Настройка prefer_global_in_and_join
*   При `distributed_product_mode=global` не учитываются таблицы с Engine для доступа к внешним ресурсам (например, `Engine=mysql`).
*   `prefer_global_in_and_join=1` включает такое же поведение (принудительный `GLOBAL`) и для этих движков.
*   `prefer_global_in_and_join=0` (по умолчанию) — отключено.

## 3. Особенности шардирования

### 3.1. Взаимосвязь с репликацией
**Взаимосвязи нет.** ClickHouse не проверяет соответствие топологии в конфиге и настроек репликации таблиц.
*   Набор взаимореплицируемых таблиц на шарде — это самостоятельный набор, независимый от других шардов.
*   Для уникализации путей таблиц разных шардов в **Keeper** рекомендуется использовать макрос `<shard>` при создании таблиц (например, в пути к ReplicatedMergeTree).

### 3.2. Очередь Distributed таблиц
*   Под капотом `Engine=Distributed` используется неоптимальный формат для нарезки вставляемых данных.
*   Это может быть узким местом и причиной реализации шардирования на стороне клиента, в обход Distributed-таблиц.
*   Отслеживать состояние очереди можно в таблице `system.distribution_queue`. Важно строить метрики на ее основе.

### 3.3. Оптимизация агрегации (distributed_group_by_no_merge)
*   Для запросов с агрегацией по ключу шардирования (если данные действительно шардированы по этому ключу и ключи уникальны между шардами) не требуется финальная агрегация на сервере-инициаторе.
*   Настройка `distributed_group_by_no_merge=1` отключает эту финальную агрегацию, что значительно ускоряет запросы (например, `uniq()` по ключу шардирования).
*   Если ключи на разных шардах повторяются, результат будет некорректным (по несколько строк на одно значение ключа).

### 3.4. Решардинг (перешардирование)
**Встроенного решардинга (изменения количества шардов для существующих данных) в ClickHouse НЕТ.**
**Способы:**
1.  **clickhouse-copier** (рекомендуемый в документации): создание нового кластера и переливка данных с помощью этой утилиты, которая генерирует `INSERT SELECT` запросы.
2.  **Копирование таблиц:** `CREATE TABLE AS` + `ALTER TABLE ATTACH PARTITION FROM`, дублирование данных через репликацию, удаление лишнего через `ALTER TABLE DELETE`.
3.  **Перенос партиций:** Скрипты с `DETACH PART/PARTITION`, физический перенос файлов, `ATTACH`.
4.  **Переливка "inplace":** через `INSERT SELECT`.

### 3.5. Подход к шардированию в Яндекс (пример архитектуры)
*   Много маленьких кластеров с использованием макроса `shard`. Сверху них — distributed-таблицы. Такие кластеры называются **`layer`**.
*   Создается общий кластер с использованием макроса `layer`. Сверху него создаются distributed-таблицы, которые ссылаются на distributed-таблицы нижнего уровня (`layer`).
*   В такой концепции становится применима переливка данных между `layer`-ами с помощью `clickhouse-copier`.

## 4. Примеры

### 4.1. Пример создания Distributed и локальной таблиц
**Distributed-таблица (таблица-прокси):**
```sql
CREATE TABLE default.events (
    date Date MATERIALIZED toDate(timestamp),
    ts DateTime,
    event_id UInt64,
    host IPv4,
    response_time_ms UInt32,
    headers Map(String, String),
    another_column String,
    one_more_column Array(String)
) ENGINE=Distributed(cluster_name, default.events_local)
```

**Локальная таблица (на каждом шарде, с репликацией):**
```sql
CREATE TABLE default.events_local (
    date Date MATERIALIZED toDate(timestamp),
    ts DateTime,
    event_id UInt64,
    host IPv4,
    response_time_ms UInt32,
    headers Map(String, String),
    another_column String,
    one_more_column Array(String)
) ENGINE=ReplicatedMergeTree('/ch/{database}/{table}/{shard}', '{replica}')
PARTITION BY date
ORDER BY (date, event_id)
SAMPLE BY event_id
```

### 4.2. Примеры конфигурации топологий (секции <remote_servers>)

**2 шарда, по 2 реплики каждый (`2sh2rep`):**
```xml
<remote_servers>
    <2sh2rep>
        <shard>
            <replica> <host>ch-server-1.zone</host> <port>9000</port> </replica>
            <replica> <host>ch-server-2.zone</host> <port>9000</port> </replica>
        </shard>
        <shard>
            <replica> <host>ch-server-3.zone</host> <port>9000</port> </replica>
            <replica> <host>ch-server-4.zone</host> <port>9000</port> </replica>
        </shard>
    </2sh2rep>
</remote_servers>
```

**1 шард, 4 реплики (`1sh4rep`):**
```xml
<remote_servers>
    <1sh4rep>
        <shard>
            <replica> <host>ch-server-1.zone</host> <port>9000</port> </replica>
            <replica> <host>ch-server-2.zone</host> <port>9000</port> </replica>
            <replica> <host>ch-server-3.zone</host> <port>9000</port> </replica>
            <replica> <host>ch-server-4.zone</host> <port>9000</port> </replica>
        </shard>
    </1sh4rep>
</remote_servers>
```

**3 шарда, по 3 реплики каждый (`3sh3rep`):**
```xml
<remote_servers>
    <3sh3rep>
        <shard>
            <replica> <host>ch-server-1.zone</host> <port>9000</port> </replica>
            <replica> <host>ch-server-2.zone</host> <port>9000</port> </replica>
            <replica> <host>ch-server-3.zone</host> <port>9000</port> </replica>
        </shard>
        <shard>
            <replica> <host>ch-server-4.zone</host> <port>9000</port> </replica>
            <replica> <host>ch-server-5.zone</host> <port>9000</port> </replica>
            <replica> <host>ch-server-6.zone</host> <port>9000</port> </replica>
        </shard>
        <shard>
            <replica> <host>ch-server-7.zone</host> <port>9000</port> </replica>
            <replica> <host>ch-server-8.zone</host> <port>9000</port> </replica>
            <replica> <host>ch-server-9.zone</host> <port>9000</port> </replica>
        </shard>
    </3sh3rep>
</remote_servers>
```

### 4.3. Bash-скрипт для клонирования структуры таблиц на новую реплику
```bash
#!/bin/bash
source_replica=один_хост
dest_replica=другой_хост

for database in $( clickhouse-client --host "${source_replica}" -q "show databases" ); do
    clickhouse-client --host "${dest_replica}" -q "create database if not exists ${database}"
    for table in $( clickhouse-client --host "${source_replica}" -d "${database}" -q "show tables" ); do
        table_sql="$( clickhouse-client --host "${source_replica}" -d "${database}" -q "show create table ${table} format TSVRaw" )"
        clickhouse-client --host "${dest_replica}" -n -d "${database}" <<<"${table_sql}"
    done
done
```
