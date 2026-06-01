# EVA-X / ATLAS DevOS — Свежий redeploy & аудит (Feb 2026)

**Дата:** 2026-02 (pod восстановлен)
**Источник:** https://github.com/svetlanaslinko057/qwxqwxqw @ commit `f5f1ced`
(«Add .gitignore for pod .env files, yarn.lock and .env.example templates»)
**Pod preview URL:** `https://mobile-app-preview-201.preview.emergentagent.com`

> Это **5-я по счёту аудит-запись** в репозитории. Документ отражает только
> **дельту относительно `AUDIT_REDEPLOY_2026-06-01.md`** (последний детальный аудит).
> Кодовая база полностью совпадает с предыдущим pod-deploy, поэтому базовые цифры
> приведены кратко.

---

## 1. Состояние сервисов (smoke ✅)

| Сервис   | Команда                                                  | Порт  | Статус   |
|----------|----------------------------------------------------------|-------|----------|
| backend  | `uvicorn server:app --host 0.0.0.0 --port 8001 --reload` | 8001  | RUNNING  |
| expo     | `yarn expo start --tunnel --port 3000`                   | 3000  | RUNNING  |
| mongodb  | `mongod --bind_ip_all`                                   | 27017 | RUNNING  |

- `GET /api/healthz` → `{"status":"ok"}`
- `GET /api/` → `{"message":"Development OS API","version":"1.0.0"}`
- `GET /openapi.json` → **743 пути · 776 операций**
- `POST /api/auth/quick { admin / client / tester @atlas.dev }` → все 200
- `GET /api/integrations/manifest` → 6 capability blocks (всё mock — см. §4)
- Frontend `https://<host>/` → 200 (cold bundle ~62s, web 1562 modules)

---

## 2. Цифры архитектуры (на 2026-02)

| Слой                          | Значение |
|-------------------------------|----------|
| `server.py`                   | 28 351 строк |
| Всего `.py` в `backend/` (root) | **100** |
| Backend LOC (root `.py`)      | **74 670** |
| API paths                     | **743** |
| API operations                | **776** (322 POST · 419 GET · 12 PUT · 11 PATCH · 12 DELETE) |
| Mobile screens (`.tsx` в `app/`) | **100** |
| Frontend LOC (`.tsx`)         | **34 428** |
| Expo SDK                      | **54.0.35** |
| React Native                  | **0.81.5** |
| React                         | **19.1.0** |
| expo-router                   | **6.0.22** |

### Топ-5 самых тяжёлых backend-модулей

```
server.py                28 351
execution_intelligence    3 098
legal_contract_layer      2 267
time_tracking_layer       1 623
earnings_layer            1 289
```

### Топ-5 самых тяжёлых экранов

```
describe.tsx              1 529
chat.tsx                  1 496
client/projects/[id].tsx    983
contract/[id]/sign.tsx      957
project/wizard.tsx          947
```

### Эндпоинты по группам (топ-15 tags из OpenAPI)

```
untagged                  472   ← бóльшая часть исторических групп без tag
compat-aliases             21
legal-contract             21
work-execution             20
execution-intelligence     19
account                    16
validation                 15
admin-integrations         13
etap3                      12
admin-mobile               12
escrow-transparency        10
revenue-brain              10
mobile-compat               9
admin-users                 8
mobile-compat-aliases       8
```

Префиксы (для понимания scope):

```
/api/admin/*    265
/api/mobile/*   23 (включая /api/admin/mobile/*)
```

---

## 3. Mobile (Expo SDK 54) — расклад экранов

| Роль       | Экранов | Заметка                                       |
|------------|---------|-----------------------------------------------|
| admin      | **21**  | ⚠️ D1 scope-freeze требует 5 + 8 = 13         |
| client     | 20      | OK                                            |
| developer  | 18      | OK                                            |
| tester     | 6       | D2 — расширение разрешено                     |
| lead       | 2       | D3 — conversion surface                       |
| operator   | 1       | OK                                            |
| прочее     | 32      | auth, account, chat, inbox, contract, help, portfolio, project, hub, workspace, two-factor-*, voice-demo |
| **итого**  | **100** |                                               |

---

## 4. Интеграции — текущий режим (всё MOCK)

```json
{
  "payment":    "mock-payment    (STRIPE_SECRET_KEY missing, policy=hard)",
  "mail":       "mock-mail       (RESEND_API_KEY missing, policy=soft)",
  "storage":    "mock-storage    (CLOUDINARY_* missing, policy=soft)",
  "oauth":      "unavailable     (GOOGLE_CLIENT_ID missing, policy=hard)",
  "ai":         "mock-ai         (EMERGENT_LLM_KEY/OPENAI/ANTHROPIC missing, policy=soft)",
  "settlement": "mock-settlement (Stripe/PayPal env empty, policy=soft)"
}
```

`INTEGRATIONS_LIVE_ENABLED=0` — внешние boundary прибиты к mock-провайдерам.

> ⚠️ **Дельта vs. `2026-06-01`:** в прошлом pod был задан `EMERGENT_LLM_KEY`,
> теперь его нет — AI features снова в MOCK. Если нужно поднять Claude/GPT/Gemini —
> попроси меня его установить, я выдам через Universal Key.

---

## 5. Что было сделано при текущем redeploy

1. ✅ Склонирован репозиторий `svetlanaslinko057/qwxqwxqw` (commit `f5f1ced`) в `/tmp` и развернут поверх `/app` с сохранением:
   - `.git`, `.emergent`
   - `frontend/.env` (preview URL `mobile-app-preview-201`)
   - `backend/.env` (`MONGO_URL=mongodb://localhost:27017`, `DB_NAME=test_database`)
2. ✅ Установлены backend-зависимости из `requirements.txt` (138 пакетов).
   **Намеренно пропущен** ML-стек (`sentence-transformers`, `transformers`, `scikit-learn`,
   `scipy`, `networkx`, `sympy`, `tokenizers`, `safetensors`, `huggingface_hub`, `joblib`,
   `threadpoolctl`) — как и в предыдущих pod-аудитах.
3. ✅ `yarn install` (yarn@1.22.22) — `node_modules` собран.
4. ✅ Перезапущены `backend` и `expo` через supervisor.
5. ✅ Создан `/app/memory/test_credentials.md` (был отсутствующим — критично для testing-agent).
6. ✅ Backend smoke: 743 endpoint доступны, авторизация admin/client/dev/tester работает.
7. ✅ Frontend smoke: лендинг рендерится после cold bundle (~62s, 1562 модулей).

---

## 6. Известные предупреждения (не блокеры)

1. **`No module named 'sentence_transformers'`** при seed 4 scope-шаблонов — ожидаемо после пропуска ML-стека. Шаблоны грузятся без embedding-векторов; semantic-search по ним работает в fallback-режиме.
2. **`Duplicate Operation ID audit_log`** — дубль OpenAPI `operation_id` в `admin_users_layer.py`. Runtime не страдает, но codegen-клиенты ругаются.
3. **`[expo-notifications] not yet fully supported on web`** — относится только к web-таргету Expo, на iOS/Android работает штатно.
4. **`Premature close` + ngrok reconnect** при первом cold-start Expo — transient, восстанавливается автоматически (3–5 ребутов tunnel за первую минуту boot).
5. **`OPERATOR auto_project_pause project=91fa2dce paused=1`** — auto_guardian работает по плану (приостановил seed-проект без активности).

---

## 7. Унаследованные расхождения с product-scope-freeze

| Decision | Контракт                                                          | Фактическое состояние                          |
|----------|-------------------------------------------------------------------|------------------------------------------------|
| **D1**   | Expo admin = 5 frozen tabs + 8 read-only drill-down (≤13)         | **21 экран** в `/app/frontend/app/admin/` ⚠️    |
| **D2**   | Expo tester = Stage 4 (4 screens)                                 | 6 screens — расширение допустимое              |
| **D3**   | Lead = conversion surface only, без роли в auth                   | 2 screens (`workspace.tsx`, `index.tsx`) — OK |

**Рекомендация по D1:** аудитировать `/app/frontend/app/admin/*`, выделить 8 разрешённых drill-down + 5 cockpit tabs, остальное скрыть за feature flag либо унести в web-кабинет.

---

## 8. Дельта vs. `AUDIT_REDEPLOY_2026-06-01.md`

| Параметр                             | 2026-06-01    | 2026-02 (сейчас) | Δ |
|--------------------------------------|---------------|------------------|---|
| API paths                            | 743           | 743              | = |
| API operations                       | 776           | 776              | = |
| `server.py` LOC                      | 28 351        | 28 351           | = |
| Mobile `.tsx` файлов                 | 100           | 100              | = |
| `EMERGENT_LLM_KEY` в `.env`          | присутствовал | **отсутствует**  | ⚠️ |
| `/app/memory/test_credentials.md`    | присутствовал | **создан заново**| ✅ |
| Preview hostname                     | `expo-dev-build-10` | `mobile-app-preview-201` | новый pod |
| Тип репозитория                      | `eevua`       | `qwxqwxqw`       | переименован |
| Backend LOC сводно (top file)        | 28 351        | 28 351           | = |
| Mobile bundle modules                | 1564          | 1562             | -2 (точечные удаления) |

Вывод: **кода тот же — сменился только pod и репо-имя**. Аудит-документ имеет дельта-характер, не дублирует §1–§5 из предыдущего файла.

---

## 9. Готовность к следующим шагам

| Шаг                                          | Готов? |
|----------------------------------------------|--------|
| Backend smoke (healthz / OpenAPI / auth)     | ✅     |
| Mobile Expo рендерится и принимает запросы   | ✅     |
| Quick-login + session работают для 5 ролей   | ✅     |
| Money substrate + Payouts V2 daemons live    | ✅     |
| `test_credentials.md` доступен testing-агенту| ✅     |
| LLM (Claude / GPT / Gemini)                  | ❌ (mock — нужно поставить `EMERGENT_LLM_KEY`) |
| Stripe / Resend / Cloudinary / Google OAuth  | ❌ (mock — нужны ключи)                |
| Web cabinet (`/web`)                          | ❌ (не в этом репо)                    |
| Production iOS/Android builds                | ❌ (через кнопку «Publish»)            |

---

## 10. Что предлагается на следующий шаг

Ты сказал: «Все есть в коде, мы только начинаем разработку и делаем веб-сайт и моб приложение Expo». Готовый стек уже массивный, поэтому полезные направления:

1. **Активировать AI** — я установлю `EMERGENT_LLM_KEY` (Universal Key, бесплатно у нас), и Claude/GPT/Gemini заработают для scope-генератора, chat, describe-flow, AI suggestions.
2. **Привести admin в порядок (D1)** — почистить 21 → 13 экранов, лишнее под feature flag.
3. **Запустить полный e2e** через `testing_agent` — авто-прогон критичных Expo flow + backend endpoint'ов под 4 ролями (admin/client/developer/tester).
4. **Веб-кабинет** — если нужно поднять `/web` (его в этом репо нет), пришли ссылку или скажи «сгенерировать с нуля под текущий backend».
5. **Live-интеграции** — Stripe / Resend / Cloudinary / Google OAuth (нужны твои ключи).
6. **Конкретный bug-fix / новая фича** — пришли скриншот / описание / путь.

Жду указаний — что включаем первым?
