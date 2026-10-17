# Локальный Lakehouse — TPC-H Q3

**Автор:** Зубалов Владимир Константинович  
**Вариант:** 1 — Shipping Priority  
**Масштаб:** sf1


## Состав

- Silo — S3-совместимое хранилище.
- PostgreSQL — состояние каталога Lakekeeper.
- Lakekeeper — Iceberg REST Catalog.
- Trino — загрузка, преобразования и аналитика.
- Superset — дополнительный BI-инструмент в Docker.
- DBeaver — SQL-клиент Windows.

Поток обработки:

`tpch.sf1 → iceberg.bronze → iceberg.silver.order_lines → iceberg.gold.shipping_priority`

## Файлы проекта

- Основные Compose-файлы — развёртывание Lakehouse.
- `trino/` — конфигурация движка и каталогов.
- `sql/` — SQL загрузки, преобразований, проверок и экспериментов.
- `compose.bi.yaml` — развёртывание Superset.
- `bi/Dockerfile` — Superset с драйвером Trino.
- `bi/superset_config.py` — локальная конфигурация Superset.
- `docs/images/` — подтверждающие скриншоты.
- `отчет.md` — полный отчёт.


## Запуск уже инициализированного окружения

Из папки проекта:

```powershell
docker compose start
docker compose -f compose.bi.yaml up -d superset
```

Проверка:

```powershell
docker compose ps -a
docker compose -f compose.bi.yaml ps -a
docker compose exec trino trino --user student --execute "SELECT count(*) FROM iceberg.gold.shipping_priority"
```

Последняя команда должна вернуть 10.

Для временной остановки:

```powershell
docker compose -f compose.bi.yaml stop
docker compose stop
```

Тома с данными при остановке сохраняются.

## Развёртывание с нуля

1. Подготовить Docker Desktop с Linux-контейнерами и WSL2.
2. Подготовить конфигурации и локальные значения переменных окружения.
3. Поднять Silo, PostgreSQL и Lakekeeper; успешно выполнить миграцию каталога.
4. Создать бакет `lakehouse`.
5. Выполнить первоначальную настройку Lakekeeper и зарегистрировать warehouse `lakehouse` с endpoint `http://silo:9000`, регионом `us-east-1`, path-style access и префиксом `warehouse`.
6. Поднять Trino с каталогами `tpch` и `iceberg`.
7. Выполнить SQL создания Bronze, проверки Bronze, создания Silver, проверки Silver, создания Gold и сравнения с исходным Q3.
8. Собрать Superset, выполнить `superset db upgrade`, создать администратора, выполнить `superset init` и запустить веб-сервис.
9. Добавить подключение `trino://student@trino:8080/iceberg/bronze` и проверить Bronze в SQL Lab.

Созданные в существующем окружении warehouse и таблицы находятся в Docker volumes. Они не переносятся автоматически через Git. При свежем развёртывании инициализацию и SQL необходимо выполнить заново; ID warehouse и снапшотов будут другими.

## Адреса

| Сервис | Адрес |
|---|---|
| Trino | http://localhost:8080 |
| Silo Console | http://localhost:9001 |
| Lakekeeper | http://localhost:8181 |
| Superset | http://localhost:8088 |

## Основные результаты

| Формат | Кодек | Размер Bronze, МБ | Запись трёх таблиц, с | Медиана Q3, с |
|---|---|---:|---:|---:|
| Parquet | ZSTD | 48,656 | 5,197 | 1,090 |
| Parquet | GZIP | 48,816 | 8,470 | 1,170 |
| ORC | ZSTD | 54,390 | 6,095 | 0,961 |

Запись — по одному измерению для каждой таблицы. Q3 — медиана пяти повторений после одного прогрева. Использована метрика Execution из EXPLAIN ANALYZE.
