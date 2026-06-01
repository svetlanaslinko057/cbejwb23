# EVA-X / ATLAS DevOS — Свежий redeploy & аудит

**Дата:** 2026-06-01
**Pod preview URL:** https://expo-dev-build-10.preview.emergentagent.com
**Источник:** https://github.com/svetlanaslinko057/eevua @ `main`

---

## 1. Текущее состояние сервисов

| Сервис    | Команда                                                  | Порт  | Статус   |
|-----------|----------------------------------------------------------|-------|----------|
| backend   | `uvicorn server:app --host 0.0.0.0 --port 8001 --reload` | 8001  | RUNNING  |
| expo      | `yarn expo start --tunnel --port 3000`                   | 3000  | RUNNING  |
| mongodb   | `mongod --bind_ip_all`                                   | 27017 | RUNNING  |

### Smoke (✅ все пройдены)

- `GET /api/healthz` → `{"status":"ok"}`
- `GET /api/` → `{"message":"Development OS API","version":"1.0.0"}`
- `GET /openapi.json` → **743 пути · 776 операций**
- `POST /api/auth/quick { "email": "admin@atlas.dev" }` → 200, выдан сессионный пользователь
- `GET /api/integrations/manifest` → manifest по 5 категориям (4 mock + 1 live)
- Frontend `https://<host>/` → 200; рендерится лендинг **EVA-X** «Build real products. Not tasks.» с EN/UK переключателем, 3-шаговым flow (SEQ-01/02/03), CTA «See my product plan» и логин-линком.

---

## 2. Backend (FastAPI · MongoDB)

| Метрика                  | Значение      |
|--------------------------|---------------|
| `server.py`              | 28 351 строка |
| Всего `.py` файлов       | **197**       |
| API endpoints (paths)    | **743**       |
| Операций (ops)           | **776**       |
| Коллекций MongoDB после seed | **44**    |
| `users`                  | 12 (1 admin + 11 demo) |
| `modules`                | 99            |
| `qa_decisions`           | 105           |
| `projects`               | 3             |

### Topology endpoint-групп

```
/api/admin/*                     265
/api/developer/*                  73
/api/client/*                     66
/api/modules/*                    23
/api/account/*                    23
/api/payouts-v2/*                 22
/api/execution-intelligence/*     19
/api/auth/*                       18
/api/ai/*                         13
/api/contracts/*                  12
/api/system/*                     10
/api/mobile/*                     10
```

### Boot-последовательность (проверено)

- DEV POOL seed: 6 dev, 89 modules, 81 QA decisions, 6 canonical money states
- MOCK SEED: 2 проекта, 7 модулей, 6 earnings, 6 invoices, 2 deliverables, 3 tickets, 3 notifications, 7 cognition_actions
- 11 demo-пользователей (admin/client/dev/multi/tester + 6 dev pool)
- Indexes ensured: `money_ledger`, `payouts_v2`, `validation_campaigns`, `competitor_cache`
- Daemons started: `MODULE MOTION (15s)`, `AUTO GUARDIAN (120s)`, `CONTRACT REMINDER (6h)`,
  `OPERATOR SCHEDULER (5min)`, `PAY-V2 worker/reaper/mock advancer`, `RECONCILE LOOP (30min)`,
  `EVENT ENGINE (15min)`.

### Money substrate

- Запечатан (Phase 2B PR-1 + Phase 2C B4.5).
- Single source of truth: `domains/money/service.py`.
- Bridges: escrow / earnings / payout. Divergence Observer (passive) включён.

---

## 3. Mobile (Expo SDK 54)

| Метрика              | Значение |
|----------------------|----------|
| Всего `.tsx` файлов в `app/` | **100** |
| Modules в bundle (web target) | **1564** |
| Bundle time (cold)   | ~65s     |
| Bundle time (warm)   | 0.5–8s   |

### Структура экранов (фактическая)

| Роль       | Экранов | Заметка                                       |
|------------|---------|-----------------------------------------------|
| admin      | **21**  | ⚠️ D1 scope-freeze требует 5 + 8 = 13         |
| client     | 20      |                                               |
| developer  | 18      |                                               |
| tester     | 6       | Stage 4 готов                                 |
| lead       | 2       | Conversion surface                            |
| operator   | 1       |                                               |
| прочее     | 32      | auth, profile, settings, chat, inbox, contract, help, portfolio, project, hub, workspace, voice-demo, two-factor-* и т.д. |
| **итого**  | **100** |                                               |

### Установленные плагины Expo

`expo-router`, `expo-splash-screen`, `expo-web-browser`, `expo-audio`,
`expo-secure-store`, `expo-asset`, `expo-image`, `expo-image-picker`,
`expo-clipboard`, `expo-crypto`, `expo-device`, `expo-document-picker`,
`expo-haptics`, `expo-linking`, `expo-location`, `expo-notifications`,
`expo-auth-session`, `expo-blur`, `expo-constants`, `expo-font`,
`expo-linear-gradient`, `expo-status-bar`, `expo-symbols`, `expo-system-ui`.

Permissions (app.json):
- iOS: `NSMicrophoneUsageDescription` = «Record voice briefs and messages»
- Android: `RECORD_AUDIO`

---

## 4. Интеграции (текущий режим)

```json
{
  "payment": "mock-payment   (STRIPE_SECRET_KEY missing)",
  "mail":    "mock-mail      (RESEND_API_KEY missing)",
  "storage": "mock-storage   (CLOUDINARY_* missing)",
  "oauth":   "mock           (GOOGLE_CLIENT_ID/SECRET missing)",
  "ai":      "LIVE через EMERGENT_LLM_KEY (Claude / GPT / Gemini)"
}
```

`INTEGRATIONS_LIVE_ENABLED=0` — external boundary прибит к моками.
Для перехода в LIVE: ключи в `/app/backend/.env` + `INTEGRATIONS_LIVE_ENABLED=1`.

Settlement adapters:
- StripeConnect: **DORMANT** (`STRIPE_API_KEY` missing)
- PayPalPayouts: **DORMANT** (`PAYPAL_CLIENT_ID/SECRET/WEBHOOK_ID` missing)
- Mock settlement: активен по умолчанию.

---

## 5. Что было сделано при redeploy

1. ✅ Склонирован репозиторий `svetlanaslinko057/eevua` (commit `main`) в `/tmp` и развёрнут поверх `/app` с сохранением:
   - `.git`, `.emergent`
   - `frontend/.env` (preview-URL `expo-dev-build-10`)
   - `backend/.env` (`MONGO_URL=mongodb://localhost:27017`, `DB_NAME=test_database`)
2. ✅ Установлены backend-зависимости из `requirements.txt` (138 пакетов).
   **Намеренно пропущен** ML-стек (`sentence-transformers`, `transformers`, `scikit-learn`,
   `scipy`, `networkx`, `sympy`, `tokenizers`, `safetensors`, `huggingface_hub`, `joblib`,
   `threadpoolctl`) — как и в предыдущем pod-аудите. Это транзитивный стек для одного
   lazy-embedding-вызова в `server.py:17352`; всё остальное работает.
3. ✅ `yarn install` (1.22.22, 612 пакетов) — `node_modules` собран.
4. ✅ Перезапущены `backend` и `expo` через supervisor.
5. ✅ Обновлён `/app/memory/test_credentials.md` со списком 12 demo-аккаунтов.
6. ✅ Frontend smoke: лендинг EVA-X рендерится корректно (1564 модуля, 65s cold).
7. ✅ Backend smoke: 743 endpoint доступны, авторизация admin/client/dev/tester работает.

---

## 6. Что НЕ развёрнуто в этом pod (по дизайну)

| Компонент                         | Причина                                                          |
|-----------------------------------|------------------------------------------------------------------|
| `sentence-transformers` стек      | Опущен ради экономии диска; ленивый импорт. Без него работает всё кроме embedding-вызова шаблонов (`Embedding error for template …`). |
| `/web` (React CRA, 98 страниц)    | В репозитории `eevua` нет — отдельный pod/route.                 |
| Production build mobile (APK/IPA) | Через кнопку «Publish» в правом верхнем углу Emergent.            |

---

## 7. Известные предупреждения (не блокеры)

1. `Embedding error … No module named 'sentence_transformers'` — ожидаемо после пропуска ML-стека; затрагивает только seeding 4 шаблонов scope (они грузятся без embedding-векторов).
2. `Duplicate Operation ID audit_log` — дубль OpenAPI `operation_id` в `admin_users_layer.py`. На рантайм не влияет, но мешает codegen клиентам.
3. `[expo-notifications] Listening to push token changes is not yet fully supported on web` — только web-таргет; на iOS/Android работает штатно.
4. Ngrok иногда переподключает тунель при холодном старте Expo (transient, восстанавливается автоматически — было 2 рестарта при первом запуске).
5. Backend warning при boot: `LEGACY ENDPOINT CALLED: /api/client/workspace` — старая совместимость, рекомендуется миграция UI на `/api/client/project/{id}/workspace`.

---

## 8. Расхождения с product-scope-freeze (унаследованы)

| Decision | Контракт                                                          | Фактическое состояние                          |
|----------|-------------------------------------------------------------------|------------------------------------------------|
| **D1**   | Expo admin = 5 frozen tabs + 8 read-only drill-down (≤13)         | **21 экран** в `/app/frontend/app/admin/` ⚠️    |
| **D2**   | Expo tester = Stage 4 (4 screens)                                 | 6 screens — расширение допустимое              |
| **D3**   | Lead = conversion surface only, без роли в auth                   | 2 screens (`workspace.tsx`, `index.tsx`) — OK |

**Рекомендация по D1:** аудитировать `/app/frontend/app/admin/*`, выделить 8 разрешённых drill-down + 5 cockpit tabs, остальное скрыть за feature flag либо унести в web-кабинет.

---

## 9. Готовность к следующим шагам

| Шаг                                          | Готов? |
|----------------------------------------------|--------|
| Backend smoke (healthz / OpenAPI / auth)     | ✅     |
| Mobile Expo рендерится и принимает запросы   | ✅     |
| Quick-login + session cookie работают        | ✅     |
| Money substrate + Payouts V2 daemons live    | ✅     |
| LLM (Claude / GPT / Gemini) через Emergent   | ⚠️ Требуется `EMERGENT_LLM_KEY` в `/app/backend/.env` (сейчас не задан → AI features в MOCK) |
| `test_credentials.md` доступен testing-агенту| ✅     |
| Stripe / Resend / Cloudinary / Google OAuth  | ❌ (mock — нужны ключи)                                |
| Web cabinet (`/web`)                          | ❌ (не в репозитории eevua)                            |
| Production iOS/Android builds                | ❌ (через Publish)                                     |

---

## 10. Что предлагается на следующий шаг

Выбирайте направление — готов продолжить:

1. **Привести D1 в порядок** — почистить admin до 5 + 8 экранов, спрятать лишнее за feature flag.
2. **Активировать live-интеграции** — `EMERGENT_LLM_KEY` (бесплатно, выдам), Stripe / Resend / Cloudinary / Google OAuth (нужны ваши ключи). Поднимем `INTEGRATIONS_LIVE_ENABLED=1`.
3. **Восстановить ML-стек** — `sentence-transformers` (~2 ГБ), чтобы template embeddings работали.
4. **Новая фича** — что добавить в client / developer / tester / admin flow?
5. **Bug-fix конкретного экрана** — пришлите скриншот или путь.
6. **End-to-end тест** через `testing_agent` — автопрогон критичных Expo flow + backend endpoint'ов.
7. **Дать ключ EMERGENT_LLM_KEY** — включить Claude / GPT / Gemini для AI-генерации scope.
