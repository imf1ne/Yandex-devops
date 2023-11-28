# Yandex-devops
Итоговый проект тренировок yandex-devops


## ТЗ кратко:
Дан бинарный исполняемый файл. Необходимо развернуть отказоустойчивую инсталляцию приложения из имеющегося бинарника до
даты запуска продукта. Планируется стабильная нагрузка в 60 RPS, пиковая в 120 RPS.
Полное ТЗ [тут](https://docs.yandex.ru/docs/view?url=ya-disk-public%3A%2F%2FaZFzSWnWcEBeFUQaZTVH7mko2C8kWAY5drce3M9qlgYGtAG6xLa0qQ37gSVWb4T4q%2FJ6bpmRyOJonT3VoXnDag%3D%3D&name=%D0%98%D1%82%D0%BE%D0%B3%D0%BE%D0%B2%D1%8B%D0%B9%20%D0%BF%D1%80%D0%BE%D0%B5%D0%BA%D1%82.docx&nosw=1)

# Локальное исследование черного ящика



### Выбор стека технологий

Проект построен на базе Yandex Cloud.
С базой данных все понятно. А на чем строить отказоустоичивость и балансировку нагрузки надо решать.
В качестве ОС я выбрал Ubuntu 20.04.
А инсталляцию решил развернуть на nginx, и ноды, и сервер балансировки. Итого - 3 машины.

# Архитектура проекта



# Ускорение старта (Патч бинаря)



# Балансер

Выбрана версия nginx - 1.25.3. (Можно закрыть пункт с поддержкой http3, так как эта версия его поддерживает)
```
sudo apt install curl gnupg2 ca-certificates lsb-release ubuntu-keyring
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
    | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
gpg --dry-run --quiet --no-keyring --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
http://nginx.org/packages/mainline/ubuntu `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
    | sudo tee /etc/apt/preferences.d/99nginx
sudo apt update
sudo apt install nginx
```

Отредактируем конфигурационный файл /etc/nginx/conf.d/default.conf.
В нем настроен upstream и http3. (Также тут есть ендпоинт status для дальнейшего снятия метрик)

```
upstream backend {
      server 192.168.10.4:80;
      server 192.168.10.18:80;
}

proxy_cache_path /var/cache/nginx keys_zone=cache_zone:1m;

server {
    listen 80;
    listen [::]:80;
    server_name bingo.myddns.me;

    location /long_dummy {
       proxy_cache cache_zone;
       proxy_cache_valid 1m;
       proxy_pass http://backend;
    }

    location / {
       proxy_pass http://backend;
       proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504 non_idempotent;
       proxy_intercept_errors on;
       proxy_http_version 1.1;
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
      }
}

log_format quic '$remote_addr - $remote_user [$time_local] '
                '"$request" $status $body_bytes_sent '
                '"$http_referer" "$http_user_agent" "$http3"';

access_log logs/access.log quic;

server {
    listen 443 quic reuseport; # QUIC
    listen 443 ssl http2;      # TCP
    server_name bingo.myddns.me;
    add_header Alt-Svc 'h3=":443"; ma=3600';
    ssl_early_data on;
    ssl_certificate /etc/letsencrypt/live/bingo.myddns.me/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/bingo.myddns.me/privkey.pem; # managed by Certbot
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;

    location /long_dummy {
       proxy_cache cache_zone;
       proxy_cache_valid 1m;
       proxy_pass http://backend;
    }

    location / {
       add_header Alt-Svc 'h3=":$server_port"; ma=86400';
       add_header X-protocol $server_protocol always;
       # include proxy_params;
       proxy_pass http://backend;
       proxy_http_version 1.1;
       proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504 non_idempotent;
       proxy_intercept_errors on;
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
      }
}

server {
    listen 8080;
    # Optionally: allow access only from localhost
    # listen 127.0.0.1:8080;

    server_name bingo;

    location /status {
        stub_status;
    }
}
```


# Нода 2

Запуск bingo как службы:

Создаем файл /etc/systemd/system/bingo.service

```
[Unit]
Description=bingo
After=multi-user.target

[Service]
Type=simple
User=dim
Restart=always
RestartSec=2
WorkingDirectory=/home/dim
ExecStart=/home/dim/bingo run_server

[Install]
WantedBy=multi-user.target
```

sudo systemctl daemon-reload
sudo systemctl enable bingo
sudo systemctl start bingo

Установка nginx

/etc/nginx/sites-available/default
```
server {
    listen 192.168.10.4:80;
    #listen [::]:80;
    server_name bingo.myddns.me;
    #location / {
    #    return 301 http://$host$request_uri:13176;
    #}
    location / {
        proxy_pass  http://127.0.0.1:7939;
    }
    allow 192.168.10.0/24;
    deny all;
}
```

# Ноды 1
Аналогично ноде 2.
Добавляется установка и настройка БД postgesql.


## Настройка базы данных

```bash
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" | sudo tee /etc/apt/sources.list.d/postgresql-pgdg.list &gt; /dev/null
sudo apt-get update
sudo apt-get install postgresql-14 postgresql-contrib
sudo systemctl start postgresql
```

Установка пароля пользователю postgres:
```bash
sudo -u postgres psql
ALTER USER postgres WITH PASSWORD 'postgres';
```


# Сбор метрик (Prometeus + Grafana)

