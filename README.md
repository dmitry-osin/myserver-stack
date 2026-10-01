# blog-stack

Совместное развёртывание приложений ahtx.ru и intqst.ru. Исходный код хранится в Git submodules; корневой Compose собирает образы из Dockerfile соответствующих проектов.

Схема: интернет → Caddy (80/443) → blog:8000 для ahtx.ru; для intqst.ru — server:8090 на /api/* и /ws/*, web:80 на остальных путях. Порты приложений на хост не публикуются. Nginx обслуживает статику InterCode во внутренней сети. Данные хранятся в именованных томах и переживают пересоздание контейнеров.

## Запуск на Linux VPS

Нужны Git, Docker Engine и Compose plugin. A/AAAA обоих доменов должны указывать на сервер; устаревшую AAAA запись исправьте или удалите. Откройте TCP 80/443 в облачном и локальном firewall, UDP 443 нужен для HTTP/3. Серверу нужен исходящий доступ к ACME и реестрам образов. Для клонирования приватных репозиториев по SSH настройте доступ к GitHub.

```bash
git clone --recurse-submodules git@github.com:dmitry-osin/myserver-stack.git blog-stack
cd blog-stack
cp .env.example .env
mkdir -p env
cp oxygen-fresh/.env.example env/oxygen.env
chmod 600 .env env/oxygen.env
```

В `.env` задайте `ACME_EMAIL`. В `env/oxygen.env` настройте `ADMIN_PASSWORD_HASH` и `SESSION_SECRET` (от 32 символов), остальные параметры — по необходимости. Bcrypt-хеш заключайте в одинарные кавычки: он содержит `$`. При миграции сохраните прежние значения секретов. `SITE_URL`, `PORT`, `KV_PATH` и `UPLOAD_DIR` переопределяются корневым Compose. Секреты в Git не добавляйте.

```bash
docker compose config --quiet
docker compose build
docker compose run --rm --no-deps caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
docker compose up -d
docker compose ps
docker compose logs --tail=100 caddy
curl -fsS https://ahtx.ru/ >/dev/null
curl -fsS https://intqst.ru/api/healthz
```

Для существующих данных сначала выполните [миграцию](docs/MIGRATION.md). Не запускайте новый стек до восстановления: приложения могут создать пустые данные и секреты.

Caddy автоматически получает сертификаты Let's Encrypt для обоих доменов, обновляет их и перенаправляет HTTP на HTTPS. ACME-состояние и ключи хранятся в `blog-stack-caddy-data`; сохраняйте этот том при пересоздании стека. Отдельные certbot/cron не нужны. См. [автоматический HTTPS](https://caddyserver.com/docs/automatic-https) и [reverse_proxy с поддержкой WebSocket](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy). Access-логи в Caddy по умолчанию не включены, в nginx отключены: адреса комнат InterCode содержат секреты.

## Обновление проектов

Зафиксированные версии:
- oxygen-fresh: 5993a2ae707c0701cd273b2db00516a1d43ec741
- intercode: 2f0b1cd3b6ab2508b5cf6e10174756b50607ff2f

У каждого submodule собственные Git-история и origin. После клонирования submodule обычно находится в detached HEAD: перед изменениями переключитесь на ветку. Пример:

```bash
git -C oxygen-fresh switch main
# Внесите изменения в oxygen-fresh, затем:
git -C oxygen-fresh add <files>
git -C oxygen-fresh commit -m "Update blog"
git -C oxygen-fresh push origin main
git add oxygen-fresh
git commit -m "Update blog revision"
git push
```

Аналогично обновляется intercode. Сначала публикуйте коммит дочернего проекта, затем указатель в корневом репозитории. Корневой репозиторий не включает незакоммиченные изменения внутри submodule. Локальные исходные папки на D: остаются отдельными рабочими копиями; здесь submodules используют свои origin.

Обновление сервера до зафиксированных версий:

```bash
git pull --ff-only
git submodule update --init --recursive
docker compose up -d --build
```

Не используйте `git submodule update --remote` для воспроизводимого запуска. Для отката выберите предыдущий коммит корневого репозитория, выполните `git submodule update --init --recursive` и пересоберите образы. Если менялся формат данных, восстановите совместимую резервную копию.

Образы приложений собираются локально из submodules. Caddy использует подвижный тег `2-alpine`, базовые образы приложений заданы в их Dockerfile. Для строго воспроизводимого запуска фиксируйте digest образов. Корневой origin: `git@github.com:dmitry-osin/myserver-stack.git`.

## Эксплуатация

```bash
docker compose logs --tail=100 blog server caddy
docker compose restart server
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile
```

Не выполняйте `docker compose down -v`: команда удаляет данные и сертификаты. Регулярно сохраняйте резервные копии всех томов, конфигурации и секретов отдельно от Git. Для согласованной копии остановите blog/server и копируйте содержимое томов целиком. Том caddy-data содержит приватные ключи.

Автозапуск контейнеров настроен; Docker должен запускаться при загрузке сервера. HTTPS, сохранность данных после перезапуска, права на тома и работу WebSocket проверяйте на VPS.
