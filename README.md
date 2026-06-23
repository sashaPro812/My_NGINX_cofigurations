# My_NGINX_cofigurations

# Theory
## Оглавление (для быстрой навигации)
1. [Архитектура NGINX](#архитектура)
2. [Установка](#установка)
3. [Структура директорий](#структура)
4. [Структура конфига](#структура-конфига)
5. [Основные команды](#команды)
6. [Директивы listen и server_name](#listen-server_name)
7. [Базовые директивы](#базовые-директивы)
8. [Виртуальные хосты (Server Blocks)](#виртуальные-хосты)
9. [Создание сайта (пошагово)](#создание-сайта)
10. [Включение/отключение сайтов](#включение-отключение)
11. [Логирование](#логирование)
12. [default_server](#default_server)
13. [Приоритеты выбора server](#приоритеты-выбора)
14. [Переменные NGINX](#переменные)
15. [Типичные ошибки](#ошибки)

---

## 1. Архитектура NGINX {#архитектура}

- **Мастер-процесс** — управляет воркерами, читает конфиги.
- **Воркеры** (worker processes) — обрабатывают запросы. Обычно = количеству ядер CPU.
- **Событийно-ориентированная модель** — один воркер может обрабатывать тысячи соединений (неблокирующий ввод-вывод, epoll/kqueue).
- **Отличие от Apache:** Apache — процессы/потоки на каждый запрос (тяжеловесно), NGINX — асинхронный (легковесно).

---

## 2. Установка NGINX (Ubuntu/Debian) {#установка}

### Установка из официального репозитория (свежая версия):

```bash
# Добавляем ключ репозитория
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null

# Добавляем репозиторий
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/ubuntu `lsb_release -cs` nginx" | sudo tee /etc/apt/sources.list.d/nginx.list

# Устанавливаем
sudo apt update
sudo apt install nginx -y
```

### Альтернатива (стандартный репозиторий, но версия старше):
```bash
sudo apt update
sudo apt install nginx -y
```

### Проверка установки:
```bash
nginx -v              # Версия
systemctl status nginx # Статус сервиса
```

---

## 3. Структура директорий {#структура}

| Путь | Назначение |
|------|------------|
| `/etc/nginx/nginx.conf` | Главный конфигурационный файл |
| `/etc/nginx/conf.d/` | Дополнительные конфиги (читаются автоматически) |
| `/etc/nginx/sites-available/` | Хранилище конфигов сайтов |
| `/etc/nginx/sites-enabled/` | Активные сайты (симлинки на available) |
| `/usr/share/nginx/html/` | Стандартная папка для статики (рекомендуется `/var/www/`) |
| `/var/log/nginx/access.log` | Лог всех запросов |
| `/var/log/nginx/error.log` | Лог всех ошибок |
| `/etc/nginx/mime.types` | Сопоставление расширений с MIME-типами |

### Порядок загрузки конфигов:
1. Читается `/etc/nginx/nginx.conf`
2. Внутри — `include /etc/nginx/conf.d/*.conf;`
3. Затем — `include /etc/nginx/sites-enabled/*;`

**Важно:** Файлы читаются в алфавитном порядке (лексикографически).

---

## 4. Структура конфигурационного файла {#структура-конфига}

```nginx
# ==========================================
# ГЛАВНЫЙ КОНТЕКСТ (main)
# ==========================================
user www-data;
worker_processes auto;   # auto = количество ядер CPU
pid /run/nginx.pid;

# ==========================================
# КОНТЕКСТ EVENTS (события)
# ==========================================
events {
    worker_connections 1024;  # Максимум соединений на воркер
    use epoll;                # Механизм мультиплексирования (Linux)
}

# ==========================================
# КОНТЕКСТ HTTP (веб-трафик)
# ==========================================
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Формат логов
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    # ==========================================
    # БЛОК SERVER (виртуальный хост)
    # ==========================================
    server {
        listen 80;
        server_name _;

        root /var/www/mysite;
        index index.html;

        access_log /var/log/nginx/mysite_access.log;
        error_log /var/log/nginx/mysite_error.log;

        # ==========================================
        # БЛОК LOCATION (маршрутизация)
        # ==========================================
        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

### Иерархия контекстов:
```
main
  ├── events
  └── http
        └── server
              └── location
```

---

## 5. Основные команды {#команды}

| Команда | Что делает |
|---------|------------|
| `sudo nginx -t` | Проверка синтаксиса конфигов |
| `sudo nginx -T` | Проверка + вывод полной итоговой конфигурации |
| `sudo nginx -s reload` | Перезагрузить конфиг без остановки сервера |
| `sudo nginx -s stop` | Быстрая остановка |
| `sudo nginx -s quit` | Плавная остановка (дожидается завершения запросов) |
| `nginx -V` | Версия + список скомпилированных модулей |
| `sudo systemctl reload nginx` | Альтернатива reload |
| `sudo systemctl status nginx` | Статус сервиса |
| `sudo ss -tulpn \| grep nginx` | Проверить, какие порты слушает NGINX |
| `sudo journalctl -u nginx -f` | Логи systemd в реальном времени |

---

## 6. Директивы listen и server_name {#listen-server_name}

### `listen` — порт и флаги:
```nginx
listen 80;                      # Обычный порт
listen 80 default_server;       # Используется, если нет совпадений server_name
listen 8080;                    # Альтернативный порт
listen 443 ssl;                 # HTTPS (SSL/TLS)
listen [::]:80;                 # IPv6
listen 80 backlog=4096;         # Размер очереди подключений
listen 80 default_server backlog=4096;  # Комбинированный вариант
```

### `server_name` — доменные имена (приоритеты):
```nginx
server_name example.com;                    # 1. Точное совпадение (высший приоритет)
server_name *.example.com;                  # 2. Wildcard (средний приоритет)
server_name ~^(www\.)?example\.com$;        # 3. Регулярное выражение
server_name _;                              # 4. Все остальные (низший приоритет)
server_name "";                             # 5. Запросы без заголовка Host
```

**Примеры wildcard:**
```nginx
server_name .example.com;      # Эквивалентно example.com *.example.com
server_name example.*;         # example.com, example.net, example.org (НЕ РАБОТАЕТ!)
```

---

## 7. Базовые директивы {#базовые-директивы}

### `root` — корневая папка сайта:
```nginx
root /var/www/site;   # Путь к файлам сайта
```

### `index` — индексные файлы по умолчанию:
```nginx
index index.html index.htm index.php;  # Проверяются по порядку
```

### `alias` — замена пути (отличается от root!):
```nginx
location /static/ {
    alias /var/www/static/;  # /static/image.jpg → /var/www/static/image.jpg
}

location /images/ {
    root /var/www/site;      # /images/photo.jpg → /var/www/site/images/photo.jpg
}
```

**Разница root vs alias:**
- `root` — добавляет URI к пути
- `alias` — полностью заменяет URI на путь

### `try_files` — проверка существования файлов:
```nginx
# Если файл есть — отдать, если нет — 404
location / {
    try_files $uri $uri/ =404;
}

# Если файла нет — отдать index.php с параметрами
location / {
    try_files $uri $uri/ /index.php?$args;
}

# Кастомная страница 404
location / {
    try_files $uri $uri/ /404.html;
}
```

### `return` — редиректы и ответы:
```nginx
return 200 "OK\n";                      # Простой текстовый ответ
return 301 /new-page;                   # Постоянный редирект
return 302 /new-page;                   # Временный редирект
return 404;                             # Страница 404
return 444;                             # Закрыть соединение без ответа (NGINX-specific)
```

### `add_header` — добавление HTTP-заголовков:
```nginx
add_header X-Frame-Options DENY;
add_header Content-Type text/plain;
add_header Cache-Control "no-cache, no-store, must-revalidate";
```

---

## 8. Виртуальные хосты (Server Blocks) {#виртуальные-хосты}

### Концепция `sites-available` и `sites-enabled`

- **`sites-available`** — Хранилище всех конфигов (включены/выключены).
- **`sites-enabled`** — Активные конфиги (только симлинки из `available`).

**Зачем:** Можно хранить конфиги отключенных сайтов, быстро включать/отключать без удаления файлов.

### Как проверить порядок загрузки:
```bash
# Показывает все файлы в порядке загрузки
sudo nginx -T 2>/dev/null | grep -E "^# configuration file"
```

---

## 9. Создание сайта (пошагово) {#создание-сайта}

### Шаг 1. Создать папку для сайта:
```bash
sudo mkdir -p /var/www/newsite
echo '<h1>Новый сайт</h1>' | sudo tee /var/www/newsite/index.html
```

### Шаг 2. Создать конфиг в `sites-available`:
```bash
sudo tee /etc/nginx/sites-available/newsite.conf > /dev/null << 'EOF'
server {
    listen 80;
    server_name newsite.local;

    root /var/www/newsite;
    index index.html;

    access_log /var/log/nginx/newsite_access.log;
    error_log /var/log/nginx/newsite_error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF
```

### Шаг 3. Активировать сайт (симлинк):
```bash
sudo ln -s /etc/nginx/sites-available/newsite.conf /etc/nginx/sites-enabled/
```

### Шаг 4. Проверить и перезагрузить:
```bash
sudo nginx -t && sudo nginx -s reload
```

### Шаг 5. Добавить домен в `/etc/hosts` (для локальной разработки):
```bash
echo "127.0.0.1 newsite.local" | sudo tee -a /etc/hosts
```

---

## 10. Включение/отключение сайтов {#включение-отключение}

### Включить сайт:
```bash
sudo ln -s /etc/nginx/sites-available/site.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo nginx -s reload
```

### Отключить сайт (без удаления конфига):
```bash
sudo rm /etc/nginx/sites-enabled/site.conf
sudo nginx -t && sudo nginx -s reload
```

### Полностью удалить конфиг сайта:
```bash
sudo rm /etc/nginx/sites-available/site.conf
sudo rm /etc/nginx/sites-enabled/site.conf  # если забыли удалить симлинк
```

### Просмотр активных сайтов:
```bash
ls -la /etc/nginx/sites-enabled/
```

---

## 11. Логирование {#логирование}

### Настройка логов в конфиге:
```nginx
# В блоке http (глобально) или server (для конкретного сайта)
access_log /var/log/nginx/my_site_access.log;          # Лог запросов
error_log /var/log/nginx/my_site_error.log warn;       # Лог ошибок + уровень
```

### Уровни ошибок (от низшего к высшему):
```
debug → info → notice → warn → error → crit → alert → emerg
```

**Продакшн-стандарт:** `error_log /path/to/log error;` (пишет только ошибки и критичные события).

### Форматы логов (кастомизация):
```nginx
# В блоке http
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';

log_format json escape=json '{'
    '"time_local":"$time_local",'
    '"remote_addr":"$remote_addr",'
    '"request":"$request",'
    '"status":$status,'
    '"body_bytes_sent":$body_bytes_sent,'
    '"http_referer":"$http_referer",'
    '"http_user_agent":"$http_user_agent"'
'}';

# Использование формата
access_log /var/log/nginx/api_access.log json;
```

### Просмотр логов:
```bash
# Посмотреть последние строки
sudo tail -n 20 /var/log/nginx/error.log
sudo tail -n 50 /var/log/nginx/access.log

# Следить в реальном времени
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log

# Искать ошибки
sudo grep "error" /var/log/nginx/error.log
sudo grep "404" /var/log/nginx/access.log
```

---

## 12. default_server {#default_server}

### Что это?
Флаг, указывающий, какой `server` отвечает, если не найдено совпадений по `server_name`.

### Синтаксис:
```nginx
server {
    listen 80 default_server;
    server_name _;
    return 444;  # Закрыть соединение без ответа
}
```

### Если `default_server` не указан явно:
NGINX автоматически назначает первый загруженный конфиг (по алфавиту в `sites-enabled/` и `conf.d/`).

### Правило:
На одном порту может быть **только один** `default_server`. Иначе ошибка:
```
nginx: [emerg] a duplicate default server for 0.0.0.0:80
```

### Лучшая практика (блокиратор левого трафика):
```nginx
# /etc/nginx/conf.d/00-default.conf
server {
    listen 80 default_server;
    listen 443 ssl default_server;
    server_name _;
    
    # Возвращаем 444 (закрываем соединение)
    return 444;
    
    # Или редирект на основной сайт:
    # return 301 https://mysite.com$request_uri;
}
```

---

## 13. Приоритеты выбора server {#приоритеты-выбора}

При запросе NGINX выбирает `server` в таком порядке:

1. **Точное совпадение** `server_name` (`example.com`)
2. **Wildcard** (`*.example.com`) — чем длиннее, тем выше приоритет
3. **Регулярное выражение** (`~^...`) — первое совпадение
4. **`default_server`** (если указан явно)
5. **Первый загруженный** (неявный default)

**Пример приоритетов:**
```nginx
server_name example.com;       # Приоритет 1 (самый высокий)
server_name *.example.com;     # Приоритет 2
server_name ~^(www\.)?example\.com$;  # Приоритет 3
server_name _;                 # Приоритет 4 (самый низкий)
```

---

## 14. Переменные NGINX {#переменные}

### Для логов и конфигов:

| Переменная | Описание | Пример |
|------------|----------|--------|
| `$remote_addr` | IP-адрес клиента | `192.168.1.1` |
| `$remote_user` | Имя пользователя (если есть basic auth) | `admin` |
| `$time_local` | Время запроса | `23/Jun/2026:18:04:56 +0000` |
| `$request` | Полный запрос | `GET /index.html HTTP/1.1` |
| `$request_uri` | Полный URI с параметрами | `/page?param=1` |
| `$uri` | URI без параметров | `/page` |
| `$args` | Параметры запроса | `param=1` |
| `$host` | Домен из заголовка Host | `example.com` |
| `$status` | Код ответа | `200`, `404`, `500` |
| `$body_bytes_sent` | Размер ответа в байтах | `1234` |
| `$http_referer` | Откуда пришел пользователь | `https://google.com/` |
| `$http_user_agent` | Браузер клиента | `Mozilla/5.0...` |
| `$http_x_forwarded_for` | Реальный IP за прокси | `10.0.0.1` |
| `$server_name` | Имя сервера из конфига | `example.com` |
| `$document_root` | Корневая папка текущего запроса | `/var/www/site` |
| `$request_method` | Метод запроса | `GET`, `POST`, `PUT` |

### Пример использования в конфиге:
```nginx
location / {
    add_header X-Request-URI $request_uri;
    add_header X-Real-IP $remote_addr;
    add_header X-Server-Name $server_name;
}
```

---

## 15. Типичные ошибки и их решение {#ошибки}

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `duplicate default server` | Два `default_server` на одном порту | Убрать у одного или удалить дефолтный конфиг |
| `nginx: [emerg] bind() to 0.0.0.0:80 failed` | Порт 80 занят | `sudo killall apache2` или `sudo systemctl stop apache2` |
| `Permission denied` | Нет прав на папку | `sudo chown -R www-data:www-data /var/www/site` |
| 404 Not Found | Неправильный путь `root` или отсутствует файл | Проверить `nginx -T` и существование файла |
| Страница не открывается по домену | Домен не прописан в `/etc/hosts` | Добавить `127.0.0.1 домен.local` |
| Сайт не появился после создания симлинка | Забыли перезагрузить | `sudo nginx -s reload` |
| Конфиг не применился | Синтаксическая ошибка | Проверить `sudo nginx -t` |
| 502 Bad Gateway | Бекенд (PHP-FPM/Node) не отвечает | Проверить, запущен ли бекенд-сервис |
| `Primary script unknown` | PHP не найден | Проверить путь в `fastcgi_pass` и существование файла |
| `nginx: [emerg] "server" directive is not allowed here` | `server` вне `http` | Поместить `server` внутрь блока `http` |

---

## 16. Быстрые шпаргалки

### Создать простой статический сайт за 1 минуту:
```bash
sudo mkdir -p /var/www/site
echo '<h1>Hello NGINX</h1>' | sudo tee /var/www/site/index.html
sudo tee /etc/nginx/sites-available/site.conf > /dev/null << 'EOF'
server {
    listen 80;
    server_name site.local;
    root /var/www/site;
    index index.html;
    location / { try_files $uri $uri/ =404; }
}
EOF
sudo ln -s /etc/nginx/sites-available/site.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo nginx -s reload
```

### Просмотр итоговой конфигурации:
```bash
sudo nginx -T | less
```

### Поиск директивы в конфигах:
```bash
grep -r "server_name" /etc/nginx/
```

### Перезапуск NGINX (полный):
```bash
sudo systemctl restart nginx
```

### Тест производительности (стресс-тест):
```bash
# Установить apache2-utils для ab
sudo apt install apache2-utils -y

# 1000 запросов, 10 одновременных
ab -n 1000 -c 10 http://localhost/
```
