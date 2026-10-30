**Пошукова система актів ВРП**

[1] Iгор Бiлик, Максим Сохацький. <a href="vrp-architecture.pdf">Архiтектура пошукової системи актiв Вищої ради правосуддя на Elixir</a>, 2026.

### Анотація

За зразком `edrsr-neo` (.NET 10 + HTMX 4 + Elasticsearch + Oracle) робимо **другий типовий проєкт** тієї ж архітектурної школи:

- Elixir (Bandit + Plug)
- PostgreSQL — **первинне джерело правди**
- OpenSearch — пошуковий індекс
- HTMX 4 (вендорений)
- Server-rendered HTML-фрагменти
- Два окремі сервіси індексації/синхронізації

### Чотири сценарії розгортання (точно як у ЄДРСР)

```
Кейс 1: Повністю автономний локальний стенд
  [ vrp-web :4000 ] → [ Local Redis ] → [ Local OpenSearch ]
  (без PostgreSQL; Seeder генерує синтетичні акти)

Кейс 2: Гібридний (локальний пошук + віддалений PostgreSQL)
  [ Remote PostgreSQL ] → [ vrp-indexer ] → [ Local OpenSearch ]
  [ vrp-web ] читає тільки локальний OpenSearch

Кейс 3: Чистий Cloud / App-сервер
  [ vrp-web ] → Remote OpenSearch + Remote Redis
  (усі інфраструктурні сервіси зовнішні, конфіг через ENV)

Кейс 4: Docker Compose (production-ready)
  docker compose up → vrp-web + redis + opensearch (+ опційно postgres)
```

### Архітектура сервісів

```
PostgreSQL (primary)
        │
        ├──► vrp-indexer   (повна первинна індексація / reindex)
        │
        └──► vrp-updater   (інкрементальні оновлення)
                │
                ▼
           OpenSearch (alias vrp_acts)
                │
                ▼
           vrp-web (Bandit/Plug + HTMX)
                │
                ├── HTML-сторінки / фрагменти
                └── JSON API
```

**Ролі:**

| Сервіс          | Відповідальність                                      | Технологія                  |
|-----------------|-------------------------------------------------------|-----------------------------|
| `vrp-web`       | Пошук, картка акта, фасети, HTML + JSON API           | Bandit + Plug + Req + Jason |
| `vrp-indexer`   | Повна індексація PostgreSQL → OpenSearch (CLI)        | Elixir CLI (escript / mix)  |
| `vrp-updater`   | Слухає зміни в PostgreSQL і пушить в OpenSearch       | Elixir (LISTEN/NOTIFY або polling) |
| Redis (опц.)    | Fragment-cache HTML + rate-limit                      | Redix                       |

PostgreSQL — єдине місце, куди пише Pentry (або зовнішні системи). Elixir **ніколи** не пише в PostgreSQL у звичайному режимі (тільки читає для індексації).

### Структура проєкту (типовий шаблон після ЄДРСР)

```
vrp-neo/
├── apps/
│   ├── vrp_web/                 # Bandit + Plug (основний веб)
│   │   ├── lib/vrp_web/
│   │   │   ├── application.ex
│   │   │   ├── router.ex
│   │   │   ├── search.ex
│   │   │   ├── act.ex
│   │   │   ├── dict.ex
│   │   │   ├── html.ex          # генерація HTML 4 + фрагменти для HTMX
│   │   │   └── open_search.ex
│   │   └── priv/static/         # htmx.min.js (v4), css
│   │
│   ├── vrp_indexer/             # CLI первинної індексації
│   │   └── lib/vrp_indexer/
│   │       ├── cli.ex
│   │       ├── pipeline.ex      # STAGE 1–4 як у OracleIndexer
│   │       └── bulk.ex
│   │
│   └── vrp_updater/             # сервіс інкрементальних оновлень
│       └── lib/vrp_updater/
│           ├── application.ex
│           ├── listener.ex      # PostgreSQL LISTEN/NOTIFY або polling
│           └── sync.ex
│
├── config/
├── infra/                       # локальний OpenSearch + Redis без Docker
├── vts/                         # smoke-тести (як vts/smoke.sh)
├── Dockerfile
├── docker-compose.yml
├── Makefile                     # make run / daemon / index / update / test ...
└── README.md
```

Монорепо на `mix` + umbrella (або три окремі mix-проєкти — як зручніше).

### Ключові технічні рішення

**1. PostgreSQL → OpenSearch**

- Mapping OpenSearch — точний контракт з числовими ключами (`319`, `186`, `164` nested тощо).
- `vrp-indexer` робить bulk-індексацію пакетами (як OracleIndexer STAGE 1–4).
- `vrp-updater` використовує один з механізмів:
  - `LISTEN/NOTIFY` + тригери на таблицях актів
  - або логічну реплікацію / polling за `updated_at`
  - або outbox-таблицю (найнадійніше)

**2. vrp-web (Bandit/Plug)**

- Повністю stateless.
- HTML генерується на сервері (iodata / EEx).
- HTMX 4 тільки для фрагментів (результати, фасети, suggest, «показати ще»).
- Основний пошук — звичайна форма `method="get"` → стан у URL (`HX-Push-Url`).
- JSON API під тим самим контрактом (`/api/search`, `/api/act/:id`).

**3. Кеш**

- Redis (як у ЄДРСР) для HTML-фрагментів і фасетів.
- Або ETS + Cachex, якщо хочеться ще менше рухомих частин.

**4. Конфігурація (ENV, як у .NET)**

```
VRP_OPENSEARCH_URL=http://localhost:9200
VRP_OPENSEARCH_INDEX=vrp_acts
VRP_POSTGRES_URL=ecto://...
VRP_REDIS=redis://localhost:6379
VRP_CACHE_SECONDS=900
VRP_CURSOR_KEY=...
```

### Makefile (дзеркало edrsr-neo)

```bash
make setup          # залежності + перевірка
make run            # локальний стенд (web + opensearch + redis)
make daemon
make index          # vrp-indexer (повна індексація)
make update         # запуск vrp-updater
make seed           # синтетичні дані
make clean-data
make test           # VTS smoke
make compose-up
make compose-down
```

### Відмінності від ЄДРСР neo (коротко)

| ЄДРСР neo (.NET)          | vrp-neo (Elixir)                  |
|---------------------------|-----------------------------------|
| Oracle                    | PostgreSQL                        |
| ASP.NET Core Minimal API  | Bandit + Plug                     |
| OracleIndexer             | vrp-indexer                       |
| (немає окремого updater)  | vrp-updater (окремий сервіс)      |
| Elasticsearch             | OpenSearch                        |
| Redis fragment cache      | Erlang VM ETS                     |
| HTMX 4                    | HTMX 4 (той самий підхід)         |

---

Це повністю «типовий другий проєкт» після ЄДРСР: та сама ідеологія clean-room, ті самі 4 кейси розгортання, ті самі принципи (серверний HTML + HTMX, стан у URL, окремий індексатор, мінімум залежностей).

Готовий до формування чистого короткого TXT-документа. Можу одразу видати фінальну версію документа в тому ж стилі, що й `edrsr-neo`.
