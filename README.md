
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
16. [Быстрые шпаргалки](#шпаргалки)
17. **[МОДУЛЬ 3: LOCATION](#module-3-location)** ⬅️ НОВЫЙ РАЗДЕЛ

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

## 16. Быстрые шпаргалки {#шпаргалки}

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

---

---

# 📘 МОДУЛЬ 3: LOCATION — МАРШРУТИЗАЦИЯ {#module-3-location}

---

## 17. Что такое `location`?

**Определение:** `location` — это блок внутри `server`, который определяет, как обрабатывать **конкретные URI** (пути) на сайте.

### Синтаксис:
```nginx
location [модификатор] URI {
    # Директивы обработки
}
```

---

## 18. Типы модификаторов location (по приоритету)

| Модификатор | Название | Пример | Приоритет | Что делает |
|-------------|----------|--------|-----------|------------|
| **`=`** | Точное совпадение | `location = /` | **1 (высший)** | Совпадает **только** с указанным URI. Никаких подпапок. |
| **`^~`** | Приоритетное префиксное | `location ^~ /static/` | **2** | Совпадает с началом URI. Если нашлось — **НЕ проверяет регулярки**. |
| **`~`** | Регулярка (регистрозависимый) | `location ~ \.php$` | **3** | Совпадает по регулярному выражению. Чувствителен к регистру букв. |
| **`~*`** | Регулярка (регистронезависимый) | `location ~* \.(jpg|png)$` | **3** | Совпадает по регулярке. **НЕ** чувствителен к регистру. |
| **Без модификатора** | Префиксное совпадение | `location /blog/` | **4** | Совпадает с началом URI. Самый низкий приоритет. |

---

## 19. АЛГОРИТМ ВЫБОРА LOCATION (САМОЕ ВАЖНОЕ!)

### Пошаговый алгоритм:

```
Шаг 1: Ищем "=" (точное совпадение)
       → Если нашли → ИСПОЛЬЗУЕМ (СТОП)

Шаг 2: Ищем все префиксные совпадения (без модификатора и с ^~)
       → Запоминаем САМОЕ ДЛИННОЕ

Шаг 3: Если среди найденных есть "^~" 
       → ИСПОЛЬЗУЕМ его (СТОП, регулярки не проверяются)

Шаг 4: Проверяем все регулярки "~" и "~*" в порядке объявления
       → Выбираем ПЕРВОЕ совпадение (СТОП)

Шаг 5: Используем самое длинное префиксное из Шага 2
```

### Визуальная схема:
```
Запрос: GET /blog/post/123?page=2
         ↓
1. location = /blog/post/123 ?  → НЕТ
         ↓
2. Ищем все префиксные:
   - location /blog/post/123  (длина 15) ← САМОЕ ДЛИННОЕ
   - location /blog/          (длина 5)
   - location /               (длина 1)
         ↓
3. Есть среди них ^~ ?  → НЕТ
         ↓
4. Проверяем регулярки:
   - location ~ \.php$   → НЕТ
   - location ~* \.(jpg|png)$ → НЕТ
         ↓
5. Используем самое длинное префиксное: /blog/post/123
```

---

## 20. Подробный разбор каждого типа

### Тип 1: Точное совпадение (`=`)

```nginx
location = / {
    return 200 "Главная страница";
}

location = /admin {
    return 200 "Только /admin, а не /admin/";
}
```

**Что совпадает:**
| Запрос | Результат |
|--------|-----------|
| `GET /` | ✅ Совпало |
| `GET /index.html` | ❌ НЕ совпало |
| `GET /admin` | ✅ Совпало |
| `GET /admin/` | ❌ НЕ совпало |

**Где использовать:**
- Главная страница (`location = /`)
- Конкретные URL: `/login`, `/health`, `/robots.txt`
- Для максимальной производительности (быстрее всего!)

---

### Тип 2: Приоритетное префиксное (`^~`)

```nginx
location ^~ /static/ {
    root /var/www/site;
    expires 30d;
}

location ^~ /uploads/ {
    root /var/www/site;
    expires 1d;
    # Запрещаем выполнять PHP в папке uploads
    location ~ \.php$ {
        return 403;
    }
}
```

**Что совпадает:**
| Запрос | Результат |
|--------|-----------|
| `GET /static/style.css` | ✅ Совпало |
| `GET /static/images/logo.png` | ✅ Совпало |
| `GET /uploads/file.pdf` | ✅ Совпало |

**Где использовать:**
- Папки со статикой (css, js, images)
- Папки с загрузками пользователей (uploads, media)
- Все, что должно иметь **высший приоритет** над редиректами

---

### Тип 3: Регулярные выражения (`~` и `~*`)

**Регистрозависимый (`~`):**
```nginx
location ~ \.php$ {
    fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
}
```

**Регистронезависимый (`~*`):**
```nginx
location ~* \.(jpg|jpeg|png|gif|ico|svg)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
    access_log off;
}
```

**Что совпадает:**
| Запрос | `~` | `~*` |
|--------|-----|------|
| `/index.php` | ✅ | ✅ |
| `/page.PHP` | ❌ | ✅ |
| `/photo.jpg` | ❌ | ✅ |
| `/photo.JPG` | ❌ | ✅ |

**Где использовать:**
- `~` → для PHP, Python, Ruby (бекенды)
- `~*` → для изображений, css, js (файлы с разным регистром)

---

### Тип 4: Префиксное совпадение (без модификатора)

```nginx
location / {
    try_files $uri $uri/ =404;
}

location /blog/ {
    root /var/www/site;
    try_files $uri $uri/ /blog/index.php;
}
```

**Что совпадает:**
| Запрос | Результат |
|--------|-----------|
| `GET /` | location `/` |
| `GET /about` | location `/` |
| `GET /blog/post/123` | location `/blog/` (длиннее) |
| `GET /blog` | location `/` (нет слеша) |

---

## 21. ВАЖНЫЕ ДИРЕКТИВЫ ВНУТРИ LOCATION

### `try_files` — проверка существования файлов

```nginx
# Базовый вариант
try_files $uri $uri/ =404;

# С передачей в PHP
try_files $uri $uri/ /index.php?$args;

# С несколькими fallback-ами
try_files $uri /fallback.html @custom_location;

# С именованным location
try_files $uri $uri/ @php_handler;

location @php_handler {
    fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
}
```

---

### `alias` — замена пути (отличие от root)

```nginx
# С root: /static/css/style.css → /var/www/site/static/css/style.css
location /static/ {
    root /var/www/site;
}

# С alias: /static/css/style.css → /var/www/static_assets/css/style.css
location /static/ {
    alias /var/www/static_assets/;
}
```

**Важно:** В конце `alias` **должен** быть слеш, если location заканчивается на слеш:

```nginx
location /static/ {
    alias /var/www/static/;  # ✅ Правильно
}

location /static/ {
    alias /var/www/static;   # ⚠️ Не рекомендуется
}
```

---

### `proxy_pass` — проксирование на бекенд

```nginx
# Прокси на локальный сервер
location /api/ {
    proxy_pass http://localhost:3000/;
}

# Прокси с сохранением URI
location /api/ {
    proxy_pass http://localhost:3000;  # Без слеша = /api/users → /api/users
}

# Прокси с изменением URI
location /api/ {
    proxy_pass http://localhost:3000/newapi/;  # Со слешем = /api/users → /users
}
```

**Разница:**
| Запрос | `proxy_pass http://backend:3000` | `proxy_pass http://backend:3000/` |
|--------|----------------------------------|-----------------------------------|
| `/api/users` | `http://backend:3000/api/users` | `http://backend:3000/users` |

---

### `rewrite` — переписывание URI

```nginx
# Постоянный редирект (301)
location /old/ {
    rewrite ^/old/(.*)$ /new/$1 permanent;
}

# Временный редирект (302)
location /temp/ {
    rewrite ^/temp/(.*)$ /new/$1 redirect;
}

# Переписывание без редиректа (внутреннее)
location /shop/ {
    rewrite ^/shop/(.*)$ /store/$1 last;
}

# Сложное переписывание с параметрами
location / {
    rewrite ^/product/([0-9]+)/(.*)$ /item.php?id=$1&slug=$2 last;
}
```

---

### `return` — быстрые ответы

```nginx
# Текстовый ответ
location /health {
    return 200 "OK\n";
    add_header Content-Type text/plain;
}

# Редирект
location /old-page {
    return 301 /new-page;
}

# Ошибка 403
location /secret/ {
    return 403;
}

# Закрыть соединение (без ответа)
location /blocked/ {
    return 444;
}
```

---

### `expires` — управление кешированием

```nginx
# Кеш на 30 дней для статики
location ~* \.(jpg|png|css|js)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
}

# Кеш на 1 час для API
location /api/ {
    expires 1h;
}

# Отключить кеш для админки
location /admin/ {
    expires -1;
    add_header Cache-Control "no-cache, no-store, must-revalidate";
}
```

---

### `internal` — внутренние location

```nginx
# Этот location доступен только для внутренних редиректов
location /internal/ {
    internal;
    root /var/www/private;
}

# Использование с try_files
location / {
    try_files $uri @fallback;
}

location @fallback {
    internal;
    proxy_pass http://backend;
}
```

---

## 22. ВЛОЖЕННЫЕ LOCATION

Location можно вкладывать друг в друга. Это мощный инструмент для переопределения настроек.

```nginx
location /admin/ {
    root /var/www/admin;
    index index.html;
    access_log /var/log/nginx/admin_access.log;

    # Вложенный location: запрещаем доступ к конфигам
    location ~* \.(conf|yml|env)$ {
        return 403;
    }

    # Вложенный location: только админы могут заходить в /admin/users/
    location /users/ {
        auth_basic "Admin Area";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}
```

**Правило:** Вложенный location использует **унаследованные** настройки, если не переопределяет их.

---

## 23. ПРАКТИЧЕСКИЕ СЦЕНАРИИ

### Сценарий 1: Статика + PHP

```nginx
server {
    root /var/www/site;

    # Статика (кеш на месяц)
    location ~* \.(css|js|jpg|png|gif|ico|svg)$ {
        expires 30d;
        access_log off;
        add_header Cache-Control "public, immutable";
    }

    # PHP через FPM
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }

    # Всё остальное
    location / {
        try_files $uri $uri/ /index.php?$args;
    }
}
```

---

### Сценарий 2: Админка с авторизацией

```nginx
server {
    root /var/www/site;

    # Админка с паролем
    location /admin/ {
        auth_basic "Restricted Area";
        auth_basic_user_file /etc/nginx/.htpasswd;

        location ~ \.php$ {
            fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
            include fastcgi_params;
        }
    }

    location / {
        try_files $uri $uri/ /index.php?$args;
    }
}
```

---

### Сценарий 3: API с маршрутизацией

```nginx
server {
    # API версии 1
    location /api/v1/ {
        proxy_pass http://backend_v1:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # API версии 2
    location /api/v2/ {
        proxy_pass http://backend_v2:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # UI приложения
    location / {
        root /var/www/frontend;
        try_files $uri $uri/ /index.html;
    }
}
```

---

### Сценарий 4: Два бекенда + балансировка

```nginx
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}

server {
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/ {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /static/ {
        root /var/www/static;
        expires 30d;
        access_log off;
    }
}
```

---

## 24. ОШИБКИ МОДУЛЯ 3

| Ошибка | Причина | Решение |
|--------|---------|---------|
| Неправильный порядок регулярных выражений | Общая регулярка перехватывает специфичную | Сначала более конкретные регулярки |
| `alias` без слеша | Опасный баг или ошибка | Всегда ставьте слеш в конце alias |
| `proxy_pass` со слешем или без | Непонятно, куда идет запрос | Явно указывайте, нужен ли слеш |
| Использование `try_files $request_uri` | Дыра в безопасности | Всегда используйте `$uri`, а не `$request_uri` |
| Location без `internal` | Может быть доступен извне | Добавьте `internal` для именованных location |

---

## 25. КОНСПЕКТ ПО МОДУЛЮ 3 (шпаргалка)

```nginx
# ============================================
# ТИПЫ LOCATION (по приоритету)
# ============================================

# 1. Точное совпадение (высший приоритет)
location = / { ... }
location = /login { ... }

# 2. Приоритетное префиксное (блокирует проверку регулярных выражений)
location ^~ /static/ { ... }
location ^~ /uploads/ { ... }

# 3. Регулярные выражения (в порядке объявления)
location ~ \.php$ { ... }          # регистрозависимый
location ~* \.(jpg|png)$ { ... }   # регистронезависимый

# 4. Префиксное (низший приоритет)
location / { ... }
location /blog/ { ... }

# ============================================
# КЛЮЧЕВЫЕ ДИРЕКТИВЫ В LOCATION
# ============================================

# Проверка существования файлов
try_files $uri $uri/ =404;
try_files $uri $uri/ /index.php?$args;

# Замена пути (отличие от root)
alias /var/www/static/;

# Прокси на бекенд
proxy_pass http://backend:3000;
proxy_pass http://backend:3000/;

# Редиректы и переписывание
return 301 /new;
return 200 "OK";
rewrite ^/old/(.*)$ /new/$1 permanent;

# Кеширование
expires 30d;
expires -1;
add_header Cache-Control "public, immutable";

# Именованные location (только для внутреннего использования)
location @fallback {
    internal;
    proxy_pass http://backend;
}

# ============================================
# АЛГОРИТМ ВЫБОРА LOCATION
# ============================================
# 1. Ищем "=" (точное совпадение) → используем, если есть
# 2. Ищем префиксные совпадения → запоминаем самое длинное
# 3. Если есть "^~" → используем его (СТОП)
# 4. Ищем регулярки "~" и "~*" → используем ПЕРВОЕ совпадение (СТОП)
# 5. Используем самое длинное префиксное из Шага 2
```
