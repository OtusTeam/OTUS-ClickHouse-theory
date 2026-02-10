## Варианты установки ClickHouse
1. **ClickHouse Cloud**: официальный ClickHouse as a service.
   - Managed-сервисы от облачных провайдеров (Yandex, VK).
2. **Quick Install**: легко загружаемый бинарник для тестирования и разработки.
3. **Production Deployments**:
   - ClickHouse может работать на любом Linux, FreeBSD или macOS с архитектурой CPU x86-64, ARM или PowerPC64LE.
4. **Образ Docker**: официальный образ Docker в Docker Hub.
5. **Homebrew** для macOS.
6. **Кластер ClickHouse на базе Kubernetes**.

### ClickHouse Cloud (Yandex Managed Service)
- Помогает разворачивать и поддерживать кластеры ClickHouse в Yandex Cloud.
- Каждый кластер состоит из одного или нескольких хостов БД (виртуальных машин).
- Позволяет самостоятельно определять количество хостов, технические характеристики, цену доступности и размер хранилища.

### Quick Install (бинарник)
1. Загрузка бинарника для ОС (Linux, macOS, FreeBSD):
   ```
   curl https://clickhouse.com/ | sh
   ```
2. Запуск сервера:
   ```
   ./clickhouse server
   ```
3. Подключение клиента в новом терминале:
   ```
   ./clickhouse client
   ```

### Docker
- Запуск сервера:
  ```
  docker run -d --name some-clickhouse-server --ulimit nofile=262144:262144 clickhouse/clickhouse-server
  ```
- Подключение через нативный клиент или curl.
- Остановка/удаление контейнера:
  ```
  docker stop some-clickhouse-server
  docker rm some-clickhouse-server
  ```

### Production Deployments
- **DEB-пакеты** для Debian и Ubuntu.
- **RPM-пакеты** для CentOS, RedHat и других rpm-дистрибутивов Linux.
- **Tgz-архивы** для дистрибутивов, где установка deb/rpm невозможна.

### ClickHouse Keeper
- Создан осенью 2022 года как замена ZooKeeper.
- Написан на C++, должен работать быстрее и потреблять меньше памяти.
- По умолчанию предоставляет те же гарантии, что и ZooKeeper, и совместимый клиент-серверный протокол.
- В производственной среде рекомендуется запускать на выделенных узлах.
- **Нюансы**: мало документации, часть конфигурации берется из основного конфига ClickHouse, нет режима суперпользователя, не полностью совместим с ZooKeeper.

## Базовая конфигурация, запуск и тестовые датасеты

### Конфигурационные файлы
- `/etc/clickhouse-server/config.xml` – настройки сервера.
- `/etc/clickhouse-server/users.xml` – конфигурация пользователей и настройки сервера, которые можно переопределить для каждого пользователя.
- **ВАЖНО**: не рекомендуется редактировать эти файлы напрямую. Вместо этого следует хранить конфигурацию в отдельных файлах в каталогах:
  - `/etc/clickhouse-server/config.d/`
  - `/etc/clickhouse-server/users.d/`

### Какой файл выбрать?
- Если настройка существует в таблице `system.settings`, её следует поместить в каталог `users.d`.
- Иначе – попробовать использовать каталог `config.d`.

### Настройки (примеры)
- `max_memory_usage`
- `max_execution_time`
- `force_primary_key`
- Источники документации:
  - [https://clickhouse.com/docs/en/operations/settings/settings](https://clickhouse.com/docs/en/operations/settings/settings)
  - [https://github.com/ClickHouse/ClickHouse/blob/master/src/Core/Settings.h](https://github.com/ClickHouse/ClickHouse/blob/master/src/Core/Settings.h)
  - [https://github.com/ClickHouse/ClickHouse/blob/master/docs/en/operations/server-configuration-parameters/settings.md?plain=1](https://github.com/ClickHouse/ClickHouse/blob/master/docs/en/operations/server-configuration-parameters/settings.md?plain=1)

### Рекомендуемые настройки для оптимизации
1. Параметры памяти (например, `max_memory_usage`).
2. Количество потоков для операций SELECT.
3. Размер буфера для минимизации операций ввода-вывода.
4. Настройки слияния (merge settings).
5. Параметры сжатия.
6. Параметры репликации.
7. Мониторинг производительности.
8. Увеличение количества открытых файловых дескрипторов (`ulimit -n` не менее 100000).
9. Увеличение объема разделяемой памяти (`kernel.shmmax` не менее 1 ГБ).
10. Отключение прозрачных огромных страниц (`transparent_hugepage=never`).
11. Изменение планировщика с `cfg` на `noop` или `deadline`.
12. Оптимизация сети (увеличение сетевых очередей, включение автонастройки TCP, отключение опций разгрузки).
13. Контроль загрузки системы (например, через `top`, `htop`).
14. Мониторинг производительности ClickHouse через встроенные средства.

### Запуск сервера
- Запуск в качестве демона:
  ```
  $ sudo clickhouse start
  ```
- Другие способы:
  ```
  $ sudo service clickhouse-server start
  $ sudo /etc/init.d/clickhouse-server start
  $ sudo systemctl start clickhouse-server.service
  $ clickhouse-server --config-file=/etc/clickhouse-server/config.xml
  ```
- Логи находятся в `/var/log/clickhouse-server/`.

### Тестовые датасеты
- Примеры датасетов (Example Datasets).
- Расширенное руководство (Advanced Tutorial).

## Интерфейсы и инструменты

### CLI (Command-Line Client) – `clickhouse-client`
- Работа в интерактивном и неинтерактивном (пакетном) режимах.
- Подключение через TCP: хост и порт (обычно 9440 с TLS или 9000 без TLS), имя базы данных, имя пользователя и пароль.
- Варианты вставки данных через `\copy`, `cat`, `echo`.
- **Заметки**:
  - В пакетном режиме по умолчанию используется `TabSeparated`.
  - Параметр `--multiquery` для выполнения нескольких запросов (кроме INSERT).
  - `\G` для вертикального формата.
  - История команд хранится в `~/.clickhouse-client-history`.
  - Для выхода: `exit`, `quit`, `logout`, `q`, `Q`, `:q`.
  - Для отмены запроса: `Ctrl+C` (дважды для завершения клиента).
- Конфигурация через файлы (поиск в порядке приоритета):
  1. Указанный в `--config-file`.
  2. `./clickhouse-client.xml`, `.yaml`, `.yml`.
  3. `~/.clickhouse-client/config.xml`, `.yaml`, `.yml`.
  4. `/etc/clickhouse-client/config.xml`, `.yaml`, `.yml`.
- Подключение через connection string:
  ```
  clickhouse://[user[:password]@][hosts_and_ports][/database][?query_parameters]
  ```

### HTTP Interface
- Предоставляет REST API для работы с ClickHouse с любой платформы и языка программирования.
- Порт HTTP: 8123, HTTPS: 8443.
- Web UI: `http://localhost:8123/play` (`/dashboard`).
- Примеры запросов:
  - Проверка доступности: `curl 'http://localhost:8123/ping'`.
  - Состояние реплик: `curl 'http://localhost:8123/replicas_status'`.
  - Для запросов, изменяющих данные, используется POST; GET — только для чтения.
- Сжатие данных:
  - Параметры `compress=1` (сервер сжимает ответ) и `decompress=1` (сервер распаковывает входящие данные).
  - Поддержка HTTP-сжатия (gzip).
- Аутентификация через URL, параметры или заголовки `X-ClickHouse-User` и `X-ClickHouse-Key`.
- Пример использования параметров в запросах.

### JDBC/ODBC Drivers
- **ClickHouse Java Libraries**: асинхронная, легкая библиотека для Java. Драйверы JDBC и R2DBC построены поверх неё.
- **ODBC Driver for ClickHouse**: официальная реализация для доступа к ClickHouse как к источнику данных.
- **ClickHouse JDBC Bridge**: позволяет ClickHouse получать доступ к данным из любого внешнего источника через JDBC-драйвер.
- **ClickHouse ODBC**: позволяет подключать приложения (Microsoft Excel, Tableau Desktop и др.) к ClickHouse.

### Database Interfaces
- **PostgreSQL Interface**:
  - Использует протокол PostgreSQL wire.
  - Позволяет ClickHouse эмулировать экземпляр PostgreSQL (порт 9005 по умолчанию).
  - Подключение через `psql`.
- **MySQL Interface**:
  - Использует протокол MySQL wire.
  - Порт 9004 по умолчанию.
  - Подключение через `mysql`.
  - В конфигурации необходимо указывать пароль пользователя в двойном SHA1.

### Автоматическое определение схемы (Schema Inference)
- ClickHouse может автоматически определять структуру входных данных почти во всех поддерживаемых форматах.
- Используется, когда структура данных неизвестна.
- Пример с JSONLines-файлом.
- Кэширование выведенной схемы для предотвращения повторного вывода.
- Работа с текстовыми форматами (JSONEachRow, CSV, Values).

### Сторонние интерфейсы
- Клиентские библиотеки.
- Интеграции.
- GUI.
- Прокси.
- **C++ Client Library**.
- **Native Interface (TCP)** – используется в CLI и для межсерверного взаимодействия.
- **gRPC Interface** – высокопроизводительный фреймворк для удаленного вызова процедур.

## Домашнее задание
1. Установить ClickHouse.
2. Подгрузить пример датасета и выполнить SELECT из таблицы.
3. Прислать скриншоты работающего инстанса ClickHouse, созданной виртуальной машины и результата запроса SELECT.
4. Провести тестирование производительности (например, через `clickhouse-benchmark`) и сохранить результаты.
5. Изучить конфигурационные файлы БД.
6. Выполнить оптимальную настройку системы на основе характеристик ОС и провести повторное тестирование.
7. Подготовить отчёт в формате PDF с описанием всех выполненных пунктов, включая анализ изменения производительности.
   - Поощряется работа в отдельных git-репозиториях.

## Рекомендации по Self-Managed ClickHouse
- ClickHouse использует все доступные аппаратные ресурсы.
- Эффективнее работает с большим количеством ядер с низкой тактовой частотой.
- Для нетривиальных запросов рекомендуется не менее 4 ГБ ОЗУ.
- Объем необходимой памяти зависит от сложности запросов и объема обрабатываемых данных.
- Для уменьшения потребления памяти возможен обмен временными данными на внешнюю память.
- В производственных средах рекомендуется отключать файл подкачки (swap).
- Расчет объема хранилища: учесть коэффициент сжатия и количество реплик.
- При кластеризации рекомендуется использовать сеть не ниже 10G.
- Пропускная способность сети критична для распределенных запросов и репликации.

## Список материалов для изучения
1. Установка и использование ClickHouse на Linux.
2. How to manage server config files in Clickhouse.
3. Available Installation Options.
4. Clickhouse init tutorial.
5. How to Install ClickHouse on Ubuntu 22.04 LTS Linux.
6. Installation and Management of clickhouse-operator for Kubernetes.

---
