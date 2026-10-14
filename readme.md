# Отчёт по практической №1

## 1. Вариант и масштаб

| Поле | Значение |
|---|---|
| ФИО (как в VK Education) | Манько Платон Олегович |
| Вариант (запрос) | Q16 |
| Масштаб `sfN` | sf10 |
| Почему выбрал такой масштаб | Чтобы была видна разница между кодеками. Можно было бы и больше, но зачем. |

## 2. Поднятые сервисы — и почему именно они

| Сервис | Роль в стеке | Почему именно он (а не альтернатива) |
|---|---|---|
| MinIO | S3-совместимое объектное хранилище (сюда физически падают parquet-файлы) | Совместим с S3, с ним работают и трино и спарк, поднимается одним контейнером. |
| PostgreSQL | БД, где Lakekeeper хранит метаданные каталога | Lakekeeper хранит метаданные в постгресе, данные сохраняются в томе. |
| Lakekeeper | Iceberg REST Catalog — каталог метаданных таблиц | Стандартный рест-каталог для айсберга, через него трино и спарк видят одни и те же таблицы. Поднимается проще чем Hive Metastore. |
| Trino | SQL-движок: ходит в Lakekeeper за метаданными и читает/пишет данные в MinIO | Быстрый, есть встроенный генератор tpch, умеет всё что нужно по айсбергу. |
| (BI: Superset / Metabase) | отчёт/дашборд поверх Gold | не поднимал, в 6.2 взял спарк |

### Скрипты, которыми поднимал стек

Сначала поднял минио и создал бакет bench, потом постгрес, затем lakekeeper и в конце трино. Порты 8080 и 8181 на винде заняты системой, поэтому трино на 8280, lakekeeper на 8281, консоль минио на 9002.

```bash
docker run -d --name lakehouse-minio -p 9000:9000 -p 9002:9001 -e MINIO_ROOT_USER=minioadmin -e MINIO_ROOT_PASSWORD=minioadmin -v minio-data:/data quay.io/minio/minio:latest server /data --console-address ":9001"
docker run --rm --entrypoint /bin/sh pgsty/mc:latest -c "mc alias set m http://host.docker.internal:9000 minioadmin minioadmin && mc mb --ignore-existing m/bench"
docker run -d --name lakehouse-pg -e POSTGRES_PASSWORD=postgres -p 5432:5432 -v pg-data:/var/lib/postgresql/data postgres:17
docker run --rm -e 'LAKEKEEPER__PG_ENCRYPTION_KEY=This-is-NOT-Secure!' -e 'LAKEKEEPER__PG_DATABASE_URL_READ=postgresql://postgres:postgres@host.docker.internal:5432/postgres' -e 'LAKEKEEPER__PG_DATABASE_URL_WRITE=postgresql://postgres:postgres@host.docker.internal:5432/postgres' quay.io/lakekeeper/catalog:latest-main migrate
docker run -d --name lakekeeper -e 'LAKEKEEPER__PG_ENCRYPTION_KEY=This-is-NOT-Secure!' -e 'LAKEKEEPER__PG_DATABASE_URL_READ=postgresql://postgres:postgres@host.docker.internal:5432/postgres' -e 'LAKEKEEPER__PG_DATABASE_URL_WRITE=postgresql://postgres:postgres@host.docker.internal:5432/postgres' -p 8281:8181 quay.io/lakekeeper/catalog:latest-main serve
curl.exe -X POST http://localhost:8281/management/v1/bootstrap -H "Content-Type: application/json" -d '{\"accept-terms-of-use\": true}'
curl.exe -X POST http://localhost:8281/management/v1/warehouse -H "Content-Type: application/json" -d "@create-warehouse-bench.json"
docker run -d --name trino -p 8280:8080 -v "${PWD}\catalog:/etc/trino/catalog" trinodb/trino:476
```

## 3. Разбор запроса — какие данные нужны

| Таблица TPC-H | Нужные колонки | Что запрос с ними делает (фильтр / джойн / агрегат / вывод) |
|---|---|---|
| part | partkey, brand, type, size | partkey для джойна с partsupp, по brand, type и size фильтр, группировка и вывод |
| partsupp | partkey, suppkey | partkey для джойна, suppkey в NOT IN и count(distinct) |
| supplier | suppkey, comment | подзапрос, по comment находятся поставщики с жалобами |

## 4. Медальон — решения по каждому слою

### Bronze (сырой)

| Таблица | Колонки (что оставил) | Строк | Вес, МБ | Формат | Сжатие | Партиционирование | Обоснование |
|---|---|---|---|---|---|---|---|
| bronze.part | все | 2 000 000 | 40,8 | PARQUET | ZSTD | нет | В бронзовый слой залил данные как есть, чтобы из них можно было пересобрать серебряный. ZSTD выбрал по тестам из п. 5. Партиции на таком объёме дали бы только мелкие файлы. |
| bronze.partsupp | все | 8 000 000 | 276,7 | PARQUET | ZSTD | нет | так же |
| bronze.supplier | все | 100 000 | 5,0 | PARQUET | ZSTD | нет | так же |

### Silver (очищенный)

| Таблица | Колонки (что оставил) | Строк | Вес, МБ | Формат | Сжатие | Партиционирование | Обоснование |
|---|---|---|---|---|---|---|---|
| silver.supplier | suppkey, name, nationkey, has_complaints | 100 000 | 0,2 | PARQUET | ZSTD | нет | Проверил жалобы один раз и сохранил флагом, лишние текстовые колонки убрал. |
| silver.part_supply | partkey, suppkey, brand, type, size | 7 995 520 | 42,7 | PARQUET | ZSTD | нет, сортировка по size, brand | Соединил partsupp и part и убрал поставщиков с жалобами. По size партиции дали бы 50 мелких файлов, поэтому просто отсортировал. |

### Gold (витрина)

| Таблица | Колонки | Строк | Вес, МБ | Формат | Сжатие | Партиционирование | Обоснование |
|---|---|---|---|---|---|---|---|
| gold.q16_part_supplier_cnt | brand, type, size, supplier_cnt | 27 840 | 0,05 | PARQUET | ZSTD | нет | Готовый результат Q16, таблица маленькая, партиции не нужны. |

## 5. Тесты производительности

Сначала пересоздал бронзовый слой с GZIP, перезапустил трино и три раза прогнал Q16. Потом то же самое сделал с ZSTD. Первый запуск холодный, остальные тёплые, время брал из system.runtime.queries.

| Сжатие на Bronze | Время запроса | Размер Bronze, МБ | Твой вывод (что лучше и почему) |
|---|---|---|---|
| GZIP | 7,7 с холодный, 2,9 и 2,0 с тёплые | 292,6 | Сжимает сильнее, но запись идёт в три раза дольше (58,6 с), и холодный запрос медленнее. |
| ZSTD | 5,3 с холодный, 2,5 и 2,2 с тёплые | 322,5 | Больше GZIP всего на 10%, запись 19,1 с, холодный запрос быстрее. Оставил его. |

## 6. Часть 2 — осмысление и работа с инструментами

### 6.1. Понять и объяснить

| Вопрос | Твой ответ |
|---|---|
| Зачем нужны большие данные и lakehouse (а не обычная БД/хранилище)? | Большие объёмы дорого держать в обычной базе. В lakehouse данные лежат файлами в дешёвом S3, а айсберг даёт поверх них таблицы, транзакции и снапшоты. Одни и те же данные можно читать разными движками. |
| Пример schema evolution на Silver: добавил/переименовал колонку — что произошло? | Сначала добавил в silver.supplier колонку load_tag, у всех строк там был NULL. Потом заполнил её у поставщиков с жалобами. Затем переименовал её в quality_tag. Данные остались на месте, файлы не переписывались, потому что айсберг хранит колонки по ID, а не по имени. |
| Пример скрипта восстановления данных из снапшота (time travel) | Сначала запомнил текущий снапшот золотой таблицы. Потом удалил из неё строки с size = 9. Затем через FOR VERSION AS OF проверил, что старая версия читается целиком, и откатил таблицу: `ALTER TABLE bench.gold.q16_part_supplier_cnt EXECUTE rollback_to_snapshot(3953636623695050696)`. Строк снова 27 840. |

### 6.2. Подключить инструмент к lakehouse

| Инструмент | Почему выбрал именно его |
|---|---|
| Spark 3.5.3 | Читает те же таблицы айсберга через lakekeeper без копирования данных. Для моего запроса трино быстрее, но спарк лучше подходит для тяжёлой обработки, например для пересборки серебряного слоя из бронзового. |

Сначала прописал в spark-defaults.conf каталог bench с адресом lakekeeper и доступом к минио. Потом запустил спарк в докере и посмотрел таблицы бронзового слоя. Затем прогнал по ним Q16, результат совпал с трино.

```bash
docker run --rm -v "C:\Users\user\lakehouse\spark\conf:/opt/spark/conf" -v "C:\Users\user\lakehouse\spark\jars:/opt/spark/iceberg-jars" -v "C:\Users\user\lakehouseVKcourse\p1-q16\spark:/work" apache/spark:3.5.3 /opt/spark/bin/spark-sql -f /work/bronze_check.sql
```

```sql
SHOW TABLES IN bench.bronze;
SELECT count(*) FROM bench.bronze.partsupp;
```

## 7. Выводы

Бронзовый слой хранит сырые данные, в серебряном один раз делается очистка, в золотом лежит готовый результат. Больше всего места сэкономил на серебряном слое, когда убрал лишние колонки: было 317 МБ, стало 43. Из кодеков лучше ZSTD, по размеру почти как GZIP, а пишется в три раза быстрее. Партиции на таком объёме не нужны.
