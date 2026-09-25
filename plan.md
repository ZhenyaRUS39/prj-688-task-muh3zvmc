# Plan: Создай CLI для резервного копирования PostgreSQL

**Project:** `PRJ-8602`  
**Task ID:** `task-muh3zvmc`  
**Repo:** `prj-688-task-muh3zvmc`  
**Progress:** 1/5 subtasks done

## Summary

Декомпозировал задачу на 5 этапов: структура+CLI-каркас → pg_dump обёртка с конфигом → сжатие/ротация retention → планировщик cron + логирование → интеграционные тесты на тестовой БД. Каждый subtask проверяем отдельно.

## Subtasks

### ❌ 1. Инициализировать структуру репозитория и CLI-каркас

- **ID:** `sub-1`
- **Profile:** `20razrab1`
- **Status:** `failed`
- **Description:** Создать pyproject.toml (или package.json+tsconfig), директории src/, tests/, docs/, .gitignore, README.md с описанием, точку входа CLI (например src/cli.py или src/index.ts) с минимальным каркасом команды `backup` через argparse/commander/yargs + Click. Должна запускаться команда `pgbackup --help` и показывать help.
- **Test plan:** 1) `git clone` чистый, `pip install -e .` (или `pnpm i`) проходит. 2) `pgbackup --help` выводит список подкоманд с описаниями. 3) `pgbackup backup --help` существует (пока stub). 4) `pytest tests/` или `npm test` — пустой smoke-test зелёный.

### ⬜ 2. Реализовать обёртку pg_dump/pg_dumpall с конфигом

- **ID:** `sub-2`
- **Profile:** `30razrab2`
- **Status:** `pending`
- **Description:** Модуль backup.py: чтение конфига (TOML/YAML/env: host, port, user, password, dbname, schema, формат custom/plain/directory), вызов `pg_dump` через subprocess с безопасной передачей пароля через `.pgpass` или env `PGPASSWORD`. Поддержка --full (pg_dumpall --globals-only + pg_dump per-db) и --db<name>. Обработка ошибок с non-zero exit-кодом и понятным сообщением.
- **Test plan:** 1) Unit-тест: при неверном пароле CLI возвращает exit code != 0 и stderr содержит 'authentication failed'. 2) Интеграционно: поднять временный postgres (например через testcontainers-python или docker run) и сделать реальный backup, проверить что файл создан и не пустой. 3) Пароль НЕ светится в `ps aux` / process listing (использовать .pgpass или PGPASSWORD через env, не argv).
- **Dependencies:** `sub-1`

### ⬜ 3. Сжатие, шифрование (опц.) и ротация по retention

- **ID:** `sub-3`
- **Profile:** `20razrab1`
- **Status:** `pending`
- **Description:** Модуль retention.py: после успешного backup — gzip/zstd сжатие (если формат не custom/directory, которые уже сжаты), опциональное шифрование через age/gpg (через флаг --encrypt), вычисление sha256 чек-суммы в .sha256-файле. Ротация: удалять бэкапы старше N дней (по умолчанию 7), хранить минимум M последних (по умолчанию 5) — даже если старше N. Логика: `files.sort(mtime).slice(0, max(keep_last, keep_within_window))`.
- **Test plan:** 1) Юнит-тест retention: создать 10 файлов с разными mtime, вызвать apply_retention(keep_days=3, keep_last=5) → останется 5 самых свежих. 2) Интеграционно: сделать backup → проверить .sha256 совпадает с `sha256sum <file>`. 3) Для шифрования: backup --encrypt → файл нечитаем напрямую, расшифровка через --decrypt даёт исходный дамп.
- **Dependencies:** `sub-2`

### ⬜ 4. Планировщик cron + структурированное логирование

- **ID:** `sub-4`
- **Profile:** `30razrab2`
- **Status:** `pending`
- **Description:** Подкоманда `pgbackup schedule install/uninstall`: генерирует crontab-строку (например `0 3 * * * pgbackup backup --config /etc/pgbackup.toml`) и устанавливает её через `crontab -` от текущего пользователя (или в `/etc/cron.d/pgbackup` под root). Команда `pgbackup schedule show` показывает текущую установленную строку. Логирование через structlog или loguru: JSON-строки с полями ts, level, event, db, duration_ms, size_bytes, checksum. Логи идут в stderr + опционально в файл --log-file.
- **Test plan:** 1) `pgbackup schedule install --cron '0 3 * * *'` → `crontab -l | grep pgbackup` содержит строку. 2) `pgbackup schedule uninstall` → строка исчезает. 3) Запуск backup пишет в stderr валидный JSON с event='backup.completed', duration_ms, size_bytes — парсится через `python -c 'import json,sys; [json.loads(l) for l in sys.stdin]'` без ошибок.
- **Dependencies:** `sub-2`

### ⬜ 5. Интеграционные e2e-тесты и документация

- **ID:** `sub-5`
- **Profile:** `20razrab1`
- **Status:** `pending`
- **Description:** Полный e2e: docker-compose с postgres 16 → `pgbackup backup --config tests/fixtures/config.toml` → проверить файл → `pgbackup restore --latest` (или через `pg_restore`) в чистую БД → сравнить схему и row counts исходной и восстановленной. README.md с примерами: quickstart, конфиг-файл, cron-установка, troubleshooting. Все тесты должны проходить в CI (GitHub Actions workflow).
- **Test plan:** 1) `docker compose up -d postgres` + `pytest tests/e2e/ -v` — все зелёные. 2) Restore-проверка: row count таблицы `users` в исходной БД == row count в восстановленной БД (через psql -c 'SELECT count(*)'). 3) GitHub Actions workflow запускается на push в PR, видим badge зелёный. 4) README содержит копипаст-рабочий quickstart: 5 команд от клона до первого бэкапа.
- **Dependencies:** `sub-3`, `sub-4`


---
*Generated by Hermes Orchestrator*
