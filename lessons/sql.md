## Диалекты
ClickHouse поддерживает 2 языка запросов (диалекта):
- **SQL** (Structured Query Language) – классический язык запросов для баз данных. ClickHouse поддерживает не все базовые операции. UPDATE и DELETE представлены альтернативами через `ALTER TABLE ... UPDATE|DELETE`.
- **PRQL** (Pipelined Relational Query Language) – альтернативный классическому SQL язык запросов. Внутри ClickHouse преобразуется к SQL вокруг цепочки подзапросов.

### Переключение диалекта
- `SET dialect='clickhouse';`
- `SET dialect='prql';` (default)

Пример PRQL-запроса:
```prql
from sales
filter date >= '2023-01-01'
group product_id (
  total_quantity = sum quantity
  average_price = average price
)
sort total_quantity -1
take 10;
```

## CRUD операции
- **SELECT** – операция чтения данных.
- **INSERT** – операция записи данных.
- **UPDATE** – операция обновления данных.
- **DELETE** – операция удаления данных.

### SELECT
Позволяет получить данные из ClickHouse, выполнив над ними заданные преобразования.

**Полный синтаксис:**
```sql
[WITH expr_list|(subquery)] 
SELECT [DISTINCT [ON (column1, column2, ...)]] expr_list 
[FROM [db.]table | (subquery) | table_function] [FINAL] 
[SAMPLE sample_coeff] 
[ARRAY JOIN ...] 
[GLOBAL] [ANY|ALL|ASOF] [INNER|LEFT|RIGHT|FULL|CROSS] [OUTER|SEMI|ANTI] JOIN (subquery)|table (ON <expr_list>)|(USING <column_list>) 
[PREWHERE expr] 
[WHERE expr] 
[GROUP BY expr_list] [WITH ROLLUP|WITH CUBE] [WITH TOTALS] 
[HAVING expr] 
[ORDER BY expr_list] [WITH FILL] [FROM expr] [TO expr] [STEP expr] [INTERPOLATE [(expr_list)]] 
[LIMIT [offset_value, ]n BY columns] 
[LIMIT [n, ]m] [WITH TIES] 
[SETTINGS ...] 
[UNION ...] 
[INTO OUTFILE filename [COMPRESSION type [LEVEL level]] ] 
[FORMAT format]
```

### INSERT
Позволяет записать данные в ClickHouse.

**Полный синтаксис:**
```sql
INSERT INTO [TABLE] [db.]table [(c1, c2, c3)] [SETTINGS ...] 
VALUES (v11, v12, v13), (v21, v22, v23), ...

INSERT INTO [TABLE] [db.]table [(c1, c2, c3)] [SETTINGS ...] 
FORMAT format

INSERT INTO [TABLE] [db.]table [(c1, c2, c3)] [SETTINGS ...] 
SELECT ...
```

### UPDATE
**Предостережение:** В ClickHouse нет точечного обновления данных. Операция обновления реализована как мутация, требующая свободного места не менее Х2 от объема мутируемых партиций. Мутация требует достаточного времени и реализована как полная перезапись всех данных партиций, затронутых обновлением. Без указания `IN PARTITION` будут перезаписаны ВСЕ данные таблицы.

**Best practice:** Не использовать ClickHouse для мутабельных данных.

### DELETE
**Предостережение:** Аналогично UPDATE.

**Best practice:** Не использовать ClickHouse для мутабельных данных.

#### Lightweight Delete
- Не удаляет данные физически моментально, а маркирует как удаленные, скрывая их от следующих SELECT’ов.
- Физическое удаление происходит в фоновом режиме во время слияния.
- Работает только с MergeTree-family.

**Синтаксис:**
```sql
DELETE FROM [db.]table [ON CLUSTER cluster] [IN PARTITION partition_expr] WHERE expr;
```

## Синтаксис

### Секция SELECT: WITH
Позволяет вычислить значения выражений, которые можно использовать дальше в запросе.

**Варианты:**
```sql
WITH c1*100/c1_total AS c1_percentage 
SELECT c1_percentage FROM table

WITH subq AS (subquery) 
SELECT c1, c2, c3 FROM subq
```

### Секция DISTINCT
Альтернативный вариант GROUP BY.

**Варианты:**
- `SELECT DISTINCT c1, c2 FROM table` эквивалентно `SELECT c1, c2 FROM table GROUP BY c1, c2`
- `SELECT DISTINCT ON (c1, c2) c3 FROM table` эквивалентно `SELECT any(c3) FROM table GROUP BY c1, c2`

### Секция column expression APPLY модификатор
Позволяет применить функцию к нескольким столбцам сразу.

**Пример:**
```sql
SELECT name, * APPLY toTypeName FROM system.metrics LIMIT 1
```

### Секция FROM
Поддерживает выборку из таблиц, табличных функций, подзапросов, джоины.

**Синтаксис:**
```sql
FROM [db.]table | (subquery) | table_function
```

#### Табличные функции
Позволяют создать на лету таблицу для обращения за данными.

**Примеры:**
- `s3(path [, NOSIGN | aws_access_key_id, aws_secret_access_key [,session_token]] [ ,format] [,structure] [,compression])` – для обращения к S3.
- `file([path_to_archive :::] path [,format] [,structure] [,compression])` – для работы с файлом из каталога, указанного в конфигурации `user_files_path`.

### Секции PREWHERE/WHERE/HAVING
Набор условий для фильтрации выборки запрашиваемых данных:
- **PREWHERE** – условие, выполняемое до WHERE.
- **WHERE** – условие, выполняемое до агрегации.
- **HAVING** – условие, выполняемое после агрегации.

### Секция GROUP BY
Уникализирует выборку по списку указанных столбцов. Требование: остальные столбцы должны быть под агрегирующей функцией.

## Создание и удаление баз и таблиц

### Создание и удаление БД

**Синтаксис:**
- **Создание:**
  ```sql
  CREATE DATABASE [IF NOT EXISTS] db_name 
  [ON CLUSTER cluster] 
  [ENGINE = engine(...)] 
  [COMMENT 'Comment']
  ```
- **Удаление:**
  ```sql
  DROP DATABASE [IF EXISTS] db 
  [ON CLUSTER cluster] 
  [SYNC]
  ```

**Модификаторы:**
- `IF NOT EXISTS` / `IF EXISTS` – проверяет наличие/отсутствие БД с таким именем.
- `ON CLUSTER` – требуется подключение к ZooKeeper. Добавляет в DISTRIBUTED DDL очередь задачу на выполнение такого же запроса на все реплики кластера.
- `ENGINE` – Database Engine создаваемой базы.
- `SYNC` – меняет поведение по умолчанию при удалении таблиц для баз на `Engine=Atomic`. По умолчанию удаление происходит в фоновом режиме, с `SYNC` – запрос выполняется до полного удаления.

#### ENGINE баз
- **Atomic** – база данных по умолчанию. Хранение через UUID таблиц, атомарные операции.
- **Ordinary** – legacy Engine, требует `allow_deprecated_database_ordinary` в конфигурации.
- **MySQL, PostgreSQL** – специальные Engine для отображения баз из других систем.
- **MaterializedMySQL, MaterializedPostgreSQL** – репликация данных из других систем (экспериментальные).
- **Replicated** – Engine с реплицируемыми запросами CREATE/DROP таблиц (экспериментальный).
- **Lazy** – только для таблиц `Engine=Log`. Держит в оперативной памяти свежие данные.

### Создание и удаление таблиц
(Синтаксис не раскрыт подробно в предоставленных материалах.)

## Модификация таблиц
**Добавление и удаление столбцов – синтаксис:**
- `ADD COLUMN [IF NOT EXISTS] name [type] [default_expr] [codec] [AFTER name_after | FIRST]`
- `DROP COLUMN [IF EXISTS] name`
- `CLEAR COLUMN name` – удаляет данные в столбце.
- `MATERIALIZE COLUMN name` – рассчитать и сохранить MATERIALIZED выражение для столбца.
- `MODIFY COLUMN name type default_expr` – меняет тип столбца и выражение DEFAULT/MATERIALIZED.

## Операции над партами и партициями
**Синтаксис:** `ALTER TABLE [db.]table ...`
- `DROP PARTITION|PART` – удаляет партицию/парт из таблицы. Поддерживает модификатор `SYNC` для `database Engine=Atomic`.
- `ATTACH PARTITION|PART` – добавляет вручную подключенные в `.../table_path/detached` на файловой системе партиции и парты.
- `REPLACE PARTITION ... FROM table2` – заменяет партицию копией из другой таблицы. Структуры таблиц должны совпадать.
- `DETACH PARTITION|PART` – перемещает партицию/парт в каталог `.../table_path/detached` на файловой системе. Происходит на всех репликах шарда.

## Рефлексия
**Вопросы для проверки:**
1. Какие SQL операции из рассмотренных на вебинаре разительно отличаются от знакомых вам по другим базам данных?
