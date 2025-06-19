# Пояснение разницы скорости выполнения первого и последующих запросов к БД PostgreSQL

### Каждый, кто работает с PostgreSQL рано или поздно задается вопросом, почему повторный запрос работает быстрее:

**Потому что срабатывает кэширование.**  
И это не одно кэширование, а сразу несколько уровней:

----------

### 1. Кэширование плана выполнения (**Query Plan Cache**)

Первый раз, когда отправляется SQL-запрос, PostgreSQL **анализирует** его:
    
-   Понимает структуру.      
-   Строит оптимальный **план выполнения** (EXPLAIN).
-   Выбирает, какие индексы использовать, как делать JOIN-ы и т.д.
        
Этот план сохраняется в памяти для текущего соединения (иногда глобально, зависит от настроек).
При **повторном запросе** сервер использует уже готовый план, **не пересчитывая его с нуля**. Это экономит много времени.

----------

### 2. Кэширование данных в памяти (**PostgreSQL Buffer Cache**)

Когда таблица, индекс или часть данных читается первый раз, PostgreSQL **загружает страницы** (страницы по 8Кб) **в оперативную память** (shared_buffers).
При последующих выполнениях запроса, если нужные данные уже есть в памяти (в shared_buffers), то **с диска** их читать **НЕ нужно**, данные берутся **мгновенно** из RAM.
Чтение из RAM многократно быстрее, чем с диска.

----------

### 3. Кэширование ОС (**Filesystem Cache**)

Даже если PostgreSQL сама не все выгрузит из shared_buffers (например, в очень загруженной системе), операционная система **держит файлы в кэше** на уровне файловой системы. В таком случае файловая система" при повторном доступе становится почти как RAM.

----------

 ### В качестве обобщения о причинах ускорения повторных запросов
- Кэш плана запроса: повторно использовать уже готовый план выполнения.
- Кэш данных в PostgreSQL: данные считываются не с диска, а из памяти (shared_buffers).
- Кэш данных в операционной системе: файлы/страницы остаются в RAM, быстрее доступ.

### Проверка утверждений

Это можно проверить на только что установленной базе с чистым кэшем, желательно, чтобы данных было достаточно, чтобы оценить разницу скорости выполнения:

```SQL
-- Первый запуск (будет медленно)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM большая_таблица WHERE условие;
``` 
```SQL
-- Сразу повторный запуск (будет заметно быстрее)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM большая_таблица WHERE условие;
``` 
**Первый `Execution Time`** будет больше,  **второй** — сильно меньше,  
**и в BUFFERS** будет видно, что чтение идет из `shared hit`, а не `disk read`.

### Что делать, если нет не кэшированной БД?

PostgreSQL есть разные уровни кэша - и очищать их нужно по-разному:

- План выполнения запросов: `DISCARD PLANS`.
- Сессионный кэш (подключение): `DISCARD ALL`.
- Кэш ОС (filesystem cache): специальной командой в Linux (ручное сбрасывание).
- Буферный кэш базы (shared_buffers): нельзя очистить напрямую — только через перезапуск сервера.

Подробнее, с примерами:
1. Очистить только **планы выполнения** (`DISCARD`) - Удаляет все кэшированные планы запроса в текущем соединении.
```SQL
DISCARD PLANS;
```
2. Очистить **буферный кэш** PostgreSQL (shared_buffers) - перезапуск сервера (PostgreSQL **сам** управляет shared_buffers, и **вручную очистить нельзя**).
```bash
sudo systemctl restart postgresql
```
или
```bash
pg_ctl restart
```
3. Очистить **кэш операционной системы (Linux)** (нужны привелегии)

```bash
# Сбросить только pagecache:
sudo sync
echo 1 | sudo tee /proc/sys/vm/drop_caches

# Или полностью (pagecache + dentries + inodes):
sudo sync
echo 3 | sudo tee /proc/sys/vm/drop_caches
```
Для полной очистки в одной команде:
```bash
sudo systemctl restart postgresql
sudo sync
echo 3 | sudo tee /proc/sys/vm/drop_caches
```

### # Как "прогреть" базу в PostgreSQL?
**Первый способ.** С помощью расширения:

PostgreSQL имеет специальное расширение для прогрева кэша: **`pg_prewarm`**.
**Шаги:**

1.  Установить расширение, если оно не установлено ранее (по умолчанию его нет):
```SQL
CREATE EXTENSION IF NOT EXISTS pg_prewarm;
```
2. Прогреть таблицу (это загрузит данные таблицы в **shared_buffers**.):
```SQL
SELECT pg_prewarm('название_таблицы');
```
3. Прогреть индекс:
```SQL
SELECT pg_prewarm('название_индекса');
```
**Второй способ.** Если нет возможности установить расширение, можно просто принудительно **прочитать** все строки:

```SQL
SELECT COUNT(*) FROM твоя_таблица;
```
или
```SQL
SELECT * FROM твоя_таблица;
```
 `COUNT(*)`предпочтительнее, не будет использоваться трафик для передачи большого ответа.
 
 **В результате:**

-   PostgreSQL прогрузит все страницы таблицы в shared_buffers. 
-   После этого реальные тестовые запросы будут выполняться "из памяти".

**Третий способ.**  Автоматический прогрев при старте PostgreSQL
Для того, чтобы **при перезапуске базы** данные прогревались сами можно использовать `pg_prewarm` в связке с настройками.

В `postgresql.conf` включить `pg_prewarm.autoprewarm`:
```conf
shared_preload_libraries = 'pg_prewarm'
```
Тогда база будет **сохранять список прогретых страниц** и автоматически загружать их в кэш при старте сервера.

### Краткое сравнение способов. Как и когда их применять?

- `pg_prewarm` вручную: при единичном тестировании или периодическом прогреве.
- Чтение всех строк (`COUNT(*)`): при простых тестах без расширений.
- Автопрогрев (`pg_prewarm + autoprewarm`): для продакшн-систем с контролем прогрева при рестарте.

### Проверка прогрета таблица или нет:
```SQL
CREATE EXTENSION IF NOT EXISTS pg_buffercache;

SELECT count(*) AS buffers_loaded
FROM pg_buffercache
WHERE relfilenode = (SELECT relfilenode FROM pg_class WHERE relname = 'имя_таблицы');
```
Нужно учесть тот момент, что в таблице  `pg_class`  в поле  `relfilenode`  значение 0 указывает на то, что это виртуальное или нефизическое отношение, которое не имеет собственного файла на диске. Обычно это относится к представлениям, индексам, секционированным таблицам или другим объектам, которые не хранят данные напрямую в файлах, а скорее зависят от других физических отношений. Так что для секционированной (партицированной) таблицы такой способ проверки не применим.

### Дополнительно
0. Включаем pg_prewarm:
```SQL
CREATE EXTENSION IF NOT EXISTS pg_prewarm;
```

1.  Скрипт для прогрева всех таблиц схемы (этот код **НЕ прогревает индексы**):


```sql
DO $$
DECLARE
    rec record;
BEGIN
    -- Для каждой таблицы в указанной схеме
    FOR rec IN
        SELECT schemaname, tablename
        FROM pg_tables
        WHERE schemaname = 'public' -- поменяй на свою схему если нужно
    LOOP
        -- Прогреваем таблицу через pg_prewarm
        RAISE NOTICE 'Prewarming table: %.%', rec.schemaname, rec.tablename;
        PERFORM pg_prewarm(quote_ident(rec.schemaname) || '.' || quote_ident(rec.tablename));
    END LOOP;
END
$$ LANGUAGE plpgsql;

```

Результат будет выглядеть примерно так:
```
NOTICE: Prewarming table: public.user NOTICE: Prewarming table: public.user_link 
NOTICE: Prewarming table: public.link NOTICE: Prewarming table: public.task 
NOTICE: Prewarming table: public.job_queue NOTICE: Prewarming table: 
public.goose_db_version NOTICE: Prewarming table: public.prepared_report ERROR: 
fork "main" does not exist for this relation CONTEXT: SQL statement "SELECT 
pg_prewarm(quote_ident(rec.schemaname) || '.' || quote_ident(rec.tablename))" 
PL/pgSQL function inline_code_block line 13 at PERFORM SQL state: 22023
```

2. Расширенный вариант с индексами (если нужно прогревать **и таблицы, и их индексы**):
```SQL
DO $$
DECLARE
    rec record;
BEGIN
    -- Прогреваем все таблицы
    FOR rec IN
        SELECT schemaname, tablename
        FROM pg_tables
        WHERE schemaname = 'public'
    LOOP
        RAISE NOTICE 'Prewarming table: %.%', rec.schemaname, rec.tablename;
        PERFORM pg_prewarm(quote_ident(rec.schemaname) || '.' || quote_ident(rec.tablename));
    END LOOP;

    -- Прогреваем все индексы
    FOR rec IN
        SELECT schemaname, indexname
        FROM pg_indexes
        WHERE schemaname = 'public'
    LOOP
        RAISE NOTICE 'Prewarming index: %.%', rec.schemaname, rec.indexname;
        PERFORM pg_prewarm(quote_ident(rec.schemaname) || '.' || quote_ident(rec.indexname));
    END LOOP;
END
$$ LANGUAGE plpgsql;

```

4. Хранимая процедура prewarm_schema, чтобы можно было прогревать любую схему одной командой:
```SQL
CREATE OR REPLACE PROCEDURE prewarm_schema(schema_name text)
LANGUAGE plpgsql
AS $$
DECLARE
    rec record;
BEGIN
    -- Прогрев всех таблиц в указанной схеме
    FOR rec IN
        SELECT schemaname, tablename
        FROM pg_tables
        WHERE schemaname = schema_name
    LOOP
        RAISE NOTICE 'Prewarming table: %.%', rec.schemaname, rec.tablename;
        PERFORM pg_prewarm(quote_ident(rec.schemaname) || '.' || quote_ident(rec.tablename));
    END LOOP;

    -- Прогрев всех индексов в указанной схеме
    FOR rec IN
        SELECT schemaname, indexname
        FROM pg_indexes
        WHERE schemaname = schema_name
    LOOP
        RAISE NOTICE 'Prewarming index: %.%', rec.schemaname, rec.indexname;
        PERFORM pg_prewarm(quote_ident(rec.schemaname) || '.' || quote_ident(rec.indexname));
    END LOOP;
END;
$$;

````

Вызов процедуры:
```SQL
CALL prewarm_schema('public'); -- или любое другое имя схемы
```

5. Хранимая процедура только для таблиц (без индексов):
```SQL
CREATE OR REPLACE PROCEDURE prewarm_only_tables(schema_name text)
LANGUAGE plpgsql
AS $$
DECLARE
    rec record;
BEGIN
    FOR rec IN
        SELECT schemaname, tablename
        FROM pg_tables
        WHERE schemaname = schema_name
    LOOP
        RAISE NOTICE 'Prewarming table: %.%', rec.schemaname, rec.tablename;
        PERFORM pg_prewarm(quote_ident(rec.schemaname) || '.' || quote_ident(rec.tablename));
    END LOOP;
END;
$$;

```
