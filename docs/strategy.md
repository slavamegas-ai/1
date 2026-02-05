# AI_Стратегия — видение и план 3–6 месяцев

## Видение

ИИ‑ассистент селлера WB/маркетплейсов, работающий как **генштаб**: он помогает понимать PnL и cashflow, видеть проблемные SKU, принимать решения по закупкам и рекламе, а также автоматизировать рутину и отчёты.[file:3]  
Через 3–6 месяцев ядро ассистента — это Smart DB на Postgres+pgvector, ежедневный PnL, on‑prem стек (Docker+n8n+Ollama+FastAPI) и Void как основной IDE/оркестратор.[file:3]  
Через 12 месяцев ассистент умеет не только считать PnL, но и объяснять причины изменений, предлагать следующий шаг (что выключить, что масштабировать), а также работает on‑prem с latency ≤10s, VRAM ≤16GB и нулевой стоимостью inference.[file:3]  

---

## Цели 3–6 месяцев (с DoD)

### 1. Void + стек агентов как основной рабочий инструмент

**Цель:** любой код/архитектура/миграции проходят через Void IDE и агентов, а не через "ручной блокнот".[file:3]  

**Definition of Done:**

1. Запущен `ollama serve` на Windows Host с GPU RTX 4080 16GB.  
   - Проверки:  
     - `curl http://localhost:11434` → 200 OK;  
     - `nvidia-smi` показывает использование VRAM при запросе к модели;  
     - Void IDE подключён к localhost:11434 через config.json;  
     - логи Ollama без ошибок CUDA/VRAM.[file:3]  

2. В Ollama установлены модели:  
   - `qwen2.5-coder:32b-instruct-q4_K_M` — **Architect-Local** (архитектура, DDL, docker-compose, wb-fetcher);  
   - `qwen2.5-coder:14b-instruct-q4_K_M` — **Coder-Fast-Local** (CRUD, endpoints);  
   - `mistral-nemo:12b-instruct-q4_0` — **Analyst-LongContext** (SQL, JSON, длинные логи);  
   - опционально `deepseek-r1:32b` — **Reasoner-Local** (сложное рассуждение).[file:3]  

3. В `~/.continue/config.json` настроены профили:  
   - Architect-Local → qwen2.5-coder:32b-instruct-q4_K_M, `numCtx ≈ 6144`, `systemMessage: /docs/prompts/ARCHITECT.md`;  
   - Coder-Fast-Local → qwen2.5-coder:14b-instruct-q4_K_M, `numCtx ≈ 4096`, `systemMessage: /docs/prompts/CODER.md`;  
   - Analyst → mistral-nemo:12b, `systemMessage: /docs/prompts/ANALYST.md`;  
   - Cloud-Senior → OpenRouter Claude Sonnet / GPT‑4.1, `systemMessage: /docs/prompts/JUDGE.md`;  
   - tabAutocomplete отключён (для экономии VRAM), `allowAnonymousTelemetry=false`.[file:3]  

4. В репозитории `/docs/prompts/` есть 4 markdown-файла:  
   - `ARCHITECT.md`, `CODER.md`, `ANALYST.md`, `JUDGE.md`.  
   Каждый файл содержит:  
   - чёткую роль и зону ответственности;  
   - правило: "используй /docs/strategy.md, /docs/kb/ и HTML‑контекст как основной источник, не придумывай с нуля";  
   - правило по PII: все секреты только в `.env`, не в коде и не в логах;  
   - ожидания по работе со Smart DB (wborders, wbfinancereports, wbproducts, mv_daily_pnl);  
   - формат вывода (markdown/JSON/SQL).[file:3]  

5. Прогнаны 3 эталонных сценария (Agentic Flow):  
   - CRUD todos (FastAPI + Postgres + pytest) — код собирается и тесты зелёные;  
   - wb-fetcher skeleton (структура папок, upsert в Postgres, mock‑запрос к WB API);  
   - docker-compose.yml для n8n + Ollama + Postgres+pgvector + Redis + FastAPI + cloudflared, `docker-compose up -d` проходит, healthchecks OK.[file:3]  

Результаты прогонов и время ответов зафиксированы в `/docs/void_benchmarks.md`.[file:3]  

---

### 2. ContextRAG через публичные HTML ссылки

**Цель:** вместо разрозненных заметок — единый контекст в таблицах, автоматически превращающийся в HTML‑страницы, доступные всем LLM и чатам.[file:3]  

**Definition of Done:**

1. Выбран мастер‑источник:  
   - либо Google Sheets (AI_Стратегия, SPACE, Master);  
   - либо локальные .xlsx, которые по расписанию обрабатывает n8n.[file:3]  

2. Есть n8n‑workflow `ContextRAG Sync`, который:  
   - читает диапазоны из таблиц (Google Sheets node / Spreadsheet File node);  
   - преобразует строки в структурированные секции (видение, цели, Space/DoD, дорожная карта L2);  
   - генерирует HTML через Template node (единый дизайн, цвет по priority);  
   - публикует HTML на GitHub Pages или VPS (nginx), формируя стабильные URL:  
     - `/ai_strategy.html`, `/ai_spaces.html`, `/wb_smartdb.html`;  
   - записывает факт обновления в `/docs/decisions.md`.[web:7][web:12][file:3]  

3. HTML‑страницы доступны анонимно:  
   - curl по каждому URL возвращает 200 OK;  
   - в браузере HTML читаемый: заголовки, таблицы, выделены CRITICAL/HIGH.[web:16][file:3]  

4. System‑промпты Architect/Coder/Analyst/Judge и инструкции для Perplexity/Gemini содержат блок:  
   - "Сначала прочитай HTML по ссылкам (стратегия, Space, Smart DB). Используй их как основной контекст; при противоречиях с другими источниками — доверься этим HTML."[file:3]  

5. Проверка: изменение текста/DoD в таблице (например, WBDataPlatform → mv_daily_pnl) в течение ≤5 минут отражается на соответствующей HTML‑странице; Perplexity/Void после обновления пересказывает актуальные цели без расхождений.[web:7][file:3]  

---

### 3. Smart DB + PnL на Postgres 16 + pgvector (VPS)

**Цель:** единая база для PnL, cashflow и RAG, к которой подключаются Excel/BI, TG‑бот и LLM.[file:3]  

**Definition of Done:**

1. Поднят VPS (Aeza/Timeweb/Selectel) 4 vCPU, 8GB RAM, NVMe, Ubuntu 22.04/24.04, установлен Postgres 16 и pgvector.[file:3]  

2. Созданы таблицы:  
   - `wborders` — заказы/продажи с полями (id, date, sku, price, qty, cost, profit, ...);  
   - `wbfinancereports` — финансовые операции (date, sku, amount, penalties, logistics, ...);  
   - `wbproducts` — справочник товаров (sku, name, brand, category, ...);  
   - `wbdocs` — документы для RAG (id, content, embedding vector(768)).[file:3]  

3. Реализована `mv_daily_pnl` и функция `get_daily_pnl(date_from, date_to)`, которые считают дневной PnL (выручка, себестоимость, штрафы, логистика, итог).[file:3]  

4. Настроен wb‑fetcher ETL: Python‑скрипт или n8n workflow, который каждые N часов тянет данные из WB API и делает upsert в таблицы.[file:3]  

5. Настроен ежедневный backup (pg_dump) с проверкой валидности, firewall и SSH‑ключи (см. Security Checklist).[file:3]  

6. Проверка: выборка `get_daily_pnl('2026-01-01','2026-01-31')` даёт 31 строку и суммы совпадают с текущими Excel/Sheets (допуск ±1%).[file:3]  

---

### 4. Telegram‑бот /pnl как основной рычаг

**Цель:** быстрый способ смотреть PnL без входа в базы и BI, прямо из Telegram.[file:3]  

**Definition of Done:**

1. TG‑бот на aiogram 3.x подключён к VPS Postgres (asyncpg/psycopg3).  
2. Команда `/pnl` (и варианты `/pnl 7d`, `/pnl 30d`) делает SELECT из `mv_daily_pnl` и отправляет форматированный ответ.  
3. Команды `/problem_sku`, `/toxic`, `/ad` подготовлены как заглушки (или сразу считают метрики по SKU).  
4. Деплой — systemd или Docker‑контейнер, логи в `/var/log/tg_bot.log`.  
5. Проверка:  
   - запрос `/pnl` от авторизованного пользователя → ответ < 3s, данные совпадают с прямым SELECT;  
   - неавторизованный user_id получает "Доступ запрещён".  

---

### 5. Security + On‑prem Docker MVP

**Цель:** on‑prem стек (n8n + Ollama + Postgres+pgvector + Redis + FastAPI + Cloudflare Tunnel) с реальной защитой и понятной процедурой аудита.[file:3]  

**Definition of Done:**

1. На VPS настроены: UFW, SSH‑ключи, pg_hba.conf, backup.sh + cron, как описано в security_checklist.md.[file:3]  
2. На on‑prem (WSL2 Ubuntu + Docker Desktop) развёрнут docker-compose.yml с сервисами n8n, Ollama, Postgres+pgvector, Redis, FastAPI, cloudflared.[file:3]  
3. Все секреты (ключи, пароли, токены) хранятся только в `.env`, который не попадает в git.[file:3]  
4. Проведён UAT E2E: TG → n8n → RAG → Postgres → ответ, latency ≤10s, VRAM ≤16GB, без падений Redis/Docker.  
5. Составлен и выполнен Security Audit (см. `/docs/security_checklist.md` и `security_audit_YYYY-MM-DD.md`).  

---

## Дорожная карта L2 (Core 3–6 месяцев)

### Таблица L2 (Core 3–6m)

| Проект | Space | Статус | Action |
|--------|-------|--------|--------|
| VoidSetup: Ollama + 4 агента + system-prompts | VoidSetup | В процессе | Доделать config.json, дописать 4 system-промпта, прогнать 3 эталонные задачи, зафиксировать результаты в `/docs/void_benchmarks.md`. |
| ContextRAG: таблица → HTML → публичные URL | ContextRAG | Планируется | Настроить n8n workflow, HTML-шаблон, GitHub Pages/nginx, обновить system-промпты и инструкции для LLM. |
| Smart DB + PnL (VPS Postgres 16+pgvector) | WBDataPlatform | В процессе | Поднять VPS, создать схемы, mv_daily_pnl, wb-fetcher ETL, backup.sh + cron. |
| TG‑бот /pnl + заготовки /problem_sku, /toxic, /ad | TG-Bot | Планируется | Сгенерировать бота через Void, подключить к Postgres, реализовать /pnl, заготовки, whitelist users. |
| Docker AI stack on‑prem | On-prem-infra | В процессе | Сформировать docker-compose.yml, .env.example, поднять стэк, убедиться, что healthchecks OK. |
| RAG v2 + Judge (wbdocs + nomic-embed + n8n workflow) | Dev-RAG | Планируется | Создать wbdocs + индекс, n8n workflow, LLM-Judge, провести бенчмарк recall/latency. |
| Security Checklist + Audit | SecurityProd | Планируется | Написать `/docs/security_checklist.md`, прогнать все тесты, зафиксировать результат в `security_audit_YYYY-MM-DD.md`. |

---

## Правила работы с ИИ

1. Стратегия, архитектура, ключевые решения и ИКР фиксируются здесь (AI_Стратегия, `/docs/strategy.md`, `/docs/decisions.md`).[file:3]  
2. Любой код, миграции, docker-compose, ETL и промпты пишутся через Void/IDE и агентов (Architect, Coder, Analyst, Judge), а не вручную.[file:3]  
3. Любые изменения в проде проходят через PoC/UAT (Dev-RAG, On-prem infra), без "запрос → сразу в прод".[file:3]  
4. Качество и безопасность проверяются через LLM-Judge, чек-листы и security_audit.[file:3]  

---

## Как проверять прогресс (self-check)

Перед тем как считать цикл 3–6 месяцев закрытым:

1. **VoidSetup**: все 4 агента работают, есть `void_benchmarks.md` с 3 эталонными задачами.  
2. **ContextRAG HTML**: ссылки доступны, LLM пересказывают контекст без расхождений.  
3. **Smart DB + PnL**: get_daily_pnl совпадает с Excel/Sheets, ETL стабильный.  
4. **TG‑бот /pnl**: используется ежедневно, latency < 3s.  
5. **Security + Docker**: есть актуальный security_audit, все тесты зелёные.

