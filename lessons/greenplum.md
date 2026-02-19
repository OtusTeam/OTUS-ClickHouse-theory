## 1. PostgreSQL и Greenplum

### 1.1. Введение в PostgreSQL
- **Истоки:** Майк Стоунбрейкер, Беркли.
- **История развития:**
    - Начало 1970-х: Ingres (INteractive Grafic REtrieval System).
    - Середина 1980-х: Postgres (Post Ingres).
    - Postgres95: добавлен SQL.
    - 1996: Основан домен postgresql.org.
- **Современное состояние:** Развивается под управлением PostgreSQL Global Development Group.
- **Основные компании-контрибьюторы:**
    - 2ndQuadrant (куплен EnterpriseDB)
    - EnterpriseDB
    - Crunchy Data
    - Postgres Professional
- **Общая характеристика:** PostgreSQL — это мощная объектно-реляционная СУБД с открытым исходным кодом, известная своей надежностью, производительностью и расширяемостью.
- **Области применения:** Веб-разработка, аналитические системы, хранение геоданных, финансовые приложения, высоконагруженные системы.

### 1.2. Greenplum
- **Определение:** Open source MPP-СУБД (массивно-параллельная реляционная СУБД) для хранилищ данных.
- **Ключевые особенности:**
    - Отличная горизонтальная масштабируемость.
    - Столбцовое хранение данных на основе PostgreSQL.
    - Мощный оптимизатор запросов.
    - Надежность.
    - Высокая скорость обработки SQL-запросов над большими объемами данных.
    - Широко применяется для промышленной аналитики Big Data.

- **История Greenplum:**
    - **2005:** Первый выпуск технологий компанией Greenplum (Калифорния).
    - **2010:** Поглощение корпорацией EMC.
    - **2011:** Релиз бесплатной версии Greenplum Community Edition от EMC.
    - **2012:** Приобретение продукта корпорацией Pivotal, развитие под брендом Pivotal.
    - **2015:** Публикация исходного кода под лицензией Apache.
    - **2018:** Интеграция с платформой Luxms BI. Российская компания «Аренадата Софтвер» выпустила Arenadata DB на основе Greenplum.
    - **2020:** Приобретение Pivotal корпорацией VMware. Коммерциализация под брендом VMware Tanzu Greenplum.

### 1.3. Архитектура Greenplum
- **Основа:** Кластер из нескольких взаимосвязанных экземпляров PostgreSQL.
- **Принципы:**
    - MPP (Massive Parallel Processing).
    - Shared Nothing (без разделения ресурсов: у каждого узла своя память, ОС и диски).
    - Наличие резервного мастер-сервера для надежности.
- **Состав кластера:**
    1.  **Мастер-сервер (Master host):** Точка входа для клиентов. Координирует работу, распределяет нагрузку между сегментами. Пользовательских данных не хранит.
    2.  **Резервный мастер (Secondary master instance):** Инстанс PostgreSQL, включаемый вручную при отказе основного мастера.
    3.  **Сервер-сегмент (Segment host):** Хранит и обрабатывает данные.
        - На одном хосте может быть 2-8 сегментов (независимых экземпляров PostgreSQL).
        - **Primary-сегмент:** Обрабатывает свою часть данных.
        - **Mirror-сегмент (зеркало):** Соответствует каждому primary, автоматически включается при его отказе.
    4.  **Интерконнект (Interconnect):** Быстрое обособленное сетевое соединение между инстансами PostgreSQL.

### 1.4. Распределение данных и запросы
- **Ключ дистрибуции:** Логика разбиения таблицы на сегменты. Выбирается по принципу равномерного распределения значений, так как именно он задает распределение данных по сегментам кластера.
- **Прохождение запроса:** (Визуальная схема, описывающая взаимодействие клиента, мастера и сегментов).

### 1.5. Применение и недостатки Greenplum
- **Применение:**
    - Системы предиктивной аналитики и регулярной отчетности.
    - Построение Data Lake и корпоративных хранилищ данных (КХД).
    - Разработка аналитических моделей (например, прогнозирование оттока клиентов).
- **Недостатки:**
    - Высокие требования к ресурсам (ЦП, память, диски, сетевая инфраструктура).
    - Низкая производительность при большом объеме простых запросов (каждая транзакция на мастере порождает множество транзакций на сегментах).
    - Неоптимальное распределение сегментов может негативно сказаться на производительности при расширении кластера.

## 2. Интеграции ClickHouse с PostgreSQL и Greenplum

### 2.1. Движки для интеграции с PostgreSQL

- **Table Engine `PostgreSQL`:**
    - Данные хранятся в PostgreSQL.
    - Метаданные в зависимости от настроек хранятся в ClickHouse или узнаются во время запроса.
- **Database Engine `PostgreSQL`:**
    - Позволяет создать view в реальном времени на все таблицы базы данных в PostgreSQL.

- **Table Engine `MaterializedPostgreSQL` (Experimental):**
    - Данные и метаданные хранятся в ClickHouse.
- **Database Engine `MaterializedPostgreSQL` (Experimental):**
    - Позволяет создать материализованное представление базы данных PostgreSQL на диске ClickHouse.

### 2.2. Примеры использования
- **Создание таблицы на основе PostgreSQL:**
  ```sql
  CREATE TABLE postgres_table ENGINE = PostgreSQL('postgres1:5432', 'postgres_database', 'postgres_table', 'user', 'pwd')
  ```
- **Создание словаря из PostgreSQL:**
  ```sql
  CREATE DICTIONARY postgres_dict (id UInt32, val UInt32)
  PRIMARY KEY id
  SOURCE(POSTGRESQL(port 5432 host 'localhost' user 'postgres_user' password 'pwd' db 'postgres_database' table 'postgresql_table'))
  LIFETIME(MIN 1 MAX 2)
  LAYOUT(HASHED());
  ```

### 2.3. Репликация данных из PostgreSQL в ClickHouse
- **Особенности работы с изменениями данных:**
    - В ClickHouse по умолчанию доступна только вставка данных, но не их изменение (UPDATE/DELETE).
    - Для работы с CDC (Change Data Capture) используется движок `ReplacingMergeTree`, который выполняет фоновую дедупликацию на основе ключа сортировки и виртуальных колонок `sign` и `version`.
- **Ключевое слово FINAL:**
    - При запросах к материализованной реплике для получения актуальных данных (с учетом дедупликации) необходимо использовать модификатор `FINAL`.
    ```sql
    CREATE TABLE postgresql_replica ENGINE = MaterializedPostgreSQL('postgres1:5432', 'postgres_database', 'postgres_table', 'user', 'pwd')
    SELECT * FROM postgresql_replica WHERE ... FINAL
    ```

### 2.4. Итоги по интеграции
- **Быстрые запросы в ClickHouse над данными из Postgres в реальном времени:**
    - Движок БД (`Database`).
    - Движок таблиц (`Table`).
    - Табличная функция `postgresql`.
    - Использование в словарях.
- **ClickHouse как реплика Postgres:**
    - Движок таблиц `Materialized`.
    - Движок БД `Materialized`.
    - Изменения на стороне публикации передаются в реальном времени. Вставка данных не блокирует их чтение.

## 3. Материалы для изучения
- [https://pgday.ru/presentation/281/60f04d3f07516.pdf](https://pgday.ru/presentation/281/60f04d3f07516.pdf)
- [https://github.com/mingpeng2live/pxf-clickhouse](https://github.com/mingpeng2live/pxf-clickhouse)
- [https://clickhouse.com/docs/en/integrations/postgresql](https://clickhouse.com/docs/en/integrations/postgresql)
- [https://clickhouse.com/blog/migrating-data-between-clickhouse-postgres](https://clickhouse.com/blog/migrating-data-between-clickhouse-postgres)
- [https://clickhouse.com/docs/en/engines/table-engines/integrations/postgresql](https://clickhouse.com/docs/en/engines/table-engines/integrations/postgresql)
- [https://clickhouse.com/blog/clickhouse-postgresql-change-data-capture-cdc-part-1](https://clickhouse.com/blog/clickhouse-postgresql-change-data-capture-cdc-part-1)
- [https://clickhouse.com/blog/clickhouse-postgresql-change-data-capture-cdc-part-2](https://clickhouse.com/blog/clickhouse-postgresql-change-data-capture-cdc-part-2)
