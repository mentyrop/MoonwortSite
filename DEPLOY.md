# Деплой moonwort.ru

Лендинг — это **статика** (`index.html`, файлы знака и внешние шрифты). Никакой
сборки, БД и бэкенда. Задача: отдавать каталог сайта по адресу
`https://moonwort.ru`.

DNS на `moonwort.ru` уже указывает на сервер. Осталось: (1) положить файлы на
сервер, (2) настроить веб-сервер отдавать их на этом домене, (3) выпустить TLS.

---

## Шаг 1. Забрать файлы на сервер

Рядом с сервисом (например в `/opt`):

```bash
cd /opt
git clone git@github.com:mentyrop/MoonwortSite.git moonwort-site
# для хостинга нужен весь каталог moonwort-site
```

> Если на сервере нет SSH-ключа с доступом к репозиторию — клонируй по HTTPS:
> `git clone https://github.com/mentyrop/MoonwortSite.git moonwort-site`

Обновление сайта потом:

```bash
cd /opt/moonwort-site && git pull
```

Дальше выбери сценарий по тому, **что сейчас держит порты 80/443** на сервере.

---

## Сценарий A. На хосте стоит nginx (reverse-proxy для сервиса)

Самый частый случай. Добавляем ещё один site.

```bash
sudo tee /etc/nginx/sites-available/moonwort.ru >/dev/null <<'NGINX'
server {
    listen 80;
    listen [::]:80;
    server_name moonwort.ru www.moonwort.ru;
    root /opt/moonwort-site;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
NGINX

sudo ln -s /etc/nginx/sites-available/moonwort.ru /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

TLS сертификат (certbot сам допишет 443-блок и редирект с http):

```bash
sudo certbot --nginx -d moonwort.ru -d www.moonwort.ru
```

Готово. Обновление: `cd /opt/moonwort-site && git pull` — nginx подхватит сразу,
перезапуск не нужен.

---

## Сценарий B. Reverse-proxy уже в Docker (Traefik или nginx-proxy)

Поднимаем крошечный контейнер `nginx:alpine`, который монтирует папку и получает
маршрут + автоTLS от твоего прокси. Файл `docker-compose.yml` рядом с сайтом.

### Если прокси — Traefik

```yaml
services:
  moonwort-site:
    image: nginx:alpine
    restart: unless-stopped
    volumes:
      - /opt/moonwort-site:/usr/share/nginx/html:ro
    networks:
      - proxy            # та же сеть, что у Traefik
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.moonwort.rule=Host(`moonwort.ru`) || Host(`www.moonwort.ru`)"
      - "traefik.http.routers.moonwort.entrypoints=websecure"
      - "traefik.http.routers.moonwort.tls.certresolver=le"   # имя твоего resolver'а
      - "traefik.http.services.moonwort.loadbalancer.server.port=80"

networks:
  proxy:
    external: true       # имя внешней сети Traefik
```

### Если прокси — nginx-proxy + acme-companion (jwilder)

```yaml
services:
  moonwort-site:
    image: nginx:alpine
    restart: unless-stopped
    volumes:
      - /opt/moonwort-site:/usr/share/nginx/html:ro
    environment:
      - VIRTUAL_HOST=moonwort.ru,www.moonwort.ru
      - LETSENCRYPT_HOST=moonwort.ru,www.moonwort.ru
      - LETSENCRYPT_EMAIL=you@example.com
    networks:
      - proxy            # та же сеть, что у nginx-proxy
networks:
  proxy:
    external: true
```

Запуск:

```bash
cd /opt/moonwort-site && docker compose up -d
```

Подставь **имя своей proxy-сети** и **имя certresolver'а** (для Traefik) —
их видно в compose-файле твоего текущего сервиса. Обновление сайта: `git pull`
(контейнер монтирует папку read-only, файлы обновляются на лету, перезапуск не нужен).

---

## Сценарий C. Ничего нет / хочу отдельно и просто — Caddy

Caddy сам получает и продлевает TLS. Один контейнер:

```yaml
services:
  caddy:
    image: caddy:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /opt/moonwort-site:/srv:ro
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
volumes:
  caddy_data:
```

`Caddyfile`:

```
moonwort.ru, www.moonwort.ru {
    root * /srv
    file_server
    try_files {path} /index.html
}
```

> Порты 80/443 должны быть свободны — не запускай, если их уже занял другой прокси
> (тогда тебе сценарий A или B).

```bash
cd /opt/moonwort-site && docker compose up -d
```

---

## Проверка

```bash
curl -I https://moonwort.ru      # ждём 200 OK
```

И открыть `https://moonwort.ru` в браузере.
