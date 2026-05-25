# Sovereign-Backup-3-2-1: Автономная система резервного копирования на базе Restic, Rclone и Docker (2026)

![License](https://img.shields.io/github/license/sovereign-backup/sovereign-backup-3-2-1?style=flat-square)
![Docker Pulls](https://img.shields.io/docker/pulls/restic/restic?style=flat-square)
![Restic Version](https://img.shields.io/badge/restic-0.16.4-blue?style=flat-square)
![Rclone Version](https://img.shields.io/badge/rclone-1.66.0-orange?style=flat-square)

Полностью автономное, локальноориентированное решение корпоративного класса для резервного копирования по платиновой стратегии «3-2-1». Архитектура обеспечивает сквозное клиентское шифрование (Zero-Knowledge), дедупликацию данных на лету средствами Restic, изоляцию в Docker-контейнерах и отказоустойчивую репликацию инкрементов в независимое S3-облако через Rclone с предварительной аппаратной валидацией накопителей через S.M.A.R.T.

```text
======================= АРХИТЕКТУРНАЯ СХЕМА СИСТЕМЫ 3-2-1 =======================

  [ Рабочая станция / Продакшн-сервер ] (Исходные данные: Копия 1)
                   │
                   ▼
  ┌────────────────────────────────────────────────────────┐
  │ Баш-скрипт автоматизации (Инициация через Cron)        │
  │ ├── 1. Проверка накопителей через smartctl             │
  │ └── 2. Запуск Docker-контейнера Restic/Rclone         │
  └────────────────────────────────────────────────────────┘
                   │
                   ▼ (Локальное шифрование + Дедупликация)
  ┌────────────────────────────────────────────────────────┐
  │ Локальный NAS / СХД (Файловая система: Копия 2)         │ ──► [AFFILIATE_LINK_HARDWARE]
  │ Path: /mnt/nas/snapshots                               │
  └────────────────────────────────────────────────────────┘
                   │
                   ▼ (Потоковое зеркалирование инкрементов)
  ┌────────────────────────────────────────────────────────┐
  │ Удаленный VPS / S3 СХД (Облако: Копия 3)               │ ──► [AFFILIATE_LINK_HOSTING]
  │ Провайдер инфраструктуры                              │     Промокод: [PROMOCODE]
  └────────────────────────────────────────────────────────┘

=================================================================================
```

## Требования к инфраструктуре

Для развертывания отказоустойчивой системы резервного копирования критически важна надежность локального аппаратного слоя. Попытка сэкономить на дисках для NAS приводит к потере данных в момент пиковой нагрузки при построении индексов дедупликации. Локальное хранилище должно быть построено на базе специализированных серверных накопителей, устойчивых к круглосуточной вибрации в режиме 24/7. 

> Важно: Настоятельно рекомендуется собирать локальный массив хранения на дисках корпоративного класса ([WD Red Plus/Pro](https://market.yandex.ru/cc/9WFAHf) или Seagate IronWolf) и использовать только экранированные SATA-кабели с металлическими защелками для предотвращения CRC-ошибок интерфейса. Приобрести надежные жесткие диски и контроллеры под NAS-серверы можно со скидкой [здесь](https://market.yandex.ru/cc/9WFAHf).

Для защиты резервных копий от непреодолимой силы на локальной точке (пожар, конфискация, кража оборудования, затопление) обязательным компонентом является географически удаленный контур. Аренда виртуального сервера (VPS) с подключенным блочным хранилищем или S3-совместимым бакетом позволяет изолировать зашифрованные мета-паки. Высокопроизводительные серверы и облачные S3-хранилища в российском дата-центре с гарантированным аптаймом доступны на [TImeweb Cloud](https://timeweb.cloud/?i=142766).

## Конфигурация и Развертывание

Система разворачивается в изолированном сетевом контуре через Docker Compose. Управление задачами инкрементального бэкапа и последующей репликации делегировано хостовому планировщику `cron`, который взаимодействует с контейнеризированными бинарниками.

### Фрагмент кода: docker-compose.yml

```yaml
version: '3.8'

services:
  sovereign-backup:
    image: alpine:3.19
    container_name: sovereign-backup-orchestrator
    network_mode: "host"
    privileged: true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /dev:/dev:ro
      - /mnt/storage:/data:ro
      - /mnt/nas/snapshots:/repo
      - /root/.config/rclone:/root/.config/rclone
      - /root/.cache/restic:/root/.cache/restic
    environment:
      - RESTIC_REPOSITORY=/repo
      - RESTIC_PASSWORD=SuperSecurePassword2026
      - RCLONE_CONFIG=/root/.config/rclone/rclone.conf
    command: "sleep infinity"
    restart: unless-stopped
```

### Фрагмент кода: Автоматический Bash-скрипт (backup-orchestrator.sh)

Скрипт выполняет каскадную проверку оборудования, снимает локальный инкрементальный снимок, очищает устаревшие snapshot'ы по политике retention и синхронизирует репозиторий с удаленным S3.

```bash
#!/usr/bin/env bash
# ==============================================================================
# Sovereign-Backup-3-2-1 Автоматизированный скрипт оркестрации
# ==============================================================================
set -euo pipefail

# Конфигурационные параметры
TARGET_DISK="/dev/sda"
CONTAINER_NAME="sovereign-backup-orchestrator"
RESTIC_REPO="/repo"
RCLONE_REMOTE="remote-s3-crypt:backup-bucket"
LOG_FILE="/var/log/sovereign-backup.log"

exec > >(tee -ia "${LOG_FILE}") 2>&1

echo "[$(date '+%Y-%m-%d %H:%M:%S')] [INFO] Старт предбекапного цикла валидации..."

# 1. Аппаратный мониторинг S.M.A.R.T.
if ! command -v smartctl &> /dev/null; then
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [WARN] smartctl не найден. Установка пакета smartmontools..."
    apk add --no-cache smartmontools
fi

SMART_STATUS=$(smartctl -H "${TARGET_DISK}" | grep -i "result:" || true)
if [[ ! "${SMART_STATUS}" =~ "PASSED" ]]; then
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [CRITICAL] Накопитель ${TARGET_DISK} сообщает о деградации S.M.A.R.T! Операция прервана." >&2
    exit 1
fi
echo "[$(date '+%Y-%m-%d %H:%M:%S')] [INFO] Здоровье диска подтверждено: ${SMART_STATUS}"

# 2. Инициализация Restic репозитория (если не создан)
docker exec "${CONTAINER_NAME}" sh -c "restic -r ${RESTIC_REPO} rev-list --all &>/dev/null || restic -r ${RESTIC_REPO} init" || true

# 3. Создание инкрементального снимка
echo "[$(date '+%Y-%m-%d %H:%M:%S')] [INFO] Запуск создания снимка Restic..."
docker exec "${CONTAINER_NAME}" restic -r "${RESTIC_REPO}" backup /data \
    --exclude-caches \
    --exclude=".git" \
    --exclude="node_modules"

# 4. Ротация по политике Retention (Удержание: 7 дней, 4 недели, 12 месяцев)
echo "[$(date '+%Y-%m-%d %H:%M:%S')] [INFO] Оптимизация и очистка устаревших снимков..."
docker exec "${CONTAINER_NAME}" restic -r "${RESTIC_REPO}" forget \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 12 \
    --prune

# 5. Репликация в независимое S3 облако через Rclone
echo "[$(date '+%Y-%m-%d %H:%M:%S')] [INFO] Инициация зеркалирования инкрементов в удаленный контур..."
docker exec "${CONTAINER_NAME}" rclone sync "${RESTIC_REPO}" "${RCLONE_REMOTE}" \
    --fast-list \
    --transfers 4 \
    --checkers 8 \
    --contimeout 60s \
    --stats 1m

echo "[$(date '+%Y-%m-%d %H:%M:%S')] [SUCCESS] Цикл резервного копирования 3-2-1 успешно завершен."
```

## Troubleshooting: Частые ошибки

### Ошибка 1: Блокировка репозитория процессами
* **Текст ошибки из консоли:**
  ```text
  unable to create lock in backend: repository is locked by PID 14203 on host storage-node by root (opened at 2026-05-20 04:12:11, age 14h2m1s)
  lock was created by UID 0
  ```
* **Решение:** Данное состояние возникает при аварийном завершении скрипта (например, при внезапном отключении питания хоста или Out-Of-Memory kill). Restic защищает целостность метаданных эксклюзивным локом. Для восстановления работоспособности убедитесь, что другие процессы бэкапа не запущены, и выполните команду снятия блокировки:
  ```bash
  docker exec -it sovereign-backup-orchestrator restic -r /repo unlock
  ```

### Ошибка 2: Хеш-несовпадение и повреждение паков при сетевом сбое
* **Текст ошибки из консоли:**
  ```text
  Load(<pack/3a9f8b2c...>, 0, 0) returned error, output: pack is corrupt or hash mismatch
  Pack ID: 3a9f8b2c7e4d8f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a
  Error: Integrity check failed
  ```
* **Решение:** Происходит при деградации каналов связи во время вызова --prune или при сбоях оперативной памяти без ECC. Требуется перестроить индексный файл и запустить сборку мусора с полной сверкой векторов:
  ```bash
  # Шаг 1: Перестроение индексов репозитория
  docker exec -it sovereign-backup-orchestrator restic -r /repo rebuild-index
  
  # Шаг 2: Поиск поврежденных паков и их изоляция
  docker exec -it sovereign-backup-orchestrator restic -r /repo check --read-data
  
  # Шаг 3: Удаление битых ссылок, если файлы не подлежат восстановлению
  docker exec -it sovereign-backup-orchestrator restic -r /repo prune
  ```
