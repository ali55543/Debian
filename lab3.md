Лабораторная работа №3 
Исмаилов А.А. ИС-21 
## Расширенные возможности и оптимизация PostgreSQL на Debian


---

## 1. Оптимизация конфигурации PostgreSQL

Полезные параметры из `postgresql.conf`, влияющие на производительность:

- **shared_buffers** — объём памяти, выделенной под кэш PostgreSQL. Обычно ставят 25–40% от RAM.
- **work_mem** — сколько памяти может использоваться на сортировку и хеш-таблицы внутри одного запроса.
- **maintenance_work_mem** — выделяется на операции VACUUM, CREATE INDEX и т.п.
- **effective_cache_size** — оценка доступного файлового кэша. Влияет на выбор планов запросов.

### Пример значений (на виртуалке с 2 ГБ ОЗУ):

```conf
shared_buffers = 512MB
work_mem = 16MB
maintenance_work_mem = 128MB
effective_cache_size = 1GB
```

После изменения конфигурации перезапустил PostgreSQL:

```bash
sudo systemctl restart postgresql
```

Проверка параметров:

```sql
SHOW shared_buffers;
SHOW work_mem;
SHOW maintenance_work_mem;
SHOW effective_cache_size;
```

---

## 2. Индексы и планы выполнения

Добавил много данных в таблицу через `generate_series`:

```sql
INSERT INTO test_table (id, name)
SELECT g, 'Name ' || g FROM generate_series(1, 1000000) g;
```

Создал индекс:

```sql
CREATE INDEX idx_test_table_name ON test_table(name);
```

Сравнил планы до и после:

```sql
EXPLAIN ANALYZE SELECT * FROM test_table WHERE name = 'Name 500000';
```

До индекса: `Seq Scan`, медленно.  
После индекса: `Index Scan`, быстрее.

---

## 3. Хранимая функция

Создал простую функцию, которая проверяет значение и либо добавляет запись, либо выкидывает ошибку:

```sql
CREATE OR REPLACE FUNCTION insert_if_positive(val integer)
RETURNS TEXT AS $$
BEGIN
  IF val < 0 THEN
    RETURN 'Ошибка: отрицательное значение';
  ELSE
    INSERT INTO numbers_table(value) VALUES (val);
    RETURN 'Запись добавлена';
  END IF;
END;
$$ LANGUAGE plpgsql;
```

Проверил:

```sql
SELECT insert_if_positive(10); -- Запись добавлена
SELECT insert_if_positive(-5); -- Ошибка
```

---

## 4. Триггеры

Добавил триггер, который запрещает отрицательную цену:

```sql
CREATE OR REPLACE FUNCTION check_price()
RETURNS trigger AS $$
BEGIN
  IF NEW.price < 0 THEN
    RAISE EXCEPTION 'Цена не может быть отрицательной';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

Создал сам триггер:

```sql
CREATE TRIGGER trg_check_price
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION check_price();
```

Проверка:

```sql
INSERT INTO products(name, price) VALUES ('Товар', -100); -- Ошибка
```

---

## 5. VACUUM и ANALYZE

Проверил параметры:

```sql
SHOW autovacuum;
SHOW autovacuum_naptime;
SHOW autovacuum_vacuum_scale_factor;
```


```sql
VACUUM ANALYZE test_table;
```

### Зачем это нужно:
- **VACUUM** — удаляет "мёртвые" строки после UPDATE/DELETE.
- **ANALYZE** — обновляет статистику, чтобы планировщик запросов работал точнее.

### Проверка статистики:

```sql
SELECT relname, n_tup_ins, n_tup_upd, n_tup_del, n_autovacuum, n_live_tup
FROM pg_stat_user_tables;
```

Также можно посмотреть:

```sql
SELECT * FROM pg_stat_all_indexes WHERE relname = 'test_table';
