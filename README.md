# Loom — Magento Backend

Backend-частина headless-проєкту Loom: Magento 2, що надає дані та бізнес-логіку виключно через GraphQL API для клієнтської частини (`loom-storefront`). Docker-оточення побудоване з нуля — власний `docker-compose.yml`, без готових дистрибутивів на кшталт Mark Shust's Docker for Magento.

## Зміст

- [Роль репозиторію в проєкті Loom](#роль-репозиторію-в-проєкті-loom)
- [Технологічний стек](#технологічний-стек)
- [Системні вимоги](#системні-вимоги)
- [Встановлення](#встановлення)
- [Змінні середовища](#змінні-середовища)
- [Запуск застосунку](#запуск-застосунку)
- [Структура проєкту](#структура-проєкту)
- [Опис API](#опис-api)
- [Корисні команди](#корисні-команди)
- [Розгортання](#розгортання)

## Роль репозиторію в проєкті Loom

Проєкт Loom складається з двох незалежних репозиторіїв: цей репозиторій (`loom-magento`) відповідає за серверну частину — постачання каталогу, обробку кошика й замовлень, автентифікацію користувачів через GraphQL. Клієнтська частина (headless storefront на Next.js) розташована в окремому репозиторії `loom-storefront` і звертається до цього backend виключно через `/graphql`, без прямого доступу до бази даних чи файлової системи Magento.

Детальний опис архітектурних рішень та acceptance criteria — у документі [`Loom_AC.md`](Loom_AC.md).

## Технологічний стек

| Шар | Технологія |
|---|---|
| CMS / Backend | Magento 2 (Open Source), 2.4.7-p9 |
| API | Magento GraphQL |
| Мова | PHP 8.3 |
| База даних | MySQL 8.0 |
| Пошук та індексація | OpenSearch 2.12 |
| Кешування, сесії | Redis 7.2 |
| Веб-сервер | Nginx |
| Контейнеризація | Docker, docker-compose (власна конфігурація) |

## Системні вимоги

- Docker Desktop з увімкненою WSL2-інтеграцією (Windows) або Docker Engine (Linux/macOS)
- не менше 4 ГБ вільної оперативної пам'яті, виділеної Docker (OpenSearch є ресурсомістким)
- акаунт та ключі доступу Adobe Commerce Marketplace (Public Key / Private Key) для встановлення пакетів Magento через Composer

## Встановлення

**1. Клонування репозиторію**

```bash
git clone https://github.com/Isoloneya/Loom.git loom-magento
cd loom-magento
```

**2. Складання та запуск контейнерів** (PHP-FPM, MySQL, OpenSearch, Redis)

```bash
docker compose up -d --build phpfpm db opensearch redis
```

**3. Налаштування Composer-автентифікації всередині контейнера**

```bash
docker compose exec phpfpm composer config -g http-basic.repo.magento.com "PUBLIC_KEY" "PRIVATE_KEY"
```

**4. Встановлення Magento у тимчасову директорію та перенесення файлів**

```bash
docker compose exec -w /tmp phpfpm composer create-project \
  --repository-url=https://repo.magento.com/ \
  magento/project-community-edition=2.4.7-p9 magento-tmp

docker compose exec phpfpm sh -c "cp -a /tmp/magento-tmp/. /var/www/html/ && rm -rf /tmp/magento-tmp"
```

**5. Ініціалізація застосунку** (створення схеми БД, адмін-акаунту)

```bash
docker compose exec phpfpm bin/magento setup:install \
  --base-url=http://localhost:8080/ \
  --db-host=db --db-name=magento --db-user=magento --db-password=magento_pass \
  --admin-firstname=Admin --admin-lastname=Loom --admin-email=admin@loom.test \
  --admin-user=admin --admin-password=CHANGE_ME \
  --language=en_US --currency=USD --timezone=Europe/Kyiv --use-rewrites=1 \
  --search-engine=opensearch --opensearch-host=opensearch --opensearch-port=9200
```

**6. Встановлення прав доступу**

```bash
docker compose exec phpfpm chown -R www-data:www-data /var/www/html
```

**7. Копіювання nginx-конфігурації з коду Magento та запуск веб-сервера**

```bash
cp loom-magento/nginx.conf.sample docker/nginx/magento.conf
docker compose up -d nginx
```

**8. Наповнення демо-даними** (категорії, товари)

```bash
docker compose exec -u www-data phpfpm bin/magento sampledata:deploy
docker compose exec -u www-data phpfpm bin/magento setup:upgrade
docker compose exec -u www-data phpfpm bin/magento indexer:reindex
docker compose exec -u www-data phpfpm bin/magento cache:flush
```

**9. Запуск фонового виконання cron-задач**

```bash
docker compose up -d cron
```

Усі команди `bin/magento` виконуються з прапорцем `-u www-data`, щоб уникнути конфлікту прав доступу між процесом PHP-FPM та файлами, створеними під час встановлення.

## Змінні середовища

Параметри підключення до бази даних та OpenSearch задаються безпосередньо в `docker-compose.yml`:

| Параметр | Значення за замовчуванням | Опис |
|---|---|---|
| `MYSQL_DATABASE` | `magento` | назва бази даних |
| `MYSQL_USER` / `MYSQL_PASSWORD` | `magento` / `magento_pass` | облікові дані застосунку |
| `MYSQL_ROOT_PASSWORD` | `magento_root_pass` | пароль root MySQL |

Реальні облікові дані Composer (Public Key/Private Key) не зберігаються в репозиторії — налаштовуються локально командою `composer config`, наведеною в кроці 3 встановлення.

## Запуск застосунку

| Сервіс | Адреса |
|---|---|
| Вітрина (Luma, стандартна тема) | http://localhost:8080/ |
| Адмін-панель | http://localhost:8080/admin_<hash> |
| GraphQL endpoint | http://localhost:8080/graphql |

Точну адресу адмін-панелі (з рандомізованим суфіксом) Magento виводить після виконання `setup:install`.

## Структура проєкту

```
loom/
├── loom-magento/          # код Magento (цей репозиторій)
├── docker/
│   ├── php/
│   │   ├── Dockerfile     # PHP 8.3-FPM з розширеннями, обов'язковими для Magento
│   │   └── php.ini
│   └── nginx/
│       ├── default.conf
│       └── magento.conf   # копія nginx.conf.sample з коду Magento
├── docker-compose.yml     # phpfpm, cron, nginx, db, opensearch, redis
└── Loom_AC.md              # технічне завдання та acceptance criteria проєкту
```

## Опис API

Повна схема GraphQL API доступна після запуску застосунку за адресою `http://localhost:8080/graphql` — сумісна з GraphQL-клієнтами на кшталт Altair GraphQL Client чи Apollo Sandbox для інспекції схеми. Перелік основних запитів і мутацій, що використовуються клієнтською частиною, наведено в розділі AC-06 документа [`Loom_AC.md`](./Loom_AC.md).

Приклад перевірки працездатності API:

```bash
curl -s -X POST http://localhost:8080/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ storeConfig { store_name base_currency_code } }"}'
```

## Корисні команди

**Перегляд стану контейнерів**
```bash
docker compose ps
```

**Перебудова індексів** (після зміни каталогу)
```bash
docker compose exec -u www-data phpfpm bin/magento indexer:reindex
```

**Очищення кешу**
```bash
docker compose exec -u www-data phpfpm bin/magento cache:flush
```

**Перегляд логу cron-задач**
```bash
docker compose exec cron tail -f /var/www/html/var/log/cron.log
```

**Вимкнення двофакторної автентифікації адмінки**
```bash
docker compose exec -u www-data phpfpm bin/magento module:disable \
  Magento_TwoFactorAuth Magento_AdminAdobeImsTwoFactorAuth
```

## Розгортання

Поточна конфігурація призначена для локального середовища розробки (WSL2 + Docker Desktop). Перенесення на хмарний хостинг передбачає винесення MySQL та OpenSearch у керовані сервіси (наприклад, DigitalOcean Managed Databases або AWS RDS/OpenSearch Service) та розгортання PHP-FPM/Nginx-контейнерів на хмарному провайдері з відповідним налаштуванням змінних середовища.
