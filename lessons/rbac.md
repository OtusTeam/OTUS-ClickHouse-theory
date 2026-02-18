## 1. Обзор сущностей RBAC

RBAC (Role-Based Access Control) в ClickHouse — это модель доступа, управляемая из SQL, которая включает 6 взаимосвязанных типов сущностей:

1.  **Пользователь (User)**
2.  **Право (Privilege)**
3.  **Роль (Role)**
4.  **Профиль (Settings Profile)**
5.  **Квота (Quota)**
6.  **Политика (Row-policy)**

### Краткое описание сущностей

*   **Пользователь (User):** Базовая учетная запись. Описывается:
    *   Именем.
    *   Способом авторизации.
    *   Белым списком сетей/DNS-имен. Остальные параметры задаются другими сущностями.
*   **Право (Privilege):** Разрешающее правило.
    *   Выдается на объект (БД, таблица, словарь или специальное правило, например, репликация).
    *   Выдается пользователю или роли.
    *   Типы прав часто совпадают с типами запросов (INSERT, SELECT, ALTER и т.д.).
*   **Роль (Role):** Именованный набор прав. Сама по себе не имеет ничего кроме имени. Права выдаются роли, а затем роль выдается пользователям.
*   **Профиль (Settings_profile):** Набор пар `настройка=значение`, назначаемый пользователям.
    *   Пользователь не может переопределить настройки, объявленные в назначенном профиле.
    *   Пример: профиль с настройкой `log_queries=1` запретит пользователю выполнить `log_queries=0`.
*   **Квота (Quota):** Ограничивает использование ресурсов на временном интервале.
    *   Работает как счетчик. При достижении порога ClickHouse возвращает исключение "квота исчерпана".
    *   По истечении интервала счетчик сбрасывается.
    *   Пример: "не более 60 запросов в минуту" или "не более N прочитанных байт в час".
*   **Политика (Row-policy):** Назначается на таблицу.
    *   Содержит список пользователей, к которым применяется, и простое выражение-фильтр (например, `column = значение` или `column IN (...)`).
    *   Строки, не соответствующие условию, скрываются из результата запроса и отсеиваются до стадии агрегации.

## 2. Пользователь (User)

### Управление пользователями
*   **Создание:** `CREATE USER name ...`
*   **Изменение:** `ALTER USER name ...` (имеет тот же синтаксис, что и CREATE)
*   **Удаление:** `DROP USER name`

### Синтаксис CREATE USER (основные блоки)
```sql
CREATE USER [IF NOT EXISTS | OR REPLACE] name1 [ON CLUSTER cluster_name1]
    [NOT IDENTIFIED | IDENTIFIED {{...}}]
    [HOST {...} | ANY | NONE]
    [VALID UNTIL datetime]
    [DEFAULT ROLE role [...]]
    [DEFAULT DATABASE database | NONE]
    [GRANTEES {...}]
    [SETTINGS ... | PROFILE 'profile_name'] [...]
```

### Разбор синтаксиса
*   **Где создавать:**
    *   `IF NOT EXISTS` — не выполнять запрос, если пользователь уже существует.
    *   `OR REPLACE` — пересоздать пользователя, если он уже есть.
    *   `ON CLUSTER cluster_name` — выполнить запрос на всех репликах/шардах кластера.
*   **Авторизация:**
    *   `NOT IDENTIFIED` или `IDENTIFIED WITH no_password` — без пароля.
    *   Парольная: `IDENTIFIED WITH plaintext_password`, `sha256_password`, `double_sha1_password BY 'password'`.
    *   Хеш пароля: `IDENTIFIED WITH sha256_hash`, `double_sha1_hash BY 'hash'`.
    *   Внешняя: `WITH ldap SERVER 'server_name'`, `WITH kerberos`, `WITH ssl_certificate`, `WITH ssh_key`.
*   **Белый список:** `HOST {LOCAL | NAME 'name' | REGEXP 'name_regexp' | IP 'address' | LIKE 'pattern'} [...] | ANY | NONE`.
*   **Срок действия:** `VALID UNTIL datetime`.
*   **Роль по умолчанию:** `DEFAULT ROLE role [...]`.
*   **БД по умолчанию:** `DEFAULT DATABASE database | NONE` (не мешает подключаться к другим).
*   **Профиль по умолчанию:** `SETTINGS ... | PROFILE 'profile_name'`.
    *   Если задается через `SETTINGS`, для пользователя создается новый профиль с указанными настройками.

### Просмотр пользователей
```sql
SELECT name, auth_type, auth_params
FROM system.users
WHERE storage = 'local_directory';
```

## 3. Право (Privilege)

### Управление правами
*   **Выдача прав:**
    ```sql
    GRANT [ON CLUSTER cluster_name]
        privilege[(column_name(s))] [...]
        ON {db.table | db.* | *.* | table | *}
        TO {user | role | CURRENT_USER} [...]
        [WITH GRANT OPTION] [WITH REPLACE OPTION]
    ```
*   **Отзыв прав:**
    ```sql
    REVOKE [ON CLUSTER cluster_name]
        privilege[(column_name(s))] [...]
        ON {db.table | db.* | *.* | table | *}
        FROM {user | CURRENT_USER} [...] | ALL | ALL EXCEPT {user | CURRENT_USER}
    ```

### Виды прав
*   Полный список для текущей версии можно получить из `system.grants`.
*   Наиболее популярные:
    *   Для работы приложений и пользователей: `SELECT`, `INSERT`, `SHOW`, `dictGet`.
    *   Для миграций: `ALTER`, `DROP`, `CREATE`.
*   Специальные операции: `SYSTEM DROP DNS CACHE` (обычно выполняются под администратором).
*   Специальные функции: `FILE`, `S3` (разрешают использование одноименных табличных функций).

### Объекты прав
*   База: `ON db.*`
*   Таблица: `ON db.table`
*   Словарь: `ON db.dict` (словари без указания БД выдать точечно нельзя).
*   Глобально: `ON *.*`

### Примеры выдачи прав
```sql
GRANT SELECT, SHOW ON public_data.* TO anonymous;
GRANT INSERT, SELECT, SHOW ON prod_data.* TO backend;
GRANT SELECT, SHOW ON stage_data.* TO developer;
GRANT dictGet ON dicts.* TO backend, developer;
```

### Просмотр текущих прав
```sql
SELECT * FROM system.grants
WHERE user_name IN (SELECT name FROM system.users WHERE storage = 'local_directory');
```

## 4. Роль (Role)

### Управление ролями
*   **Создать роль:** `CREATE ROLE name`
*   **Удалить роль:** `DROP ROLE name`
*   **Назначить роль пользователю:** `GRANT role TO user`
*   **Отозвать роль у пользователя:** `REVOKE role FROM user`

### Просмотр ролей
*   Список, кому выдана какая роль:
    ```sql
    SELECT * FROM system.role_grants;
    ```
*   Список прав, выданных на роль, находится в `system.grants` (столбец `role_name`).

## 5. Профиль (settings_profile)

### Управление профилями
*   **Создать:** `CREATE SETTINGS PROFILE [IF NOT EXISTS | OR REPLACE] name ...`
*   **Изменить:** `ALTER SETTINGS PROFILE ...` (синтаксис как у CREATE)
*   **Удалить:** `DROP SETTINGS PROFILE [IF EXISTS] name`
*   **Назначить пользователю/роли:** `ALTER USER/ROLE ... PROFILE 'profile_name'`

### Синтаксис CREATE SETTINGS PROFILE
```sql
CREATE SETTINGS PROFILE name
    [ON CLUSTER cluster_name]
    [SETTINGS variable [= value] [MIN [=] min_value] [MAX [=] max_value] [CONST|READONLY|WRITABLE|CHANGEABLE_IN_READONLY] | INHERIT 'profile_name'] [...]
```
*   `SETTINGS` — набор пар ключ=значение.
*   `INHERIT` — унаследовать настройки из другого профиля.

### Просмотр профилей
Текущие профили можно посмотреть в таблице `system.settings_profiles`.

## 6. Квота (Quota)

### Управление квотами
*   **Создание/Изменение:** `CREATE QUOTA ...` / `ALTER QUOTA ...` (синтаксис идентичен)
*   **Удаление:** `DROP QUOTA name`

### Синтаксис CREATE QUOTA
```sql
CREATE QUOTA [IF NOT EXISTS | OR REPLACE] name [ON CLUSTER cluster_name]
    [KEYED BY {user_name | ip_address | client_key | ...} | NOT KEYED]
    [FOR [RANDOMIZED] INTERVAL number {second | minute | hour | day | week | month | quarter | year}
        {MAX { {queries | query_selects | ... | read_rows | read_bytes | execution_time} = number } [...] }
        | NO LIMITS | TRACKING ONLY} [...]
    [TO {role [...]} | ALL | ALL EXCEPT role [...]}]
```

### Разбор синтаксиса квот
*   **Ключ группировки:** `KEYED BY ...` — отдельный счетчик на каждый IP-адрес или пользователя.
*   **Интервал:** Задается через `FOR INTERVAL ...`.
*   **Счетчики:** `MAX {queries = number, read_bytes = number, ...}`. Можно указать несколько через запятую.
*   **Режим:** `NO LIMITS` (не ограничивать, но вести счетчик) или `TRACKING ONLY` (только учет).
*   **Применение:** `TO ...` — пользователи или роли, к которым применяется квота.

### Просмотр квот и лимитов
*   Список лимитов: `SELECT * FROM system.quota_limits`
*   Список квот: `SELECT * FROM system.quotas`

## 7. Политика (row-policy)

### Синтаксис CREATE ROW POLICY
```sql
CREATE ROW POLICY [IF NOT EXISTS | OR REPLACE] policy_name1 [ON CLUSTER cluster_name1]
    ON [db1.]table1 | db1.*
    [AS {PERMISSIVE | RESTRICTIVE}]
    [FOR SELECT] USING condition
    [TO {role1 [, role2 ...] | ALL | ALL EXCEPT role1 [, role2 ...]}]
```

### Разбор синтаксиса row-policy
*   **Объект:** Создается на таблицу (`db.table`) или все таблицы БД (`db.*`).
*   **Условие:** `USING condition`. Поддерживаются простые выражения: `column = константа` или `column IN ('константа1','константа2')`.
*   **Режим:**
    *   `AS PERMISSIVE` — должна быть соблюдена любая из политик.
    *   `AS RESTRICTIVE` — должны быть соблюдены все политики.
*   **Применение:** `TO ...` — роли, к которым применяется политика.

### Просмотр политик
Текущие политики можно посмотреть в таблице `system.row_policies`.
