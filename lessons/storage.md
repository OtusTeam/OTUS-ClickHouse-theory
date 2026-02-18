## Часть 1: Резервное копирование в ClickHouse

### Зачем нужно резервное копирование?

- Защита от катастрофических сбоев, приводящих к полной потере данных.
- Восстановление после случайного удаления базы данных или таблицы.
- Отладка проблем с использованием копии производственных данных.
- Тестирование обновлений схемы или версии ClickHouse перед применением.
- Загрузка схемы и конфигурации для развертывания новых инсталляций.

### Что нужно спасать?

В документе явно не перечислено, но подразумевается: данные, схема, конфигурации и, возможно, RBAC.

### Варианты бекапирования

В документе представлена таблица сравнения инструментов:

| Tool | Description | Configs | Schema | Data | RBAC | Replication |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: |
| **Use ReplicatedMergeTree** | Использование репликации для отказоустойчивости | ☐ | ☐ | ☑ | ☐ | ☑ |
| **ClickHouse Copier** | Работает с ZooKeeper для копирования данных кластера | ☐ | ☐ | ☑ | ☐ | ☐ |
| **Altinity clickhouse-backup** | Отдельная утилита для всех версий ClickHouse | ☑ | ☑ | ☑ | ☑ | ☐ |
| **ClickHouse BACKUP & RESTORE** | Встроенные SQL-команды в новых версиях ClickHouse | ☐ | ☑ | ☑ | ☐ | ☐ |

### Утилита `clickhouse-backup`

Это инструмент для простого резервного копирования и восстановления ClickHouse с поддержкой многих типов хранилищ.

**Основные возможности:**
- Простое создание и восстановление бэкапов всех или определенных таблиц.
- Эффективное хранение нескольких резервных копий в файловой системе.
- Загрузка и выгрузка с потоковым сжатием.
- Поддержка облачных хранилищ: AWS, GCS, Azure, Tencent COS, а также FTP, SFTP.
- Поддержка `Atomic` Database Engine.
- Поддержка многодисковых инсталляций.
- Поддержка пользовательских типов удаленных хранилищ.
- Поддержка инкрементного резервного копирования на удаленное хранилище.

#### Установка

```bash
# Скачать последний релиз с GitHub
wget https://github.com/Altinity/clickhouse-backup/releases/download/v2.6.40/clickhouse-backup-linux-amd64.tar.gz
tar -xf clickhouse-backup-linux-amd64.tar.gz

# Установить
sudo install -o root -g root -m 0755 build/linux/amd64/clickhouse-backup /usr/local/bin

# Проверить версию
/usr/local/bin/clickhouse-backup -v
```

#### Подготовка конфигурации

```bash
# Создать директорию для конфигов
sudo -u clickhouse mkdir /etc/clickhouse-backup

# Сгенерировать конфиг по умолчанию
sudo -u clickhouse clickhouse-backup default-config > /etc/clickhouse-backup/config.yml

# Отредактировать конфиг (прописываются секции general, clickhouse, s3 и т.д.)
sudo vi /etc/clickhouse-backup/config.yml

# Выставить правильного владельца
sudo chown -R clickhouse:clickhouse /etc/clickhouse-backup/
```

#### Основные команды

**Создание бэкапа:**
Процесс: `clickhouse-backup create` → данные сохраняются локально в `/var/lib/clickhouse/backup`.
```bash
clickhouse-backup create mybackup
```

**Загрузка бэкапа в удаленное хранилище:**
```bash
clickhouse-backup upload mybackup
```

**Восстановление из бэкапа:**
Процесс: `clickhouse-backup download` (если бэкап удаленный) → `clickhouse-backup restore`.
```bash
# Скачать бэкап из удаленного хранилища
clickhouse-backup download mybackup

# Восстановить данные из локального бэкапа
clickhouse-backup restore mybackup
```

#### Примеры команд

**Создание:**
```bash
# Полный локальный бэкап всего (включая RBAC и конфиги)
sudo -u clickhouse clickhouse-backup create mybackup --rbac --configs

# Локальный бэкап одной таблицы
sudo -u clickhouse clickhouse-backup create mybackup_table_local -t default.ex2

# Создание и загрузка удаленного бэкапа для всех таблиц БД default
sudo -u clickhouse clickhouse-backup create_remote mybackup_database_remote -t 'default.*'
```

**Восстановление:**
```bash
# Восстановить все данные из уже загруженного бэкапа
sudo -u clickhouse clickhouse-backup restore mybackup

# Восстановить одну таблицу из локального бэкапа
sudo -u clickhouse clickhouse-backup restore mybackup -t default.ex2

# Скачать и восстановить одну базу данных из удаленного бэкапа
sudo -u clickhouse clickhouse-backup restore_remote mybackup -t 'default.*'
```

**Управление:**
```bash
# Просмотр списка бэкапов
sudo -u clickhouse clickhouse-backup list
sudo -u clickhouse clickhouse-backup list local
sudo -u clickhouse clickhouse-backup list remote

# Удаление бэкапов
sudo -u clickhouse clickhouse-backup delete local mybackup
sudo -u clickhouse clickhouse-backup delete remote mybackup
```

### Встроенные команды BACKUP / RESTORE

Встроенные SQL-команды, доступные в последних версиях ClickHouse.

**Создание бэкапов:**
```sql
-- Полный бэкап базы данных
BACKUP DATABASE my_database TO 'file://backups/my_database_backup';

-- Инкрементный бэкап (только изменения с последнего полного)
BACKUP DATABASE my_database TO 'file://backups/my_database_backup_incremental' WITH increment;

-- Дифференциальный бэкап (изменения с последнего любого бэкапа)
BACKUP DATABASE my_database TO 'file://backups/my_database_backup_differential' WITH differential;
```

**Постановка на расписание (cron):**
```bash
# Cron job для ежедневного полного бэкапа в 2 часа ночи
0 2 * * * clickhouse-client --query="BACKUP DATABASE my_database TO 'file:///backups/my_database_backup'"
```

**Восстановление из бэкапов:**
```sql
-- Восстановление из полного бэкапа
RESTORE DATABASE my_database FROM 'file:///backups/my_database_backup';

-- Восстановление из инкрементного бэкапа
RESTORE DATABASE my_database FROM 'file:///backups/my_database_backup_incremental';

-- Восстановление из дифференциального бэкапа
RESTORE DATABASE my_database FROM 'file:///backups/my_database_backup_differential';
```

---

## Часть 2: Storage Policies в ClickHouse

### Что такое Storage Policy?

Это набор правил хранения и управления данными, который:
- Определяет, **как** и **где** хранятся данные.
- Контролирует места и методы хранения.
- Позволяет оптимизировать производительность запросов и затраты на хранение.

### Компоненты Storage Policy

1.  **Диски (Disks):** Физические или логические единицы хранения (локальные SSD/HDD, сетевые хранилища, S3).
2.  **Тома (Volumes):** Логическая группировка одного или нескольких дисков. Данные могут распределяться по дискам внутри тома.
3.  **Конфигурация хранилища (Storage Configuration):** Настройка в `config.xml`, которая связывает диски и тома в именованные политики.

### Типы дисков и их характеристики

| Тип диска | Скорость | Масштабируемость | Стоимость | Плюсы | Минусы |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Локальные** | Высокий | Низкий | Высокий | Низкая задержка, высокая производительность | Ограниченная емкость, высокая цена за ГБ |
| **Сетевые** | Средний | Средний | Средний | Общие ресурсы, умеренная масштабируемость | Задержки в сети, возможные перегрузки |
| **Облачные (S3)** | Низкий | Высокий | Низкий | Практически неограниченный объем, дешевизна | Высокая задержка, зависимость от сети |

### Пример конфигурации Storage Policy

Этот пример настраивает политику `s3`, которая использует один том `main`, состоящий из диска типа `s3`.

```xml
<clickhouse>
    <storage_configuration>
        <disks>
            <s3>
                <type>s3</type>
                <endpoint>https://s3.eu-west-1.amazonaws.com/clickhouse-eu-west-1.clickhouse.com/data/</endpoint>
                <use_environment_credentials>1</use_environment_credentials>
            </s3>
        </disks>
        <policies>
            <s3>
                <volumes>
                    <main>
                        <disk>s3</disk>
                    </main>
                </volumes>
            </s3>
        </policies>
    </storage_configuration>
</clickhouse>
```

### Преимущества использования Storage Policy

- **Эффективность:** Ускорение поиска данных и выполнения запросов за счет размещения "горячих" данных на быстрых носителях.
- **Оптимизация затрат:** Хранение "холодных" (исторических) данных на дешевых носителях вроде S3.
- **Масштабируемость:** Легкость расширения хранилища за счет подключения новых дисков.
- **Пример:** Финансовая компания хранит часто используемые торговые данные на быстрых локальных дисках, а исторические данные выгружает в S3, балансируя между производительностью и стоимостью.

### Полезные системные запросы

```sql
-- Информация о всех дисках, настроенных в системе
SELECT name, path,
       formatReadableSize(free_space) AS free,
       formatReadableSize(total_space) AS total,
       formatReadableSize(keep_free_space) AS reserved
FROM system.disks;

-- Информация о всех политиках хранения, томах и дисках в них
SELECT policy_name, volume_name, disks
FROM system.storage_policies;

-- Информация о том, на каком диске лежит конкретная часть (part) таблицы
SELECT name, disk_name, path
FROM system.parts;

-- Информация о том, какая политика хранения используется для таблицы
SELECT name, data_paths, metadata_path, storage_policy
FROM system.tables
WHERE name LIKE 'sample%';
```

---

## Материалы для изучения

(Ссылки из документа)
- Документация по системным таблицам: `clickhouse.com/docs/en/operations/system-tables/storage_policies`
- Блоги Altinity по multi-volume storage: Часть 1 и Часть 2.
- Репозиторий утилиты: `github.com/Altinity/clickhouse-backup`
- Презентация Altinity по использованию `clickhouse-backup`.
- Введение в бэкапы ClickHouse и `clickhouse-backup`.
