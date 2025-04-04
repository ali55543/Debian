Отчет по лабораторной работе №1 базовая настройка PostgreSQL на Debian
Исмаилов А.А. ИС-21
## 1. Подготовка среды

### 1.1 Установка виртуальной машины
Для начала я установил Debian 11 (можно использовать и Debian 12) на виртуальную машину. Для этого подойдут:
- VMware  
- VirtualBox  
- HyperV  
- WSL

### 1.2 Обновление системы
Сразу после установки я обновил систему:
```bash
sudo apt-get update && sudo apt-get upgrade -y
```
- `apt-get update` — обновляет список доступных пакетов.  
- `apt-get upgrade` — обновляет установленные пакеты.

---

## 2. Установка PostgreSQL

PostgreSQL я установил через `apt`:

```bash
sudo apt-get install postgresql postgresql-contrib -y
```

- `postgresql` — основной сервер СУБД.  
- `postgresql-contrib` — дополнительные модули.

---

## 3. Служебная учётная запись

После установки PostgreSQL автоматически создаётся системная учётка `postgres`.  
Именно через неё я выполнял все административные действия на сервере.

---

## 4. Первичная настройка конфигурации

### 4.1 Основные конфигурационные файлы:
- `postgresql.conf` — общие настройки PostgreSQL (порт, память и т.д.)  
- `pg_hba.conf` — управление доступом и аутентификацией.

### 4.2 Пример правок

Изменил порт при необходимости:

```ini
# /etc/postgresql/<версия>/main/postgresql.conf
port = 5432
```

Настроил метод аутентификации:

```ini
# /etc/postgresql/<версия>/main/pg_hba.conf
host    all    all    127.0.0.1/32    md5
```

После изменений обязательно перезапустил сервис:

```bash
sudo systemctl restart postgresql
```

---

## 5. Управление сервисом

Для работы с PostgreSQL использовал systemd:

- Проверить статус:
  ```bash
  sudo systemctl status postgresql
  ```
- Включить автозапуск:
  ```bash
  sudo systemctl enable postgresql
  ```

---

## 6. Создание тестовой базы данных

### 6.1 Создание пользователя и БД

1. Переключился на пользователя `postgres`:
   ```bash
   sudo -i -u postgres
   ```
2. Создал нового пользователя `alIsmailov` с паролем:
   ```sql
   CREATE ROLE alIsmailov WITH LOGIN PASSWORD 'your_password';
   ```
3. Создал базу данных `Ismailov_db` с владельцем `alIsmailov`:
   ```sql
   CREATE DATABASE Ismailov_db OWNER alIsmailov;
   ```

### 6.2 Подключение к БД

Через `psql` проверил подключение:
```bash
psql -U alIsmailov -d Ismailov_db -h 127.0.0.1 -W
```

---

## 7. Работа со схемами

### 7.1 Что такое схема
Схема — это логическая группа объектов внутри базы данных.  
По умолчанию используется `public`.

### 7.2 Создание новой схемы

Создал схему `test_schema`:
```sql
CREATE SCHEMA test_schema;
```

Выдал доступ пользователю:
```sql
GRANT USAGE ON SCHEMA test_schema TO alIsmailov;
```

Настроил `search_path`:
```sql
SET search_path TO test_schema, public;
```

---

## 8. Использование psql

### 8.1 Работа в `public`

Создал таблицу:
```sql
CREATE TABLE public.test_table (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);
```

Добавил записи:
```sql
INSERT INTO public.test_table (name) VALUES ('Test1'), ('Test2');
```

Выполнил базовые действия:
```sql
SELECT * FROM public.test_table;
UPDATE public.test_table SET name = 'Updated' WHERE id = 1;
DELETE FROM public.test_table WHERE id = 2;
```

### 8.2 Работа в `test_schema`

Создал таблицу:
```sql
CREATE TABLE test_schema.another_table (
    id SERIAL PRIMARY KEY,
    description TEXT
);
```

Вставил запись:
```sql
INSERT INTO test_schema.another_table (description) VALUES ('Example record');
```

Обращение к таблице:
```sql
SELECT * FROM test_schema.another_table;
-- или
SET search_path TO test_schema, public;
SELECT * FROM another_table;
```

### 8.3 Скрипт создания таблицы в схеме

```sql
-- Файл: create_table_test_schema.sql
CREATE TABLE test_schema.example (
    id SERIAL PRIMARY KEY,
    data VARCHAR(50)
);
```

Запустил скрипт:
```bash
psql -U alIsmailov -d Ismailov_db -f create_table_test_schema.sql
```

---

## 9. Настройка подключений

### 9.1 Разрешение внешних подключений

В `postgresql.conf`:
```ini
listen_addresses = '*'
```

### 9.2 Изменения в `pg_hba.conf`:

```ini
host    all    all    0.0.0.0/0    md5
```

Перезапустил PostgreSQL:
```bash
sudo systemctl restart postgresql
```

### 9.3 Подключение через GUI

Подключался через pgAdmin или DBeaver, указывая IP, порт (5432), имя пользователя и пароль.

---

## 10. Логирование

### 10.1 Настройка логов

В `postgresql.conf`:

```ini
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_statement = 'all'
```

Перезапустил сервис:
```bash
sudo systemctl restart postgresql
```

### 10.2 Просмотр логов

```bash
tail -n 20 /var/log/postgresql/postgresql-<date>.log
```

---

## 11. Роли и права

### 11.1 Создание ограниченной роли

Создал роль:
```sql
CREATE ROLE limited_user WITH LOGIN PASSWORD 'limited_password';
```

### 11.2 Назначение прав

Ограниченный доступ к таблице:
```sql
GRANT SELECT ON public.test_table TO limited_user;
```

### 11.3 Наследование прав

Создал общую роль и привязал к пользователю:
```sql
CREATE ROLE common_role;
GRANT SELECT, INSERT ON public.test_table TO common_role;
ALTER ROLE limited_user INHERIT common_role;
```

---

Если хочешь, я могу это всё экспортировать в PDF или Markdown-файл — только скажи!