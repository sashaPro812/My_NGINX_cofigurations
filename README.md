## ОГЛАВЛЕНИЕ

1. [Архитектура NGINX](#1-архитектура-nginx)
2. [Установка NGINX](#2-установка-nginx)
3. [Структура директорий](#3-структура-директорий)
4. [Структура конфигурационного файла](#4-структура-конфигурационного-файла)
5. [Основные команды](#5-основные-команды)
6. [Директивы listen и server_name](#6-директивы-listen-и-server_name)
7. [Базовые директивы](#7-базовые-директивы)
8. [Виртуальные хосты (Server Blocks)](#8-виртуальные-хосты-server-blocks)
9. [Логирование](#9-логирование)
10. [default_server и приоритеты](#10-default_server-и-приоритеты)
11. [Переменные NGINX (полный список)](#11-переменные-nginx-полный-список)
12. [Модуль 3: Location и маршрутизация](#12-модуль-3-location-и-маршрутизация)
13. [Модуль 4: Proxy_pass и балансировка](#13-модуль-4-proxy_pass-и-балансировка)
14. [Модуль 5: Безопасность и SSL](#14-модуль-5-безопасность-и-ssl)
15. [Модуль 6: Оптимизация и кеширование](#15-модуль-6-оптимизация-и-кеширование)
16. [Модуль 7: Мониторинг и отладка](#16-модуль-7-мониторинг-и-отладка)
17. [Чек-лист: 15 обязательных компонентов](#17-чек-лист-15-обязательных-компонентов)
18. [Полный шаблон продакшен-конфига](#18-полный-шаблон-продакшен-конфига)

---

## 1. АРХИТЕКТУРА NGINX

### ТЕОРИЯ:

NGINX использует **асинхронную, событийно-ориентированную** архитектуру. Это означает, что один рабочий процесс (воркер) может обрабатывать тысячи соединений одновременно, в отличие от Apache, который создает отдельный процесс или поток на каждое соединение.

**Ключевые компоненты:**

| Компонент | Описание |
|-----------|----------|
| **Мастер-процесс** | Запускается от root. Читает конфиги, управляет воркерами, выполняет reload. |
| **Воркеры** (worker processes) | Обрабатывают фактические запросы. Работают от пользователя www-data. Количество обычно = числу ядер CPU. |
| **Кеш-менеджер** | Управляет кешем на диске (очистка, индексация). |
| **Кеш-загрузчик** | Загружает кеш в память при старте. |

**Механизм работы:**
1. Мастер-процесс читает конфиг и создает сокеты.
2. Воркеры конкурируют за новые соединения (или используют `reuseport`).
3. Воркер принимает соединение, обрабатывает его (неблокирующий I/O).
4. Используется **epoll** (Linux) или **kqueue** (BSD) для мультиплексирования.

**Отличие от Apache:**
- Apache: 1 процесс/поток на запрос → тяжеловесно, потребляет память.
- NGINX: 1 воркер на тысячи запросов → легковесно, экономит память.

---

### 🚀 ШПАРГАЛКА:

```nginx
# ============================================
# ОСНОВНЫЕ ДИРЕКТИВЫ АРХИТЕКТУРЫ
# ============================================

# --- КОНТЕКСТ: main (главный) ---

user www-data;                    # Пользователь Linux, от которого работают воркеры
                                  # www-data — стандартный для Ubuntu/Debian
                                  # nginx — для CentOS/RHEL
                                  # Другие варианты: nobody, nginx, или любой системный пользователь

worker_processes auto;            # Количество воркеров
                                  # auto = автоматически = кол-ву ядер CPU
                                  # Можно указать число: worker_processes 4;
                                  # Для высоконагруженных: worker_processes 8;
                                  # Для статики: worker_processes auto;

worker_rlimit_nofile 65535;       # Максимальное количество открытых файлов на воркера
                                  # По умолчанию: 1024
                                  # Для высоких нагрузок: 65535 или 100000
                                  # Должно быть >= worker_connections * 2

pid /run/nginx.pid;               # Путь к файлу с PID мастер-процесса
                                  # Используется для команд: nginx -s reload

# --- КОНТЕКСТ: events ---

events {
    worker_connections 1024;      # Максимум соединений на один воркер
                                  # По умолчанию: 512
                                  # Общее = worker_processes * worker_connections
                                  # Например: 4 * 1024 = 4096 одновременных соединений
                                  # Для высоких нагрузок: 1024-4096 на ядро
    
    use epoll;                    # Механизм мультиплексирования I/O
                                  # epoll — Linux (самый быстрый, для современных систем)
                                  # kqueue — FreeBSD/macOS
                                  # select — универсальный, но медленный (до 1024 соединений)
                                  # poll — универсальный, медленнее epoll
                                  # eventport — Solaris
                                  # Если не указан, NGINX выбирает сам
    
    multi_accept on;              # Принимать несколько соединений за один системный вызов
                                  # on — улучшает производительность при высокой нагрузке
                                  # off — по умолчанию (принимает по одному)
}
```

---

## 2. УСТАНОВКА NGINX

### ТЕОРИЯ:

NGINX можно установить двумя основными способами:
1. **Из официального репозитория** — свежая стабильная версия (рекомендуется).
2. **Из стандартного репозитория дистрибутива** — версия старше, но стабильнее.

**Важно:** После установки NGINX автоматически запускается и добавляется в автозагрузку.

---

### 🚀 ШПАРГАЛКА:

```bash
# ============================================
# УСТАНОВКА ИЗ ОФИЦИАЛЬНОГО РЕПОЗИТОРИЯ (рекомендуется)
# ============================================

# 1. Добавляем официальный ключ NGINX (для проверки подписи пакетов)
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
# gpg --dearmor — преобразует ключ в формат, понятный APT

# 2. Добавляем репозиторий в список источников
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/ubuntu `lsb_release -cs` nginx" | sudo tee /etc/apt/sources.list.d/nginx.list
# lsb_release -cs — подставляет кодовое имя вашей Ubuntu (jammy, focal, bionic)

# 3. Устанавливаем NGINX
sudo apt update                     # Обновляем кеш пакетов
sudo apt install nginx -y           # -y = автоматически соглашаться на установку

# ============================================
# АЛЬТЕРНАТИВА (стандартный репозиторий)
# ============================================

sudo apt update
sudo apt install nginx -y           # Версия будет старше, но стабильнее

# ============================================
# ПРОВЕРКА УСТАНОВКИ
# ============================================

nginx -v                            # Показать версию NGINX
                                    # Вывод: nginx version: nginx/1.26.0

nginx -V                            # Версия + список скомпилированных модулей
                                    # Показывает все модули: --with-http_ssl_module и т.д.

systemctl status nginx              # Проверить статус сервиса
                                    # Active: active (running) — работает
                                    # Active: inactive (dead) — остановлен

sudo nginx -t                       # Проверить синтаксис конфигов
                                    # Вывод: syntax is ok, test is successful
```

---

## 3. СТРУКТУРА ДИРЕКТОРИЙ

### ТЕОРИЯ:

NGINX имеет четкую иерархию папок, каждая из которых выполняет свою роль. Понимание этой структуры критически важно для администрирования.

**Порядок загрузки конфигов:**
1. Читается `/etc/nginx/nginx.conf`
2. Внутри него — `include /etc/nginx/conf.d/*.conf;` (в алфавитном порядке)
3. Затем — `include /etc/nginx/sites-enabled/*;` (в алфавитном порядке)

**Важно:** Файлы читаются в **алфавитном порядке**. Используйте числовые префиксы для контроля: `00-default.conf`, `01-gzip.conf`, `99-custom.conf`.

---

### 🚀 ШПАРГАЛКА:

```bash
# ============================================
# ОСНОВНЫЕ ПАПКИ И ИХ НАЗНАЧЕНИЕ
# ============================================

/etc/nginx/nginx.conf               # ГЛАВНЫЙ КОНФИГ — основа всего
                                    # Без него NGINX не запустится
                                    # Содержит: user, worker_processes, events, http

/etc/nginx/conf.d/                  # Глобальные настройки (читаются автоматически)
                                    # Сюда кладут:
                                    #   - 01-gzip.conf     (сжатие)
                                    #   - 02-security.conf (заголовки безопасности)
                                    #   - 03-ssl.conf      (SSL настройки)
                                    #   - 04-limits.conf   (лимиты и rate limiting)
                                    #   - 05-cache.conf    (зоны кеша)
                                    #   - 06-upstream.conf (бекенды)
                                    # Файлы читаются в алфавитном порядке!

/etc/nginx/sites-available/         # Хранилище конфигов ВСЕХ сайтов
                                    # Даже тех, которые временно отключены
                                    # Это как "архив" или "библиотека"

/etc/nginx/sites-enabled/           # Активные сайты (только симлинки из available)
                                    # Что здесь есть — то работает
                                    # Что здесь нет — то выключено

/usr/share/nginx/html/              # Стандартная папка для статики
                                    # НЕ РЕКОМЕНДУЕТСЯ использовать
                                    # Лучше: /var/www/

/var/www/                           # Рекомендуемая папка для ваших сайтов
                                    # Создаете подпапки: /var/www/site1/, /var/www/site2/

/var/log/nginx/access.log           # Общий лог ВСЕХ запросов
                                    # Если не настроили свои логи для каждого сайта

/var/log/nginx/error.log            # Общий лог ВСЕХ ошибок

/etc/nginx/mime.types               # Сопоставление расширений с MIME-типами
                                    # .css → text/css
                                    # .jpg → image/jpeg
                                    # .html → text/html

# ============================================
# ПРОВЕРКА ПОРЯДКА ЗАГРУЗКИ
# ============================================

sudo nginx -T 2>/dev/null | grep -E "^# configuration file"
# Показывает все файлы в порядке их загрузки
# Вывод:
# # configuration file /etc/nginx/nginx.conf
# # configuration file /etc/nginx/conf.d/01-gzip.conf
# # configuration file /etc/nginx/conf.d/02-security.conf
# # configuration file /etc/nginx/sites-enabled/example.com.conf
```

---

## 4. СТРУКТУРА КОНФИГУРАЦИОННОГО ФАЙЛА

### ТЕОРИЯ:

Конфигурация NGINX построена по **иерархическому принципу** с вложенными контекстами. Каждый контекст имеет свои директивы и наследует настройки от родительского контекста.

**Иерархия контекстов:**
```
main (глобальный)
  ├── events (настройки соединений)
  └── http (веб-трафик)
        └── server (виртуальный хост)
              └── location (маршрутизация URI)
                    └── location (вложенный)
```

**Наследование:** Директивы из родительского контекста действуют во всех дочерних, если не переопределены.

---

### 🚀 ШПАРГАЛКА (скелет с комментариями):

```nginx
# ============================================
# КОНТЕКСТ: main (главный)
# ============================================

user www-data;                      # Пользователь Linux для воркеров
worker_processes auto;              # Количество воркеров
pid /run/nginx.pid;                 # Файл с PID

# ============================================
# КОНТЕКСТ: events (управление соединениями)
# ============================================

events {
    worker_connections 1024;        # Макс. соединений на воркер
    use epoll;                      # Механизм мультиплексирования
    multi_accept on;                # Принимать несколько соединений за раз
}

# ============================================
# КОНТЕКСТ: http (веб-трафик)
# ============================================

http {
    # --------------------------------------------
    # Базовые настройки HTTP
    # --------------------------------------------
    
    include /etc/nginx/mime.types;   # Подключаем MIME-типы
    default_type application/octet-stream;  # MIME-тип по умолчанию
    
    # --------------------------------------------
    # Формат логов
    # --------------------------------------------
    
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';
    
    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;
    
    # --------------------------------------------
    # БЛОК SERVER (виртуальный хост)
    # --------------------------------------------
    
    server {
        listen 80;                      # Порт
        server_name _;                  # Имя сервера
        
        root /var/www/mysite;           # Корневая папка
        index index.html;               # Индексные файлы
        
        access_log /var/log/nginx/mysite_access.log main;
        error_log /var/log/nginx/mysite_error.log error;
        
        # --------------------------------------------
        # БЛОК LOCATION (маршрутизация)
        # --------------------------------------------
        
        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

---

## 5. ОСНОВНЫЕ КОМАНДЫ

### ТЕОРИЯ:

Все команды NGINX можно разделить на:
1. **Проверочные** — проверка синтаксиса, просмотр конфига.
2. **Управляющие** — запуск, остановка, перезагрузка.
3. **Информационные** — версия, модули, статус.

**Важно:** Всегда сначала делайте `sudo nginx -t`, а потом `sudo nginx -s reload`. Это спасет от недоступности сервера из-за ошибок в конфиге.

---

### 🚀 ШПАРГАЛКА:

```bash
# ============================================
# ПРОВЕРКА КОНФИГУРАЦИИ (ВСЕГДА ПЕРВЫМ!)
# ============================================

sudo nginx -t                       # Проверка синтаксиса
                                    # Вывод: nginx: configuration file /etc/nginx/nginx.conf test is successful

sudo nginx -T                       # Проверка + вывод полной конфигурации
                                    # Показывает все файлы и их содержимое

sudo nginx -t -c /path/to/nginx.conf  # Проверить конкретный файл конфига

# ============================================
# УПРАВЛЕНИЕ СЕРВЕРОМ
# ============================================

sudo nginx -s reload                # Перезагрузить конфиг БЕЗ остановки
                                    # Бесшовно! Не роняет соединения

sudo nginx -s stop                  # Быстрая остановка (аварийная)
                                    # Обрывает все соединения

sudo nginx -s quit                  # Плавная остановка
                                    # Дожидается завершения всех запросов

sudo nginx -s reopen                # Переоткрыть логи (для logrotate)

# ============================================
# ИНФОРМАЦИЯ
# ============================================

nginx -v                            # Версия NGINX
nginx -V                            # Версия + список модулей (--with-*)

# ============================================
# SYSTEMD КОМАНДЫ (альтернатива)
# ============================================

sudo systemctl status nginx         # Статус сервиса
sudo systemctl reload nginx         # Перезагрузить конфиг
sudo systemctl restart nginx        # Полный перезапуск
sudo systemctl stop nginx           # Остановить
sudo systemctl start nginx          # Запустить
sudo systemctl enable nginx         # Добавить в автозагрузку
sudo systemctl disable nginx        # Убрать из автозагрузки

# ============================================
# ДИАГНОСТИКА
# ============================================

sudo ss -tulpn | grep nginx         # Проверить, какие порты слушает NGINX
sudo journalctl -u nginx -f         # Логи systemd в реальном времени
sudo journalctl -u nginx --since "1 hour ago"  # Логи за последний час

# ============================================
# ПОИСК В КОНФИГАХ
# ============================================

grep -r "server_name" /etc/nginx/   # Найти все server_name
grep -r "proxy_pass" /etc/nginx/    # Найти все proxy_pass
```

---

## 6. ДИРЕКТИВЫ listen И server_name

### ТЕОРИЯ:

**`listen`** — определяет, на каком IP-адресе, порту и с какими параметрами NGINX будет принимать соединения.

**`server_name`** — определяет, для каких доменных имен этот `server` блок будет обрабатывать запросы.

**Приоритеты server_name:**
1. Точное совпадение (высший приоритет)
2. Wildcard (средний приоритет)
3. Регулярное выражение (низший приоритет)
4. `default_server` (если указан)
5. Первый загруженный (неявный default)

---

### 🚀 ШПАРГАЛКА (с возможными значениями):

```nginx
# ============================================
# listen — ВСЕ ВОЗМОЖНЫЕ ЗНАЧЕНИЯ
# ============================================

# --- Базовые варианты ---
listen 80;                          # Порт 80 на всех IP (IPv4 и IPv6)
listen 8080;                        # Порт 8080 на всех IP
listen [::]:80;                     # Только IPv6
listen 127.0.0.1:8080;              # Только локальный интерфейс
listen 192.168.1.100:80;            # Конкретный IP-адрес

# --- Unix-сокет ---
listen unix:/var/run/nginx.sock;    # Unix-сокет (вместо TCP/IP)

# --- Флаги ---
listen 80 default_server;           # default_server — fallback для неизвестных доменов
listen 80 ssl;                      # ssl — включить SSL/TLS (HTTPS)
listen 80 ssl http2;                # http2 — включить HTTP/2
listen 80 ssl http2 default_server; # Комбинация нескольких флагов

# --- Параметры производительности ---
listen 80 backlog=4096;             # backlog — размер очереди ожидающих соединений
                                    # По умолчанию: зависит от ОС (обычно 511-1024)
                                    # Для высоких нагрузок: 4096-65535

listen 80 reuseport;                # reuseport — разделить порт между воркерами
                                    # Улучшает балансировку при высокой нагрузке
                                    # Требует поддержки ядра (Linux 3.9+)

listen 80 so_keepalive=on;          # so_keepalive — включить TCP keepalive
                                    # on, off, или [keepidle]:[keepintvl]:[keepcnt]

# --- Proxy Protocol (для балансировщиков) ---
listen 80 proxy_protocol;           # Принимать PROXY-протокол
                                    # Используется за HAProxy, AWS ELB

# --- Deferred (отложенная обработка) ---
listen 80 deferred;                 # deferred — отложить принятие соединения
                                    # Улучшает производительность при высоких нагрузках

# --- Fastopen (ускорение TCP) ---
listen 80 fastopen=256;             # fastopen — TCP Fast Open
                                    # Ускоряет установку соединения

# --- Имя сетевого интерфейса ---
listen 80 bind;                     # bind — явно привязаться к порту
                                    # Используется при нескольких интерфейсах

# ============================================
# server_name — ВСЕ ВОЗМОЖНЫЕ ЗНАЧЕНИЯ
# ============================================

# --- Точное совпадение (ВЫСШИЙ ПРИОРИТЕТ) ---
server_name example.com;            # Совпадает только с example.com
server_name example.com www.example.com;  # Несколько имен через пробел

# --- Wildcard (средний приоритет) ---
server_name *.example.com;          # Совпадает с любым поддоменом: blog.example.com, api.example.com
                                    # НЕ совпадает с example.com (нужен поддомен)

server_name .example.com;           # Эквивалентно: example.com *.example.com
                                    # Сокращенная запись

server_name example.*;              # НЕ РАБОТАЕТ! NGINX не поддерживает суффиксные wildcard

# --- Регулярные выражения (низкий приоритет) ---
server_name ~^(www\.)?example\.com$;   # ~ — регистрозависимый
                                    # Совпадает с example.com и www.example.com

server_name ~*^(www\.)?example\.com$;  # ~* — регистронезависимый
                                    # Совпадает с Example.Com, WWW.Example.Com

server_name ~^api\..*\.example\.com$;  # Сложная регулярка

# --- Специальные значения ---
server_name _;                      # _ (подчеркивание) — "невозможное" имя
                                    # Используется с default_server
                                    # Никогда не совпадает с реальным запросом

server_name "";                     # Пустая строка — совпадает с запросами БЕЗ заголовка Host
                                    # Для старых HTTP/1.0 клиентов

server_name localhost;              # Используется для локальной разработки

# --- IP-адрес как имя (не рекомендуется) ---
server_name 192.168.1.100;          # Можно, но лучше не использовать
```

---

## 7. БАЗОВЫЕ ДИРЕКТИВЫ

### ТЕОРИЯ:

| Директива | Контекст | Описание |
|-----------|----------|----------|
| `root` | http, server, location | Корневая папка сайта |
| `index` | http, server, location | Индексные файлы по умолчанию |
| `alias` | location | Замена URI на другой путь |
| `try_files` | server, location | Проверка существования файлов |
| `return` | server, location, if | Редиректы и ответы |
| `add_header` | http, server, location, if | Добавление HTTP-заголовков |

---

### 🚀 ШПАРГАЛКА (с возможными значениями):

```nginx
# ============================================
# root — КОРНЕВАЯ ПАПКА
# ============================================

root /var/www/site;                 # Абсолютный путь
root /home/user/site;               # Путь может быть любым
root /var/www/$host;                # С переменной $host (динамический)

# ВАЖНО: NGINX добавляет URI к пути root
# Запрос: /images/photo.jpg
# root: /var/www/site
# Итог: /var/www/site/images/photo.jpg

# ============================================
# index — ИНДЕКСНЫЕ ФАЙЛЫ
# ============================================

index index.html;                   # Один файл
index index.html index.htm;         # Проверяются по порядку
index index.html index.php index.htm;  # Сначала index.html, потом index.php, потом index.htm

# Если ни один из файлов не найден — NGINX пытается отдать список папок (если autoindex on)

# ============================================
# alias — ЗАМЕНА ПУТИ (отличается от root!)
# ============================================

location /static/ {
    alias /var/www/static/;         # /static/image.jpg → /var/www/static/image.jpg
                                    # alias ЗАМЕНЯЕТ часть URI на свой путь
}

location /images/ {
    root /var/www/site;             # /images/photo.jpg → /var/www/site/images/photo.jpg
                                    # root ДОБАВЛЯЕТ URI к пути
}

# ВАЖНОЕ ПРАВИЛО:
# Если location заканчивается на слеш (/static/), alias тоже должен заканчиваться на слеш!
location /static/ {
    alias /var/www/static/;         # ✅ Правильно
}

location /static/ {
    alias /var/www/static;          # ⚠️ Опасно! Может работать неправильно
}

location /static {
    alias /var/www/static;          # ✅ Правильно (без слеша)
}

# ============================================
# try_files — ПРОВЕРКА СУЩЕСТВОВАНИЯ ФАЙЛОВ
# ============================================

# Синтаксис: try_files файл1 файл2 ... fallback;

location / {
    try_files $uri $uri/ =404;              # 1) точный файл, 2) папка, 3) 404
}

location / {
    try_files $uri $uri/ /index.php?$args;  # Если нет — отдать index.php
}

location / {
    try_files $uri /fallback.html @custom;  # Если нет — перейти в именованный location
}

location @custom {
    proxy_pass http://backend;              # Именованный location (только внутренний)
}

# $uri — путь без параметров
# $uri/ — проверка папки (ищет индексный файл)
# =404 — вернуть 404
# /index.php?$args — отдать index.php с параметрами
# @custom — перейти в именованный location

# ============================================
# return — РЕДИРЕКТЫ И ОТВЕТЫ
# ============================================

# Синтаксис: return код [текст/URL];

return 200 "OK\n";                  # 200 — успешный ответ с текстом
return 301 /new-page;               # 301 — постоянный редирект (SEO)
return 302 /new-page;               # 302 — временный редирект
return 303 /new-page;               # 303 — See Other (для POST → GET)
return 307 /new-page;               # 307 — временный редирект (сохраняет метод)
return 308 /new-page;               # 308 — постоянный редирект (сохраняет метод)
return 404;                         # 404 — Not Found
return 403;                         # 403 — Forbidden
return 500;                         # 500 — Internal Server Error
return 444;                         # 444 — закрыть соединение без ответа (NGINX-specific)
return 301 https://$server_name$request_uri;  # Редирект с сохранением пути

# Коды ответов (основные):
# 2xx — успех: 200, 201, 204
# 3xx — редиректы: 301, 302, 303, 307, 308
# 4xx — ошибки клиента: 400, 401, 403, 404, 429
# 5xx — ошибки сервера: 500, 502, 503, 504

# ============================================
# add_header — ДОБАВЛЕНИЕ HTTP-ЗАГОЛОВКОВ
# ============================================

# Синтаксис: add_header имя значение [always];

add_header X-Frame-Options DENY;    # Защита от кликджекинга
add_header X-Content-Type-Options nosniff;  # Защита от MIME-путаницы
add_header X-XSS-Protection "1; mode=block";  # Защита от XSS

add_header Cache-Control "no-cache, no-store, must-revalidate";  # Отключить кеш
add_header Cache-Control "public, immutable";  # Включить кеш

add_header X-Cache-Status $upstream_cache_status;  # Статус кеша (для отладки)

add_header Content-Type text/plain; # Указать MIME-тип

# always — добавлять заголовок даже для ошибок (4xx, 5xx)
add_header X-Frame-Options DENY always;

# Удалить заголовок (в отличие от add_header)
# proxy_hide_header Server;  # Скрыть заголовок Server
```

---

## 8. ВИРТУАЛЬНЫЕ ХОСТЫ (SERVER BLOCKS)

### ТЕОРИЯ:

Виртуальные хосты позволяют запускать несколько сайтов на одном сервере. NGINX выбирает нужный `server` блок по заголовку `Host` в запросе.

**Почему `sites-available` и `sites-enabled`:**
- **sites-available** — хранилище всех конфигов (как библиотека).
- **sites-enabled** — только активные конфиги (как книги на полке).

Это позволяет:
- Хранить конфиги отключенных сайтов (без потери данных).
- Быстро включать/отключать сайты (удалить/создать симлинк).
- Избежать путаницы (все конфиги в одном месте).

---

### 🚀 ШПАРГАЛКА:

```bash
# ============================================
# СОЗДАНИЕ НОВОГО САЙТА (ПОШАГОВО)
# ============================================

# 1. Создать папку для файлов сайта
sudo mkdir -p /var/www/newsite
# -p — создать все родительские папки, если их нет

# 2. Создать index.html
echo '<h1>Новый сайт</h1><p>Работает на NGINX</p>' | sudo tee /var/www/newsite/index.html

# 3. Создать конфиг в sites-available
sudo tee /etc/nginx/sites-available/newsite.conf > /dev/null << 'EOF'
server {
    listen 80;                          # Слушаем порт 80
    server_name newsite.local;          # Домен сайта

    root /var/www/newsite;              # Корневая папка
    index index.html;                   # Индексный файл

    access_log /var/log/nginx/newsite_access.log;   # Свой лог запросов
    error_log /var/log/nginx/newsite_error.log error;  # Свой лог ошибок

    location / {
        try_files $uri $uri/ =404;      # Дефолтный обработчик
    }
}
EOF

# 4. Активировать сайт (создать симлинк)
sudo ln -s /etc/nginx/sites-available/newsite.conf /etc/nginx/sites-enabled/

# 5. Проверить конфиг и перезагрузить
sudo nginx -t && sudo nginx -s reload

# 6. Добавить домен в /etc/hosts (для локальной разработки)
echo "127.0.0.1 newsite.local" | sudo tee -a /etc/hosts

# ============================================
# ВКЛЮЧЕНИЕ/ОТКЛЮЧЕНИЕ САЙТОВ
# ============================================

# Включить сайт
sudo ln -s /etc/nginx/sites-available/site.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo nginx -s reload

# Отключить сайт (конфиг остается в available)
sudo rm /etc/nginx/sites-enabled/site.conf
sudo nginx -t && sudo nginx -s reload

# Полностью удалить сайт
sudo rm /etc/nginx/sites-available/site.conf
sudo rm /etc/nginx/sites-enabled/site.conf  # если осталась ссылка

# ============================================
# ПРОСМОТР АКТИВНЫХ САЙТОВ
# ============================================

ls -la /etc/nginx/sites-enabled/
# Вывод:
# lrwxrwxrwx 1 root root 35 Jun 23 18:00 example.com.conf -> /etc/nginx/sites-available/example.com.conf
```

---

## 9. ЛОГИРОВАНИЕ

### ТЕОРИЯ:

Логирование в NGINX имеет два уровня:
1. **access_log** — логи всех запросов (кто, что, когда запросил).
2. **error_log** — логи ошибок и предупреждений.

**Уровни error_log (от низшего к высшему):**

| Уровень | Что записывается | Когда использовать |
|---------|------------------|-------------------|
| `debug` | Абсолютно всё (каждая системная операция) | Только для разработки. Опасно для продакшена! |
| `info` | Информационные сообщения | Редко используется |
| `notice` | Важные события, но не ошибки | Для мониторинга SSL-сертификатов |
| `warn` | Предупреждения (может привести к проблеме) | Полезно для раннего обнаружения проблем |
| `error` | **Ошибки** (реальные проблемы) | **Стандарт для продакшена** |
| `crit` | Критические ошибки (сервер может упасть) | Аварийные ситуации |
| `alert` | Требуется немедленное вмешательство | Почти никогда не используется |
| `emerg` | Система в нерабочем состоянии | Почти никогда не используется |

**Иерархия:** Если указан `error`, то пишутся: `error`, `crit`, `alert`, `emerg`. `warn` и ниже — НЕ пишутся.

---

### 🚀 ШПАРГАЛКА:

```nginx
# ============================================
# НАСТРОЙКА ЛОГОВ
# ============================================

access_log /var/log/nginx/site_access.log main buffer=32k flush=5s;
# access_log — лог запросов
# /var/log/nginx/site_access.log — путь к файлу
# main — формат (log_format)
# buffer=32k — буфер 32 КБ (пишем на диск пачками, быстрее)
# flush=5s — принудительно записывать буфер каждые 5 секунд

error_log /var/log/nginx/site_error.log error;
# error_log — лог ошибок
# error — уровень логирования (пишем только error и выше)

access_log off;                     # Отключить логирование

# ============================================
# ФОРМАТЫ ЛОГОВ (log_format)
# ============================================

log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for" '
                'rt=$request_time uct="$upstream_connect_time" '
                'uht="$upstream_header_time" urt="$upstream_response_time"';

log_format json escape=json '{'
    '"time":"$time_local",'
    '"remote_addr":"$remote_addr",'
    '"remote_user":"$remote_user",'
    '"request":"$request",'
    '"status":$status,'
    '"body_bytes_sent":$body_bytes_sent,'
    '"request_time":$request_time,'
    '"http_referrer":"$http_referer",'
    '"http_user_agent":"$http_user_agent",'
    '"http_x_forwarded_for":"$http_x_forwarded_for",'
    '"upstream_addr":"$upstream_addr",'
    '"upstream_response_time":"$upstream_response_time"'
'}';

# ============================================
# ПРОСМОТР ЛОГОВ (команды)
# ============================================

sudo tail -f /var/log/nginx/access.log          # Следить за запросами
sudo tail -f /var/log/nginx/error.log           # Следить за ошибками

sudo tail -n 100 /var/log/nginx/access.log      # Последние 100 строк

sudo grep "404" /var/log/nginx/access.log       # Найти все 404 ошибки
sudo grep "error" /var/log/nginx/error.log      # Найти все ошибки

sudo grep "192.168.1.100" /var/log/nginx/access.log  # Найти по IP

# ============================================
# АНАЛИЗ ЛОГОВ (быстрые команды)
# ============================================

# Топ-10 самых частых IP
sudo awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -10

# Топ-10 самых частых запросов
sudo awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -10

# Топ-10 самых медленных запросов
sudo awk '{print $7, $NF}' /var/log/nginx/access.log | sort -k2 -nr | head -10

# Количество ошибок по статусам
sudo awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

---

## 10. default_server И ПРИОРИТЕТЫ

### ТЕОРИЯ:

**`default_server`** — это флаг, который указывает, какой сервер будет отвечать, если не найдено совпадений по `server_name`.

**Как работает выбор server:**

| Шаг | Проверка | Пример |
|-----|----------|--------|
| 1 | Точное совпадение `server_name` | `example.com` → найден |
| 2 | Wildcard (`*.example.com`) | `blog.example.com` → найден |
| 3 | Регулярное выражение | `~^(www\.)?example\.com$` → найден |
| 4 | `default_server` (если указан) | Используется этот |
| 5 | Первый загруженный (неявный default) | Используется самый первый |

**Почему нужен `default_server`:**
- Без него ваш сайт доступен по IP-адресу → SEO-дубликаты.
- Хакеры сканируют IP-адреса → могут найти ваш сайт.
- Неизвестные домены могут получить доступ к вашему контенту.

---

### 🚀 ШПАРГАЛКА:

```nginx
# ============================================
# БЛОКИРАТОР ЛЕВОГО ТРАФИКА (BEST PRACTICE)
# ============================================

# Создаем самый первый файл: /etc/nginx/conf.d/00-default.conf
server {
    listen 80 default_server;           # default для порта 80
    listen [::]:80 default_server;      # default для IPv6
    listen 443 ssl default_server;      # default для HTTPS
    listen [::]:443 ssl default_server; # default для HTTPS IPv6
    
    server_name _;                      # _ = "невозможное" имя
    
    # Генерация dummy-сертификата:
    # openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    #     -keyout /etc/nginx/ssl/dummy.key \
    #     -out /etc/nginx/ssl/dummy.crt
    ssl_certificate /etc/nginx/ssl/dummy.crt;
    ssl_certificate_key /etc/nginx/ssl/dummy.key;
    
    return 444;                         # Закрыть соединение без ответа
                                        # 444 — NGINX-specific код
    
    access_log off;                     # Не логируем мусорные запросы
    error_log off;                      # Экономит место на диске
}

# ============================================
# ПРИМЕР РАЗНЫХ default_server НА РАЗНЫХ ПОРТАХ
# ============================================

# default для порта 80
server {
    listen 80 default_server;
    server_name _;
    return 301 https://example.com$request_uri;  # Редирект на HTTPS
}

# default для порта 8080
server {
    listen 8080 default_server;
    server_name _;
    return 444;
}
```

---

## 11. ПЕРЕМЕННЫЕ NGINX (ПОЛНЫЙ СПИСОК)

### ТЕОРИЯ:

Переменные в NGINX используются для:
- **Логирования** — в форматах `log_format`.
- **Заголовков** — в `add_header` и `proxy_set_header`.
- **Маршрутизации** — в `try_files`, `rewrite`, `if`.
- **Редиректов** — в `return`.

**Типы переменных:**
1. **Встроенные** — предопределены в NGINX (начинаются с `$`).
2. **Пользовательские** — создаются через `set $var value;`.
3. **Системные** — из заголовков запроса (`$http_*`) и ответа (`$sent_http_*`).

---

### 🚀 ШПАРГАЛКА (полный список с примерами):

```nginx
# ============================================
# ИНФОРМАЦИЯ О КЛИЕНТЕ
# ============================================

$remote_addr                    # IP-адрес клиента
                                # Пример: 192.168.1.100
                                # Всегда IPv4 или IPv6 адрес

$remote_port                    # Порт клиента
                                # Пример: 54321

$remote_user                    # Имя пользователя (basic auth)
                                # Пример: admin

$http_user_agent                # Браузер клиента (User-Agent)
                                # Пример: Mozilla/5.0 (Windows NT 10.0; Win64; x64)

$http_referer                   # Откуда пришел пользователь (Referer)
                                # Пример: https://google.com/

$http_x_forwarded_for           # Реальный IP за прокси (цепочка)
                                # Пример: 10.0.0.1, 192.168.1.1

$http_host                      # Заголовок Host из запроса
                                # Пример: example.com

$http_accept_language           # Язык пользователя
                                # Пример: ru-RU,ru;q=0.9

# ============================================
# ИНФОРМАЦИЯ О ЗАПРОСЕ
# ============================================

$request                        # Полный запрос (метод + URI + протокол)
                                # Пример: GET /index.html HTTP/1.1

$request_uri                    # Полный URI с параметрами (оригинал!)
                                # Пример: /page?param=1
                                # ВНИМАНИЕ! Не нормализуется (опасно для try_files)

$uri                            # URI без параметров, нормализованный
                                # Пример: /page
                                # БЕЗОПАСНЫЙ! Убирает ../ и декодирует %20

$document_uri                   # Синоним $uri
                                # Пример: /page

$args                           # Параметры запроса (без ?)
                                # Пример: param=1

$query_string                   # Синоним $args
                                # Пример: param=1

$request_method                 # Метод запроса
                                # Возможные значения: GET, POST, PUT, DELETE, HEAD, OPTIONS, PATCH

$request_length                 # Размер запроса в байтах
                                # Пример: 1234

$request_time                   # Время обработки запроса (NGINX) в секундах
                                # Пример: 0.123 (123 миллисекунды)

$scheme                         # Протокол
                                # Возможные значения: http, https

$host                           # Домен из заголовка Host
                                # Пример: example.com

$server_protocol                # Протокол запроса
                                # Пример: HTTP/1.1, HTTP/2

$proxy_protocol_addr            # IP из PROXY-протокола (если включен)
                                # Используется за HAProxy, AWS ELB

# ============================================
# ИНФОРМАЦИЯ ОТВЕТА
# ============================================

$status                         # Код ответа
                                # Возможные значения: 200, 301, 302, 403, 404, 500, 502, 503, 504

$body_bytes_sent                # Размер ответа в байтах (без заголовков)
                                # Пример: 1234

$bytes_sent                     # Размер ответа в байтах (включая заголовки)
                                # Пример: 5678

$sent_http_*                    # Любой заголовок ответа
                                # Пример: $sent_http_content_type → text/html

$server_port                    # Порт, на который пришел запрос
                                # Пример: 80, 443, 8080

# ============================================
# ИНФОРМАЦИЯ О БЕКЕНДЕ (upstream)
# ============================================

$upstream_addr                  # Адрес бекенда, который обработал запрос
                                # Пример: 10.0.0.1:8080

$upstream_connect_time          # Время подключения к бекенду в секундах
                                # Пример: 0.005

$upstream_header_time           # Время получения заголовков от бекенда
                                # Пример: 0.100

$upstream_response_time         # Время получения полного ответа от бекенда
                                # Пример: 0.150

$upstream_cache_status          # Статус кеша
                                # Возможные значения: HIT, MISS, EXPIRED, UPDATING, BYPASS

$upstream_response_length       # Размер ответа бекенда в байтах
                                # Пример: 1234

# ============================================
# ИНФОРМАЦИЯ О СЕРВЕРЕ
# ============================================

$server_name                    # Имя сервера из конфига
                                # Пример: example.com

$document_root                  # Корневая папка текущего запроса
                                # Пример: /var/www/site

$request_filename               # Полный путь к файлу на диске
                                # Пример: /var/www/site/index.html

$realpath_root                  # Реальный путь (без симлинков) к файлу
                                # Пример: /var/www/site

$pid                            # PID мастер-процесса NGINX
                                # Пример: 1234

$msec                           # Текущее время в секундах с микросекундами
                                # Пример: 1687544400.123

$connection                     # Номер соединения
                                # Пример: 12345

$connection_requests            # Количество запросов в этом соединении
                                # Пример: 3

# ============================================
# HTTP-ЗАГОЛОВКИ ЗАПРОСА (все, что прислал клиент)
# ============================================

# Любой заголовок доступен как $http_<имя_заголовка_в_нижнем_регистре>

$http_accept                    # Accept: text/html,application/xhtml+xml
$http_accept_encoding           # Accept-Encoding: gzip, deflate, br
$http_cache_control             # Cache-Control: no-cache
$http_cookie                    # Cookie: sessionid=123; user=admin
$http_referer                   # Referer: https://google.com/
$http_upgrade                   # Upgrade: websocket (для WebSocket)

# ============================================
# HTTP-ЗАГОЛОВКИ ОТВЕТА (устанавливаемые NGINX)
# ============================================

# Можно читать заголовки, которые установил NGINX
# $sent_http_<имя_заголовка_в_нижнем_регистре>

$sent_http_content_type         # Content-Type: text/html
$sent_http_content_length       # Content-Length: 1234
$sent_http_cache_control        # Cache-Control: public, immutable

# ============================================
# ПРИМЕРЫ ИСПОЛЬЗОВАНИЯ
# ============================================

# В заголовках (для отладки)
add_header X-Request-URI $request_uri;          # Показать, что запросил клиент
add_header X-Real-IP $remote_addr;              # Показать реальный IP клиента
add_header X-Cache-Status $upstream_cache_status;  # Показать статус кеша

# В логах
log_format main '$remote_addr "$request" $status rt=$request_time';

# В редиректах (сохранить параметры)
return 301 https://$server_name$request_uri;    # Редирект с сохранением полного пути

# В условных проверках (if)
if ($request_method = POST) {
    proxy_pass http://backend_post;
}

if ($http_user_agent ~* "bot") {
    return 403;                                 # Блокировать ботов
}

# В кешировании (ключ кеша)
proxy_cache_key "$scheme$host$request_uri";     # Ключ по умолчанию

# ============================================
# ПОЛЬЗОВАТЕЛЬСКИЕ ПЕРЕМЕННЫЕ
# ============================================

set $my_var "Hello";                    # Создать переменную
set $cache_key "$host$request_uri";     # Составить ключ
set $loggable 1;                        # Флаг для условного логирования

# Использование в условиях
if ($my_var = "Hello") {
    return 200 "World";
}
```

---

## 12. МОДУЛЬ 3: LOCATION И МАРШРУТИЗАЦИЯ

### ТЕОРИЯ:

`location` — это сердце маршрутизации в NGINX. Он определяет, как обрабатывать конкретные URI.

**Алгоритм выбора location (САМОЕ ВАЖНОЕ!):**

| Шаг | Что делает | Пример |
|-----|------------|--------|
| 1 | Ищет `=` (точное совпадение) | `location = /` |
| 2 | Ищет все префиксные совпадения | `location /`, `location /blog/` |
| 3 | Запоминает самое длинное префиксное | `/blog/post/123` → длиннее, чем `/blog/` |
| 4 | Если есть `^~` → использует его (СТОП) | `location ^~ /static/` |
| 5 | Проверяет регулярки в порядке объявления | `~ \.php$` → первое совпадение |
| 6 | Использует самое длинное префиксное | Fallback |

---

### 🚀 ШПАРГАЛКА:

```nginx
# ============================================
# ВСЕ ТИПЫ LOCATION (по приоритету)
# ============================================

# 1. Точное совпадение (ВЫСШИЙ ПРИОРИТЕТ)
# Совпадает ТОЛЬКО с указанным URI
# Скорость: максимальная (быстрее всего)
location = / {
    return 200 "Главная страница";
}

location = /admin {
    return 200 "Только /admin, не /admin/";
}

# 2. Приоритетное префиксное (БЛОКИРУЕТ РЕГУЛЯРКИ!)
# Если совпало — регулярки НЕ проверяются (ускорение)
# Используется для статики и папок
location ^~ /static/ {
    root /var/www/site;
    expires 30d;
    access_log off;
}

location ^~ /uploads/ {
    root /var/www/site;
    expires 1d;
    # Запрещаем выполнять PHP в папке uploads
    location ~ \.php$ {
        return 403;
    }
}

# 3. Регулярные выражения (проверяются в порядке объявления)
# ~ — регистрозависимый (чувствителен к регистру)
# ~* — регистронезависимый (НЕ чувствителен к регистру)

location ~ \.php$ {                     # Только .php (маленькие буквы)
    fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
}

location ~* \.(jpg|jpeg|png|gif|ico|svg|css|js)$ {  # Любой регистр
    expires 30d;
    add_header Cache-Control "public, immutable";
    access_log off;
}

# 4. Префиксное совпадение (НИЗШИЙ ПРИОРИТЕТ)
# Совпадает с началом URI
location / {
    try_files $uri $uri/ =404;          # Дефолтный обработчик
}

location /blog/ {
    try_files $uri $uri/ /blog/index.php?$args;
}

# ============================================
# ПРИМЕР ПОЛНОГО КОНФИГА С LOCATION
# ============================================

server {
    root /var/www/site;
    index index.html index.php;

    # 1. Точное совпадение для главной
    location = / {
        try_files $uri $uri/ =404;
    }

    # 2. Точное совпадение для favicon
    location = /favicon.ico {
        expires 30d;
        access_log off;
        log_not_found off;
    }

    # 3. Приоритетное префиксное для статики
    location ^~ /static/ {
        root /var/www/site;
        expires 30d;
        access_log off;
    }

    # 4. Приоритетное префиксное для загрузок
    location ^~ /uploads/ {
        root /var/www/site;
        expires 1d;
        # Запрет выполнения PHP в uploads
        location ~ \.php$ {
            return 403;
        }
    }

    # 5. Регулярка для картинок (регистронезависимая)
    location ~* \.(jpg|jpeg|png|gif|ico|svg)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # 6. Регулярка для CSS/JS (регистронезависимая)
    location ~* \.(css|js)$ {
        expires 7d;
        add_header Cache-Control "public, immutable";
        access_log off;
        gzip on;
    }

    # 7. Регулярка для PHP (регистрозависимая)
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }

    # 8. Префиксное (дефолтный)
    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # 9. Именованный location (для fallback)
    location @fallback {
        internal;
        proxy_pass http://backend;
    }
}
```

---

## 13. МОДУЛЬ 4: proxy_pass И БАЛАНСИРОВКА

### ТЕОРИЯ:

**Проксирование** — это когда NGINX принимает запрос и перенаправляет его на другой сервер (бекенд).

**Зачем:**
1. Скрыть бекенд от внешнего мира (безопасность)
2. Распределять нагрузку между несколькими бекендами
3. Кешировать ответы бекенда
4. Терминировать SSL (HTTPS → HTTP)

**Алгоритмы балансировки:**

| Алгоритм | Описание | Когда использовать |
|----------|----------|-------------------|
| **Round-Robin** (по умолчанию) | Запросы по очереди | Стандартный случай |
| **Least Connections** | На сервер с наименьшими соединениями | Запросы разной длительности |
| **IP Hash** | По IP-адресу (сессии) | Когда нужны сессии на сервере |
| **Hash** | По любому ключу (URI, кука) | Для кеширования |
| **Random** | Случайный выбор | Для больших распределенных систем |

---

### 🚀 ШПАРГАЛКА:

```nginx
# ============================================
# БАЗОВОЕ ПРОКСИРОВАНИЕ
# ============================================

location / {
    proxy_pass http://backend:8080;     # Все запросы на бекенд
    
    # ============================================
    # ОБЯЗАТЕЛЬНЫЕ ЗАГОЛОВКИ
    # ============================================
    proxy_set_header Host $host;                # Оригинальный домен
    proxy_set_header X-Real-IP $remote_addr;    # Реальный IP клиента
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  # Цепочка прокси
    proxy_set_header X-Forwarded-Proto $scheme;  # Протокол (http/https)
    
    # ============================================
    # ТАЙМАУТЫ (все значения в секундах)
    # ============================================
    proxy_connect_timeout 5s;           # Время на установку соединения с бекендом
                                        # По умолчанию: 60s
                                        # Для медленных бекендов: 30-60s
    
    proxy_read_timeout 60s;             # Время на получение данных от бекенда
                                        # По умолчанию: 60s
                                        # Для долгих запросов (API): 120-300s
    
    proxy_send_timeout 60s;             # Время на отправку запроса бекенду
                                        # По умолчанию: 60s
    
    # ============================================
    # БУФЕРИЗАЦИЯ
    # ============================================
    proxy_buffering on;                 # Включить буферизацию (по умолчанию on)
                                        # off — для больших файлов, стриминга
    
    proxy_buffer_size 4k;               # Размер буфера для заголовков
    proxy_buffers 8 4k;                 # Количество и размер буферов
    proxy_busy_buffers_size 8k;         # Максимальный размер занятых буферов
}

# ============================================
# ПРАВИЛА proxy_pass (со слешем и без)
# ============================================

# БЕЗ слеша — URI добавляется к URL бекенда
location /api/ {
    proxy_pass http://backend:8080;     # /api/users → /api/users
}

# СО слешем — URI ЗАМЕНЯЕТСЯ
location /api/ {
    proxy_pass http://backend:8080/;    # /api/users → /users
}

# Без слеша в location — работает по-другому
location /api {
    proxy_pass http://backend:8080/;    # /api/users → //users (ОШИБКА!)
}

# ============================================
# UPSTREAM (БАЛАНСИРОВКА) — ВСЕ ВОЗМОЖНЫЕ ПАРАМЕТРЫ
# ============================================

upstream backend {
    # --- Алгоритмы балансировки (выберите один) ---
    # round-robin;          # По умолчанию (по очереди)
    # least_conn;           # Наименьшее количество соединений
    # ip_hash;              # По IP-адресу (сессии)
    # hash $request_uri;    # По URI (для кеширования)
    # random;               # Случайный выбор
    
    least_conn;                     # Алгоритм: least connections
    
    # --- Сервера с параметрами ---
    server 10.0.0.1:8080 weight=2 max_fails=3 fail_timeout=30s;
    # server — бекенд-сервер
    # weight=2 — вес (получает в 2 раза больше запросов)
    #            По умолчанию: 1
    #            Диапазон: 1-100
    
    # max_fails=3 — после 3 неудачных попыток пометить как "мертвый"
    #                По умолчанию: 1
    #                Диапазон: 1-10
    
    # fail_timeout=30s — на 30 секунд исключить из балансировки
    #                    По умолчанию: 10s
    #                    Диапазон: 1s - 600s
    
    server 10.0.0.2:8080 weight=1 max_fails=3 fail_timeout=30s;
    
    server 10.0.0.3:8080 backup;    # backup — резервный (включается, если основные упали)
    
    server 10.0.0.4:8080 down;      # down — помечать как недоступный (не использовать)
    
    server unix:/var/run/backend.sock;  # Unix-сокет (быстрее TCP)
    
    # --- Keepalive (переиспользование соединений) ---
    keepalive 32;                   # Держать 32 соединения открытыми
                                    # По умолчанию: 0 (отключено)
                                    # Рекомендация: 16-64
    
    keepalive_requests 1000;        # Максимум запросов на одно соединение
                                    # По умолчанию: 1000
    
    keepalive_timeout 60s;          # Закрывать соединение через 60 секунд
                                    # По умолчанию: 60s
}

# Использование upstream
server {
    location / {
        proxy_pass http://backend;  # Используем upstream
        proxy_http_version 1.1;     # Обязательно для keepalive
        proxy_set_header Connection "";  # Отключаем заголовок Connection
    }
}

# ============================================
# WEBSOCKET (полная настройка)
# ============================================

location /ws/ {
    proxy_pass http://backend:8080;
    
    # Обязательно для WebSocket
    proxy_http_version 1.1;                  # HTTP 1.1 (поддерживает upgrade)
    proxy_set_header Upgrade $http_upgrade;  # Превращаем HTTP в WebSocket
    proxy_set_header Connection "upgrade";   # Говорим, что это апгрейд
    
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    
    # Таймауты для долгих соединений
    proxy_read_timeout 3600s;                # 1 час
    proxy_send_timeout 3600s;                # 1 час
    proxy_connect_timeout 60s;               # 60 секунд
}
```

---

## 14. МОДУЛЬ 5: БЕЗОПАСНОСТЬ И SSL

### ТЕОРИЯ:

**SSL/TLS** — протоколы шифрования для защиты данных между клиентом и сервером.

**Типы сертификатов:**
| Тип | Доверие браузера | Стоимость | Использование |
|-----|------------------|-----------|---------------|
| **Самоподписанный** | ❌ Нет (ошибка) | Бесплатно | Только тесты |
| **Let's Encrypt** | ✅ Да | Бесплатно | Продакшен (рекомендуется) |
| **DV (Domain Validated)** | ✅ Да | Платно | Стандартные сайты |
| **OV (Organization Validated)** | ✅ Да | Дорого | Бизнес-сайты |
| **EV (Extended Validation)** | ✅ Да | Очень дорого | Банки, госсайты |

**Rate Limiting** — защита от DDoS и брутфорса.
**Базовая аутентификация** — простая защита паролем.

---

### 🚀 ШПАРГАЛКА:

```nginx
# ============================================
# ГЕНЕРАЦИЯ САМОПОДПИСАННОГО СЕРТИФИКАТА (для тестов)
# ============================================

# Создаем папку
sudo mkdir -p /etc/nginx/ssl

# Генерируем ключ и сертификат
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/example.key \
    -out /etc/nginx/ssl/example.crt \
    -subj "/C=RU/ST=Moscow/L=Moscow/O=MyCompany/CN=example.com"

# Параметры:
# -x509 — создать самоподписанный сертификат
# -nodes — без пароля на ключ
# -days 365 — срок действия (1 год)
# -newkey rsa:2048 — алгоритм и длина ключа
# -keyout — куда сохранить ключ
# -out — куда сохранить сертификат
# -subj — информация о владельце
#   C=Страна, ST=Область, L=Город, O=Организация, CN=Домен

# ============================================
# ПОЛНАЯ НАСТРОЙКА SSL/HTTPS
# ============================================

server {
    listen 443 ssl http2;                   # HTTPS + HTTP/2
    listen [::]:443 ssl http2;              # IPv6
    server_name example.com;

    # Сертификаты (Let's Encrypt)
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    # fullchain.pem — сертификат + промежуточные (CA-цепочка)
    
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    # privkey.pem — приватный ключ (хранить в секрете!)

    # ============================================
    # SSL ПРОТОКОЛЫ
    # ============================================
    ssl_protocols TLSv1.2 TLSv1.3;          # Только современные протоколы
    # TLSv1, TLSv1.1 — отключаем (небезопасны)
    
    # ============================================
    # SSL ШИФРЫ (Ciphers)
    # ============================================
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    # ECDHE — современные алгоритмы
    # GCM — блочное шифрование
    # CHACHA20 — для мобильных устройств
    
    ssl_prefer_server_ciphers off;          # Отключаем предпочтение серверных шифров
    # on — сервер выбирает шифр (для старых клиентов)
    # off — клиент выбирает шифр (современный стандарт)

    # ============================================
    # SSL СЕССИИ (ускорение повторных подключений)
    # ============================================
    ssl_session_cache shared:SSL:10m;       # Кеш сессий в памяти
    # shared:SSL:10m — 10 МБ для сессий (~40000 сессий)
    # off — отключено (медленнее)
    
    ssl_session_timeout 10m;                # Время жизни сессии
    # 10m — 10 минут (баланс скорости и безопасности)
    # 1h — 1 час (быстрее, но менее безопасно)
    
    ssl_session_tickets off;                # Отключаем (для безопасности)
    # on — включено (ускоряет, но менее безопасно)
    # off — отключено (рекомендуется)

    # ============================================
    # OCSP STAPLING (ускорение проверки сертификатов)
    # ============================================
    ssl_stapling on;                        # Включить OCSP Stapling
    ssl_stapling_verify on;                 # Проверять ответы OCSP
    ssl_trusted_certificate /etc/nginx/ssl/example.crt;  # CA-цепочка
    resolver 8.8.8.8 1.1.1.1 valid=300s;    # DNS-резолверы
    resolver_timeout 5s;                    # Таймаут резолвера
}

# ============================================
# HTTP → HTTPS (РЕДИРЕКТ)
# ============================================

server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;  # Постоянный редирект
}

# ============================================
# HSTS (HTTP Strict Transport Security)
# ============================================

add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
# max-age=31536000 — 1 год (обязательно!)
# includeSubDomains — применяется ко всем поддоменам
# preload — добавляет в список HSTS preload (для браузеров)

# ============================================
# ЗАГОЛОВКИ БЕЗОПАСНОСТИ (обязательный набор)
# ============================================

add_header X-Frame-Options DENY always;     # Защита от кликджекинга
add_header X-Content-Type-Options nosniff always;  # Защита от MIME-путаницы
add_header X-XSS-Protection "1; mode=block" always;  # Защита от XSS
add_header Referrer-Policy "strict-origin-when-cross-origin" always;  # Контроль реферера

# ============================================
# RATE LIMITING (защита от DDoS и брутфорса)
# ============================================

http {
    # Зона лимитов по IP (10 МБ памяти, 50 запросов/сек)
    limit_req_zone $binary_remote_addr zone=ddoslimit:10m rate=50r/s;
    # $binary_remote_addr — IP-адрес (в бинарном виде, экономит память)
    # zone=ddoslimit:10m — 10 МБ для хранения счетчиков
    # rate=50r/s — 50 запросов в секунду
    
    # Зона лимитов по IP + URI (для поиска)
    limit_req_zone $binary_remote_addr$uri zone=searchlimit:10m rate=1r/s;
    
    # Зона лимитов подключений
    limit_conn_zone $binary_remote_addr zone=connlimit:10m;
    
    server {
        location / {
            # Лимит запросов
            limit_req zone=ddoslimit burst=30 nodelay;
            # burst=30 — разрешить 30 запросов "в буфер" (без 503)
            # nodelay — отдавать буферные запросы мгновенно
            limit_req_status 503;               # Код ответа при превышении
            
            # Лимит подключений
            limit_conn connlimit 20;            # Не более 20 соединений с одного IP
            limit_conn_status 503;
        }
        
        # Строгий лимит для логина (защита от брутфорса)
        location /login {
            limit_req zone=loginlimit burst=2 nodelay;
            limit_req_status 429;  # Too Many Requests
        }
    }
}

# ============================================
# БАЗОВАЯ АУТЕНТИФИКАЦИЯ (пароль на сайт)
# ============================================

# Создать файл с паролем (первый пользователь)
sudo htpasswd -c /etc/nginx/.htpasswd admin
# -c — создать файл (только для первого пользователя)

# Добавить еще пользователя
sudo htpasswd /etc/nginx/.htpasswd user2

# В конфиге
location /admin/ {
    auth_basic "Restricted Area";           # Текст, который увидит пользователь
    auth_basic_user_file /etc/nginx/.htpasswd;  # Файл с паролями
    
    proxy_pass http://backend;
}

# ============================================
# ОГРАНИЧЕНИЕ ДОСТУПА ПО IP
# ============================================

location /admin/ {
    allow 192.168.1.0/24;           # Разрешить локальную сеть
    allow 10.0.0.1;                 # Разрешить конкретный IP
    allow 127.0.0.1;                # Разрешить localhost
    deny all;                       # Запретить все остальные
    
    proxy_pass http://backend;
}

# ============================================
# ЗАЩИТА ОТ SQL-ИНЪЕКЦИЙ И XSS
# ============================================

# Блокируем SQL-инъекции и XSS
if ($request_uri ~* "(\%27)|(\')|(\-\-)|(\%23)|(#)|(<script)") {
    return 403;
}

# Блокируем попытки подняться вверх по папкам
if ($request_uri ~* "\.\./") {
    return 403;
}

# Блокируем доступ к системным файлам
if ($request_uri ~* "(/etc/passwd|/proc/self/environ)") {
    return 403;
}
```

---

## 15. МОДУЛЬ 6: ОПТИМИЗАЦИЯ И КЕШИРОВАНИЕ

### ТЕОРИЯ:

**Кеширование** — это сохранение часто запрашиваемых данных для быстрой отдачи.

**Типы кеширования:**
| Тип | Где хранится | Зачем | Пример |
|-----|--------------|-------|--------|
| **Браузерный кеш** | На стороне клиента | Не загружать статику повторно | Картинки, CSS, JS |
| **Прокси-кеш** | На стороне NGINX | Не нагружать бекенд | API, динамические страницы |
| **Микро-кеш** | На стороне NGINX | Сглаживать пиковые нагрузки | Главная страница (1-2 секунды) |
| **SSL-кеш** | В NGINX | Ускорять SSL-рукопожатия | Сессии SSL |

**Gzip** — сжатие данных на лету, уменьшает размер в 3-5 раз.

---

### 🚀 ШПАРГАЛКА:

```nginx
# ============================================
# БРАУЗЕРНЫЙ КЕШ (статики)
# ============================================

location ~* \.(jpg|jpeg|png|gif|ico|svg|webp|woff|woff2|ttf|eot|css|js)$ {
    expires 30d;                                # Кеш на 30 дней
    # Возможные значения: 1s, 1m, 1h, 1d, 1w, 1M, 1y
    # or: off (отключить), epoch (1 января 1970)
    
    add_header Cache-Control "public, immutable";   # Инструкции кеша
    # public — кешировать можно где угодно (браузер, прокси)
    # private — только в браузере пользователя
    # immutable — никогда не менять (если изменилось — новое имя файла)
    # no-cache — проверять наличие новой версии (условно)
    # no-store — вообще не кешировать
    # must-revalidate — проверять при каждом запросе
    
    access_log off;                             # Не логируем статику
    gzip_static on;                             # Отдавать сжатые версии (.gz)
}

# ============================================
# ПРОКСИ-КЕШ (кеш ответов бекенда)
# ============================================

http {
    # Объявляем зону кеша
    proxy_cache_path /var/cache/nginx levels=1:2 
                     keys_zone=mycache:10m 
                     max_size=1g 
                     inactive=60m 
                     use_temp_path=off
                     manager_files=100
                     manager_threshold=200ms;
    
    # Параметры proxy_cache_path:
    # /var/cache/nginx — путь к папке кеша
    # levels=1:2 — иерархия папок (1 папка первого уровня, 2 — второго)
    # keys_zone=mycache:10m — имя зоны и размер в памяти
    # max_size=1g — максимальный размер кеша на диске
    # inactive=60m — удалять неиспользуемые файлы через 60 минут
    # use_temp_path=off — хранить сразу в финальной папке (быстрее)
    # manager_files=100 — обрабатывать 100 файлов за цикл очистки
    # manager_threshold=200ms — максимальное время цикла очистки
    
    server {
        location / {
            proxy_cache mycache;                # Включаем кеш
            
            # ============================================
            # КЕШИРОВАНИЕ ПО КОДАМ ОТВЕТА
            # ============================================
            proxy_cache_valid 200 302 10m;      # 200 и 302 → 10 минут
            proxy_cache_valid 404 1m;           # 404 → 1 минута
            proxy_cache_valid 500 0s;           # 500 → не кешировать
            proxy_cache_valid any 5m;           # Все остальные → 5 минут
            
            # ============================================
            # ИСПОЛЬЗОВАНИЕ ПРОСРОЧЕННОГО КЕША
            # ============================================
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503;
            # Если бекенд упал или ошибка — отдаем просроченный кеш
            
            # ============================================
            # ОБНОВЛЕНИЕ КЕША В ФОНЕ
            # ============================================
            proxy_cache_background_update on;   # Обновлять кеш в фоне
            proxy_cache_lock on;                # Блокировка (чтобы 1 запрос обновлял)
            proxy_cache_lock_timeout 10s;       # Максимальное время блокировки
            
            # ============================================
            # ЗАГОЛОВОК О СОСТОЯНИИ КЕША
            # ============================================
            add_header X-Cache-Status $upstream_cache_status;
            # Значения: HIT, MISS, EXPIRED, UPDATING, BYPASS
            
            proxy_pass http://backend;
        }
    }
}

# ============================================
# МИКРО-КЕШ (1-2 секунды для высоких нагрузок)
# ============================================

# Используется для сглаживания пиковых нагрузок
# Если 1000 пользователей одновременно заходят — бекенд обрабатывает 1 раз

proxy_cache_path /var/cache/nginx-micro levels=1:2 
                 keys_zone=microcache:10m 
                 max_size=100m 
                 inactive=5m 
                 use_temp_path=off;

location / {
    proxy_cache microcache;
    proxy_cache_valid 200 1s;          # Всего 1 секунда!
    proxy_cache_use_stale error timeout updating;
    proxy_cache_background_update on;
    add_header X-Micro-Cache $upstream_cache_status;
    proxy_pass http://backend;
}

# ============================================
# GZIP СЖАТИЕ (уменьшает размер в 3-5 раз)
# ============================================

gzip on;                                # Включить сжатие
gzip_comp_level 6;                      # Уровень сжатия (1-9)
                                        # 1 — быстро, слабое сжатие
                                        # 5-6 — оптимальный баланс
                                        # 9 — медленно, максимальное сжатие

gzip_min_length 1000;                   # Минимальный размер файла для сжатия
                                        # Меньше 1000 байт — не сжимать

gzip_vary on;                           # Добавлять Vary: Accept-Encoding
                                        # Для прокси-серверов

gzip_types text/plain text/css text/xml text/javascript 
           application/javascript application/json 
           application/xml application/rss+xml image/svg+xml;
# Типы файлов для сжатия

gzip_disable "msie6";                   # Отключаем для старых браузеров

gzip_http_version 1.1;                  # Минимальная версия HTTP

# ============================================
# ОПТИМИЗАЦИЯ СОЕДИНЕНИЙ
# ============================================

sendfile on;                            # Включить быструю отдачу файлов
sendfile_max_chunk 1m;                  # Максимальный чанк за раз

tcp_nopush on;                          # Отправлять пакеты максимального размера
tcp_nodelay on;                         # Отключать алгоритм Нейгла (меньше задержка)

keepalive_timeout 65s;                  # Закрывать соединение через 65 секунд
keepalive_requests 1000;                # Максимум запросов в одном соединении

# ============================================
# ОПТИМИЗАЦИЯ БУФЕРОВ
# ============================================

client_header_buffer_size 1k;           # Буфер для заголовков запроса
large_client_header_buffers 4 8k;       # Большие буферы для длинных заголовков

client_body_buffer_size 16k;            # Буфер для тела запроса
client_body_timeout 5s;                 # Таймаут на чтение тела
client_header_timeout 5s;               # Таймаут на чтение заголовков

proxy_buffering on;                     # Включить буферизацию прокси
proxy_buffer_size 4k;                   # Буфер для заголовков прокси
proxy_buffers 8 4k;                     # Количество и размер буферов
proxy_busy_buffers_size 8k;             # Максимальный размер занятых буферов
```

---

## 16. МОДУЛЬ 7: МОНИТОРИНГ И ОТЛАДКА

### ТЕОРИЯ:

**Мониторинг** — это наблюдение за состоянием сервера в реальном времени.

**Метрики NGINX:**
| Метрика | Что показывает | Норма |
|---------|----------------|-------|
| Active connections | Открытые соединения | Зависит от нагрузки |
| Reading | Читающие заголовки | Обычно 0-10 |
| Writing | Отправляющие ответ | Обычно 0-10 |
| Waiting | В ожидании (keepalive) | Может быть большим |
| Requests per second | Запросов в секунду | Зависит от нагрузки |

---

### 🚀 ШПАРГАЛКА:

```nginx
# ============================================
# СТАТУСНАЯ СТРАНИЦА (stub_status)
# ============================================

server {
    listen 127.0.0.1:8080;                  # Только локальный доступ
    server_name localhost;

    location /nginx_status {
        stub_status on;                      # Включаем статусную страницу
        allow 127.0.0.1;                     # Разрешаем localhost
        allow 192.168.1.0/24;                # Разрешаем локальную сеть
        deny all;                            # Запрещаем всех остальных
        access_log off;                      # Не логируем запросы к статусу
    }
}

# Проверка:
curl http://127.0.0.1:8080/nginx_status

# Вывод:
# Active connections: 125 
# server accepts handled requests
#  1254 1254 5432 
# Reading: 0 Writing: 2 Waiting: 123

# Расшифровка:
# Active connections — открытые соединения (все)
# accepts — всего принятых соединений за всё время
# handled — всего обработанных соединений (обычно = accepts)
# requests — всего обработанных запросов за всё время
# Reading — соединения, читающие заголовки запроса
# Writing — соединения, отправляющие ответ
# Waiting — соединения в ожидании (keepalive)

# ============================================
# ПРОМЕТЕУС-ЭКСПОРТЕР (продвинутый мониторинг)
# ============================================

# Установка:
wget https://github.com/nginxinc/nginx-prometheus-exporter/releases/latest/download/nginx-prometheus-exporter_0.10.0_linux_amd64.tar.gz
tar -xzf nginx-prometheus-exporter_0.10.0_linux_amd64.tar.gz
./nginx-prometheus-exporter -nginx.scrape-uri http://127.0.0.1:8080/nginx_status &

# Проверка метрик:
curl http://127.0.0.1:9113/metrics

# Ключевые метрики:
# nginx_connections_active 125
# nginx_connections_reading 0
# nginx_connections_writing 2
# nginx_http_requests_total 5432

# ============================================
# ПРОВЕРКА КОНФИГА (отладка)
# ============================================

sudo nginx -t                            # Проверка синтаксиса
sudo nginx -T                            # Полная конфигурация (все includes)

# Найти конкретную директиву
sudo nginx -T | grep -A 5 "server_name"

# Найти конкретный server
sudo nginx -T | grep -A 20 "server {"

# Проверить порядок загрузки
sudo nginx -T 2>/dev/null | grep -E "^# configuration file"

# ============================================
# ОТЛАДКА ЧЕРЕЗ CURL
# ============================================

# Только заголовки ответа
curl -I https://example.com/

# Все детали запроса-ответа
curl -v https://example.com/

# Подставить конкретный Host
curl -H "Host: example.com" http://127.0.0.1/

# Следовать за редиректами
curl -L https://example.com/

# Проверить SSL-сертификат
openssl s_client -connect example.com:443 -tls1_2

# Проверить SSL без проверки сертификата
curl -k https://example.com/

# Проверить кеш
curl -I https://example.com/ | grep X-Cache

# Принудительное обновление (пропуск кеша)
curl -H "Cache-Control: no-cache" https://example.com/

# Проверить ответ с таймингом
curl -o /dev/null -s -w "Time: %{time_total}s\n" https://example.com/

# ============================================
# ТЕСТИРОВАНИЕ ПРОИЗВОДИТЕЛЬНОСТИ
# ============================================

# Apache Bench (ab)
sudo apt install apache2-utils -y

# 1000 запросов, 10 одновременных
ab -n 1000 -c 10 https://example.com/

# С сохранением куки
ab -n 1000 -c 10 -C "sessionid=123" https://example.com/

# POST-запросы с данными
ab -n 100 -c 5 -p data.json -T application/json https://example.com/api

# Ключевые результаты:
# Requests per second: 426.42 [#/sec]  ← главный показатель
# Time per request: 23.45 [ms]         ← среднее время ответа

# ============================================
# SYSTEMD ЛОГИ
# ============================================

sudo systemctl status nginx              # Статус сервиса
sudo journalctl -u nginx -f              # Логи в реальном времени
sudo journalctl -u nginx --since "1 hour ago"  # За последний час
sudo journalctl -u nginx --since "2024-01-01"  # С указанной даты

# ============================================
# АНАЛИЗ ЛОГОВ (быстрые команды)
# ============================================

# Топ-10 IP
sudo awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -10

# Топ-10 самых частых запросов
sudo awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -10

# Топ-10 самых медленных запросов
sudo awk '{print $7, $NF}' /var/log/nginx/access.log | sort -k2 -nr | head -10

# Количество ошибок по статусам
sudo awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr

# Количество запросов в минуту
sudo awk '{print substr($4,2,16)}' /var/log/nginx/access.log | sort | uniq -c

# Запросы с 500 ошибками
sudo grep " 500 " /var/log/nginx/access.log

# Медленные запросы (> 1 секунды)
sudo awk '$NF > 1 {print $0}' /var/log/nginx/access.log
```

---

## 17. ЧЕК-ЛИСТ: 15 ОБЯЗАТЕЛЬНЫХ КОМПОНЕНТОВ

### 🚀 ШПАРГАЛКА:

| № | Компонент | Конфиг | Почему |
|---|-----------|--------|--------|
| 1 | **SSL/HTTPS** | `listen 443 ssl http2;` | Безопасность, доверие, SEO |
| 2 | **HSTS** | `add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;` | Принудительное шифрование |
| 3 | **Заголовки безопасности** | `X-Frame-Options`, `X-Content-Type-Options`, `X-XSS-Protection` | Защита от XSS, clickjacking |
| 4 | **Rate Limiting** | `limit_req_zone`, `limit_conn_zone` | Защита от DDoS, брутфорса |
| 5 | **Ограничение размера тела** | `client_max_body_size 10M;` | Защита от переполнения диска |
| 6 | **Скрытие версии** | `server_tokens off;` | Усложняет жизнь хакерам |
| 7 | **Кеширование статики** | `expires 30d; add_header Cache-Control "public, immutable";` | Ускорение, снижение нагрузки |
| 8 | **Gzip сжатие** | `gzip on; gzip_comp_level 6;` | Экономия трафика, ускорение |
| 9 | **Индивидуальные логи** | `access_log /var/log/nginx/site_access.log; error_log /var/log/nginx/site_error.log;` | Возможность отладки |
| 10 | **Логи с временем ответа** | `log_format main ... rt=$request_time ...` | Поиск медленных запросов |
| 11 | **`try_files`** | `try_files $uri $uri/ =404;` | Дефолтная маршрутизация |
| 12 | **Защита от SQL-инъекций** | `if ($request_uri ~* "(\%27)|(\')|(\-\-)|(\%23)|(#)") { return 403; }` | Безопасность |
| 13 | **Кастомные страницы ошибок** | `error_page 404 /404.html;` | Профессиональный вид |
| 14 | **`default_server`** | `listen 80 default_server; return 444;` | Блокировка доступа по IP |
| 15 | **Status page** | `stub_status on;` | Мониторинг |

```bash
# ============================================
# БЫСТРАЯ ПРОВЕРКА ВСЕХ КОМПОНЕНТОВ
# ============================================

# Проверка SSL (оценка A+)
openssl s_client -connect example.com:443 -tls1_2 2>/dev/null | openssl x509 -text

# Проверка заголовков безопасности
curl -I https://example.com/ | grep -E "(Strict-Transport-Security|X-Frame-Options|X-Content-Type-Options|X-XSS-Protection)"

# Проверка кеша статики
curl -I https://example.com/static/style.css | grep -E "(Cache-Control|Expires)"

# Проверка Rate Limiting
for i in {1..100}; do curl -s -o /dev/null -w "%{http_code}\n" https://example.com/; done | sort | uniq -c

# Проверка Status Page
curl http://127.0.0.1:8080/nginx_status

# Проверка логов
sudo tail -n 10 /var/log/nginx/access.log
sudo tail -n 10 /var/log/nginx/error.log
```

---

## 18. ПОЛНЫЙ ШАБЛОН ПРОДАКШЕН-КОНФИГА

### 🚀 ГОТОВЫЙ КОНФИГ ДЛЯ КОПИРОВАНИЯ:

```nginx
# ============================================
# /etc/nginx/nginx.conf — ГЛАВНЫЙ КОНФИГ
# ============================================

user www-data;                              # Пользователь для воркеров
worker_processes auto;                      # Количество воркеров
pid /run/nginx.pid;                         # Файл с PID

events {
    worker_connections 1024;                # Макс. соединений на воркер
    use epoll;                              # Механизм мультиплексирования
    multi_accept on;                        # Принимать несколько за раз
}

http {
    # ============================================
    # БАЗОВЫЕ НАСТРОЙКИ
    # ============================================
    
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    sendfile on;
    sendfile_max_chunk 1m;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65s;
    keepalive_requests 1000;
    
    # ============================================
    # БУФЕРЫ
    # ============================================
    
    client_header_buffer_size 1k;
    large_client_header_buffers 4 8k;
    client_body_buffer_size 16k;
    client_body_timeout 5s;
    client_header_timeout 5s;
    client_max_body_size 10M;
    
    proxy_buffering on;
    proxy_buffer_size 4k;
    proxy_buffers 8 4k;
    proxy_busy_buffers_size 8k;
    
    # ============================================
    # ЛОГИ
    # ============================================
    
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';
    
    log_format json escape=json '{'
        '"time":"$time_local",'
        '"remote_addr":"$remote_addr",'
        '"request":"$request",'
        '"status":$status,'
        '"request_time":$request_time'
    '}';
    
    access_log /var/log/nginx/access.log main buffer=32k flush=5s;
    error_log /var/log/nginx/error.log error;
    
    # ============================================
    # БЕЗОПАСНОСТЬ
    # ============================================
    
    server_tokens off;
    
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    
    # ============================================
    # GZIP
    # ============================================
    
    gzip on;
    gzip_comp_level 6;
    gzip_min_length 1000;
    gzip_vary on;
    gzip_types text/plain text/css text/xml text/javascript 
               application/javascript application/json 
               application/xml application/rss+xml image/svg+xml;
    gzip_disable "msie6";
    
    # ============================================
    # SSL
    # ============================================
    
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_session_tickets off;
    
    # ============================================
    # RATE LIMITING
    # ============================================
    
    limit_req_zone $binary_remote_addr zone=ddoslimit:10m rate=50r/s;
    limit_conn_zone $binary_remote_addr zone=connlimit:10m;
    
    # ============================================
    # ПРОКСИ-КЕШ
    # ============================================
    
    proxy_cache_path /var/cache/nginx levels=1:2 
                     keys_zone=mycache:10m 
                     max_size=1g 
                     inactive=60m 
                     use_temp_path=off;
    
    proxy_cache_path /var/cache/nginx-micro levels=1:2 
                     keys_zone=microcache:10m 
                     max_size=100m 
                     inactive=5m 
                     use_temp_path=off;
    
    # ============================================
    # UPSTREAM
    # ============================================
    
    upstream backend {
        least_conn;
        server 10.0.0.1:8080 weight=2 max_fails=3 fail_timeout=30s;
        server 10.0.0.2:8080 weight=1 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }
    
    # ============================================
    # ВКЛЮЧАЕМ ВСЕ КОНФИГИ
    # ============================================
    
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}

# ============================================
# /etc/nginx/conf.d/00-default.conf — БЛОКИРАТОР
# ============================================

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    server_name _;
    return 444;
    access_log off;
    error_log off;
}

# ============================================
# /etc/nginx/sites-available/example.com.conf — САЙТ
# ============================================

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com www.example.com;
    
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    
    root /var/www/example.com;
    index index.html index.php;
    
    access_log /var/log/nginx/example_com_access.log main buffer=32k flush=5s;
    error_log /var/log/nginx/example_com_error.log error;
    
    # Статика
    location ~* \.(jpg|jpeg|png|gif|ico|svg|webp|woff|woff2|ttf|eot|css|js)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }
    
    # Защита от SQL-инъекций
    if ($request_uri ~* "(\%27)|(\')|(\-\-)|(\%23)|(#)") { return 403; }
    if ($request_uri ~* "\.\./") { return 403; }
    
    # Rate Limiting
    limit_req zone=ddoslimit burst=30 nodelay;
    limit_req_status 503;
    limit_conn connlimit 20;
    limit_conn_status 503;
    
    # PHP
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
    
    # Основной location
    location / {
        try_files $uri $uri/ /index.php?$args;
    }
    
    # Страницы ошибок
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;
    
    location = /404.html {
        root /var/www/errors;
        internal;
    }
    
    location = /50x.html {
        root /var/www/errors;
        internal;
    }
}

# HTTP → HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}
```
