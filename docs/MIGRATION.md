# Миграция с текущих серверов

Инструкция основана на конфигурации репозиториев, а не на проверке production. Имена контейнеров, IP, фактические пути данных и схема запуска требуют проверки на сервере. Все команды ниже выполняются в Bash на Linux. Сохраните старый стек и резервные копии: удалять их можно только после проверки нового сервера и завершения периода отката.

## Что переносить

| Сервис | Путь внутри контейнера по исходному Compose | Содержимое |
| --- | --- | --- |
| oxygen-fresh | /app/data | Deno KV: kv.sqlite3 и все сопутствующие файлы SQLite |
| oxygen-fresh | /app/static/uploads | Загруженные изображения и файлы |
| intercode server | /app/data | sessions/*.json, docs/*.bin, telemetry/*.jsonl |
| Новый Caddy | /data и /config | ACME-состояние, сертификаты, приватные ключи и конфигурация |

В исходном Compose блога указан GHCR-образ `sha-725264e`, поэтому версия production может отличаться от submodule. InterCode может запускаться из локальной сборки или GHCR latest, с nginx, certbot и cron. Не запускайте прежний `setup-ssl.sh` поверх нового стека. Новый Caddy получает собственные сертификаты; старый `/etc/letsencrypt` сохраните для отката.

При первом запуске блог может создавать данные в KV: восстановите хранилище до запуска приложения. Сохраните действующие `SESSION_SECRET`, bcrypt-хеш, настройки и секреты. Авторизацию, сессии и настройки проверяйте после переноса всей KV, а не отдельных записей.

InterCode сохраняет документы и телеметрию в файлы, при SIGTERM выполняет финальное сохранение. Неактивные комнаты могут удаляться через 12 часов, поэтому учитывайте срок хранения. Перед переносом проверьте обработку завершения в используемой версии; для согласованной копии остановите приём изменений, завершите сервер штатно и скопируйте данные.

## 1. Инвентаризация и подготовка

На старом сервере:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}'
docker compose ls
# Укажите фактические имена контейнеров:
export OLD_BLOG=oxygen-blog
export OLD_SERVER=<intercode-server-container>
docker inspect "$OLD_BLOG" --format '{{json .Mounts}}'
docker inspect "$OLD_SERVER" --format '{{json .Mounts}}'
docker inspect "$OLD_BLOG" --format '{{.Config.Image}} {{.Image}}'
docker inspect "$OLD_SERVER" --format '{{.Config.Image}} {{.Image}}'
crontab -l
sudo ss -ltnp | head -40
```

Проверьте фактические `KV_PATH`, `UPLOAD_DIR`, `DATA_DIR` и mount-пути работающих контейнеров. Если пути отличаются, скорректируйте команды копирования. Выясните, кто занимает 80/443: nginx хоста, контейнер web или другой proxy. Сохраните старые Compose, .env, конфигурацию, cron, версии и digest образов в защищённом месте. Проверьте также системные cron-задачи обновления сертификатов.

Клонируйте корневой репозиторий с submodules и настройте его по README. Выполните `docker compose config --quiet` и `docker compose build`, пока не запуская приложения. Сравните версии нового стека с production; при необходимости зафиксируйте совместимые версии submodules. Заранее уменьшите DNS TTL, если меняется сервер.

## 2. Согласованная копия на старом сервере

Закройте доступ на запись и остановите приложения перед копированием. Дождитесь завершения server и убедитесь по логам, что InterCode сохранил данные. Проверьте отсутствие принудительного SIGKILL и кода выхода 137.

```bash
umask 077
mkdir -p migration-backup/{blog-data,blog-uploads,intercode-data}
docker stop --time 60 "$OLD_SERVER" "$OLD_BLOG"
docker logs --tail=50 "$OLD_SERVER"
docker inspect "$OLD_SERVER" --format '{{.State.ExitCode}}'
# docker cp работает и с остановленными контейнерами.
docker cp "$OLD_BLOG":/app/data/. migration-backup/blog-data/
docker cp "$OLD_BLOG":/app/static/uploads/. migration-backup/blog-uploads/
docker cp "$OLD_SERVER":/app/data/. migration-backup/intercode-data/
tar -C migration-backup -czf blog-data.tar.gz blog-data
tar -C migration-backup -czf blog-uploads.tar.gz blog-uploads
tar -C migration-backup -czf intercode-data.tar.gz intercode-data
sha256sum blog-data.tar.gz blog-uploads.tar.gz intercode-data.tar.gz > SHA256SUMS
```

Копируйте весь каталог KV после остановки, а не только kv.sqlite3 работающего приложения: WAL/SHM могут содержать последние изменения. Перенесите три архива, SHA256SUMS и секреты на новый сервер через SSH/SCP. Храните копии в защищённом каталоге: они содержат данные комнат и учётные данные. Не запускайте старые приложения во время переключения.

## 3. Восстановление в новые тома

На новом сервере, из каталога blog-stack:

```bash
mkdir -p backups
# Скопируйте архивы и SHA256SUMS в backups.
(cd backups && sha256sum -c SHA256SUMS)
# create создаёт остановленные контейнеры и тома, не запуская приложения.
docker compose create blog server web
docker volume inspect blog-stack-blog-data blog-stack-blog-uploads blog-stack-intercode-data
```

Восстанавливайте данные только в новые пустые тома. Скрипт прервётся, если том уже содержит файлы; не обходите проверку для существующих данных.

```bash
restore_volume() {
  local volume="$1" archive="$2" directory="$3"
  docker run --rm \
    -v "$volume":/target \
    -v "$PWD/backups":/backup:ro \
    alpine:3.22 sh -ec '
      test -z "$(ls -A /target)" || { echo "Target volume is not empty" >&2; exit 1; }
      tar -xzf "/backup/$1" -C /target --strip-components=1 "$2"
    ' sh "$archive" "$directory"
}
restore_volume blog-stack-blog-data blog-data.tar.gz blog-data
restore_volume blog-stack-blog-uploads blog-uploads.tar.gz blog-uploads
restore_volume blog-stack-intercode-data intercode-data.tar.gz intercode-data
# Node-приложение работает от node; в node:20-alpine UID/GID равны 1000.
# Проверьте пользователя в используемом образе и настройте права:
docker compose run --rm --no-deps --user root server chown -R node:node /app/data
```

При переносе на том же сервере можно использовать существующие тома, указав их имена в корневом Compose: сначала проверьте содержимое и сделайте копию. Если старый стек ещё работает, остановите его; для изоляции миграции предпочтительны новые тома из этого Compose.

## 4. Переключение HTTPS и проверка

Остановите старые web/proxy, занимающие 80/443, либо перенастройте nginx хоста. Убедитесь, что порты свободны. Отключите certbot cron и задачи, установленные setup-ssl.sh, только для переносимых сайтов; проверьте задачу intercode-certbot-renew. Сохраните прежнюю конфигурацию для отката.

При смене сервера обновите A/AAAA ahtx.ru и intqst.ru и дождитесь распространения DNS. Откройте TCP 80/443 и исходящий доступ к Let's Encrypt; UDP 443 нужен для HTTP/3. Для HTTP-проверки ACME DNS должен вести на Caddy. Старые сертификаты импортировать в Caddy не требуется.

```bash
docker compose run --rm --no-deps caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
docker compose up -d
docker compose ps
docker compose logs --tail=100 caddy blog server
curl -fsSI http://ahtx.ru/
curl -fsSI http://intqst.ru/
curl -fsS https://ahtx.ru/ >/dev/null
curl -fsS https://intqst.ru/api/healthz
```

Не используйте `curl -k` при проверке сертификатов. Проверьте цепочку сертификата и HTTP → HTTPS в браузере. В блоге проверьте вход администратора, старые публикации, изображения и создание записи. В InterCode проверьте существующие документы и обмен изменениями между двумя клиентами: синхронизацию, WebSocket со статусом 101, переподключение и сохранение. HTTP healthz не проверяет WebSocket. Перезапустите стек и убедитесь, что данные сохранились.

Автоматическое обновление ACME проверяйте по состоянию Caddy и логам в течение эксплуатации. Оно зависит от доступности DNS/сети и сохранности caddy-data. После проверки подождите прежний DNS TTL и завершите период наблюдения перед удалением старого стека.

## 5. Откат

Остановите новый стек: `docker compose down` (без `-v`). На том же сервере верните старые контейнеры и конфигурацию, запустите прежний proxy и восстановите нужные certbot-задачи. При смене сервера верните A/AAAA на прежний IP. Учитывайте TTL и DNS-кеши.

Старая копия содержит данные до переключения; записи, созданные на новом сервере, в ней отсутствуют. Перед откатом остановите запись и подготовьте согласованный перенос новых данных, если формат совместим. Не запускайте оба стека для записи одновременно. Старые тома и резервные копии удаляйте только после проверки миграции и завершения периода отката.
