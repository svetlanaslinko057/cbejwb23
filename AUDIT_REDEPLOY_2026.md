# ATLAS DevOS / EVA-X — Redeploy & Audit (свежий pod)

**Дата:** 2026-05-31
**Pod preview URL:** https://7ba3eed7-f471-49fd-9778-5711caafeea7.preview.emergentagent.com
**Источник:** https://github.com/svetlanaslinko057/eveveveve @ `f02daba` (main)

---

## 1. Состояние сервисов

| Сервис    | Команда                                                  | Порт  | Статус   |
|-----------|----------------------------------------------------------|-------|----------|
| backend   | `uvicorn server:app --host 0.0.0.0 --port 8001 --reload` | 8001  | RUNNING  |
| expo      | `yarn expo start --tunnel --port 3000`                   | 3000  | RUNNING  |
| mongodb   | `mongod --bind_ip_all`                                   | 27017 | RUNNING  |

Smoke-проверки:
- `GET /api/healthz` → `{"status":"ok"}` ✅
- `GET /api/` → `{"message":"Development OS API","version":"1.0.0"}` ✅
- `GET /openapi.json` → **743 пути / 776 операций** ✅
- `POST /api/auth/quick` (`admin@atlas.dev` / `john@atlas.dev` / `client@atlas.dev` / `tester@atlas.dev`) → 200 ✅
- `GET /api/integrations/manifest` → manifest по 5 категориям ✅
- Frontend `https://<host>/` → 200, expo-router бандлит `1564 модуля`, рендерится ✅

---

## 2. Backend (FastAPI · MongoDB)

| Метрика                  | Значение      |
|--------------------------|---------------|
| `server.py`              | 28 351 строка |
| Всего `.py` файлов       | **197**       |
| API endpoints (paths)    | **743**       |
| Операций (ops)           | **776**       |
| Коллекций MongoDB после seed | **44**    |
| Документов в `users`     | 12 (1 admin + 11 demo) |
| Документов в `modules`   | 99            |
| Документов в `qa_decisions` | 105        |
| Документов в `projects`  | 3             |

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

### Boot-последовательность (verified)
- DEV POOL seed: 6 dev, 89 modules, 81 QA decisions, 6 canonical money states
- MOCK SEED: 2 проекта, 7 модулей, 6 earnings, 6 invoices, 2 deliverables, 3 tickets, 3 notifications, 7 cognition_actions
- 11 demo-пользователей (admin/client/dev/multi/tester + 6 pool)
- Indexes ensured: money_ledger, payouts_v2, validation_campaigns, competitor_cache
- Daemons started: MODULE MOTION (15s), AUTO GUARDIAN (120s), CONTRACT REMINDER (6h),
  OPERATOR SCHEDULER (5min), PAY-V2 worker/reaper/mock advancer, RECONCILE LOOP (30min),
  EVENT ENGINE (15min)

### Money substrate
- Запечатан (Phase 2B PR-1 + Phase 2C B4.5).
- Single source of truth: `domains/money/service.py`.
- Bridges: escrow / earnings / payout. Divergence Observer (passive) включён.

---

## 3. Mobile (Expo SDK 54)

| Метрика              | Значение |
|----------------------|----------|
| Всего `.tsx` файлов в `app/` | **100** |
| Modules в bundle     | **1564** (web target) |
| Bundle time (cold)   | ~65s     |
| Bundle time (warm)   | 0.5-8s   |

### Структура экранов (фактическая)

| Роль       | Экранов | Заметка                                       |
|------------|---------|-----------------------------------------------|
| admin      | **21**  | ⚠️ D1 scope-freeze требует 5 + 8 = 13         |
| client     | 17      |                                               |
| developer  | 18      |                                               |
| tester     | 6       | Stage 4 готов                                 |
| lead       | 2       | Conversion surface                            |
| operator   | 1       |                                               |
| прочее (auth/profile/settings/chat/inbox/contract/help/portfolio/project/hub/...) | 35 | |
| **итого**  | **100** |                                               |

### Установленные плагины Expo
`expo-router`, `expo-splash-screen`, `expo-web-browser`, `expo-audio`,
`expo-secure-store`, `expo-asset`.

Permissions (app.json):
- iOS: `NSMicrophoneUsageDescription` = "Record voice briefs and messages"
- Android: `RECORD_AUDIO`

---

## 4. Интеграции (текущий режим)

```json
{
  "payment": "mock-payment   (STRIPE_SECRET_KEY missing)",
  "mail":    "mock-mail      (RESEND_API_KEY missing)",
  "storage": "mock-storage   (CLOUDINARY_* missing)",
  "oauth":   "mock           (GOOGLE_CLIENT_ID/SECRET missing)",
  "ai":      "LIVE via EMERGENT_LLM_KEY (Claude / GPT / Gemini)"
}
```

`INTEGRATIONS_LIVE_ENABLED=0` — external boundary прибит к моках.
Для перехода в LIVE: ключи в `/app/backend/.env` + `INTEGRATIONS_LIVE_ENABLED=1`.

Settlement adapters:
- StripeConnect: **DORMANT** (STRIPE_API_KEY missing)
- PayPalPayouts: **DORMANT** (PAYPAL_CLIENT_ID/SECRET/WEBHOOK_ID missing)
- Mock settlement: активен по умолчанию.

---

## 5. Что было сделано при redeploy

1. ✅ Склонирован репозиторий `eveveveve` (commit `f02daba`) в `/tmp` и развёрнут поверх `/app` (с сохранением `.git`, `.emergent`, preview-URL в `frontend/.env`, `MONGO_URL` в `backend/.env`).
2. ✅ Очищен pip cache (освобождено ~3 ГБ).
3. ✅ Установлены backend-зависимости из `requirements.txt` (138 пакетов) — **с сознательным пропуском** ML-стека (`sentence-transformers`, `transformers`, `scikit-learn`, `scipy`, `networkx`, `sympy`, `tokenizers`, `safetensors`, `huggingface_hub`, `hf-xet`, `joblib`, `threadpoolctl`) — как и было сделано в предыдущем аудите для экономии диска. Эти пакеты транзитивны для одного lazy embedding-вызова в `server.py:17352`; всё остальное работает.
4. ✅ Установлены frontend-зависимости через `yarn install` (1.22.22, 612 пакетов).
5. ✅ Перезапущены `backend` и `expo` через supervisor.
6. ✅ Создан `/app/memory/test_credentials.md` со списком 12 demo-аккаунтов.

---

## 6. Что НЕ развёрнуто в этом pod (по дизайну)

| Компонент                         | Причина                                                          |
|-----------------------------------|------------------------------------------------------------------|
| `sentence-transformers` стек      | Опущен ради экономии диска; ленивый импорт, без него работает всё кроме embedding-вызова шаблонов |
| `/web` (React CRA, 98 страниц)    | В репозитории `eveveveve` нет (был в исходном `evav1111`); нужен отдельный pod/route |
| Production build mobile (APK/IPA) | Через кнопку «Publish» в правом верхнем углу Emergent            |

---

## 7. Известные предупреждения (не блокеры)

1. `Embedding error … No module named 'sentence_transformers'` — ожидаемо после пропуска ML-стека; затрагивает только seeding шаблонов scope (4 шаблона прогрузились без embedding).
2. `Duplicate Operation ID audit_log` — дубль OpenAPI operation_id в `admin_users_layer.py`. На рантайм не влияет.
3. `LEGACY ENDPOINT CALLED: /api/client/workspace` — устаревший URL, мигрировать на `/api/client/project/{id}/workspace`.
4. `[expo-notifications] Listening to push token changes is not yet fully supported on web` — только web таргет; на iOS/Android работает штатно.
5. Ngrok иногда переподключает тунель при холодном старте Expo (transient, восстанавливается автоматически).

---

## 8. Расхождения с product-scope-freeze (без изменений)

| Decision | Контракт                                                          | Фактическое состояние                          |
|----------|-------------------------------------------------------------------|------------------------------------------------|
| **D1**   | Expo admin = 5 frozen tabs + 8 read-only drill-down (≤13)         | **21 экран** в `/app/frontend/app/admin/` ⚠️    |
| **D2**   | Expo tester = Stage 4 (4 screens)                                 | 6 screens — расширение допустимое              |
| **D3**   | Lead = conversion surface only, без роли в auth                   | 2 screens (`workspace.tsx`, `index.tsx`) — OK |

**Рекомендация по D1:** аудитировать `/app/frontend/app/admin/*`, выделить 8 разрешённых drill-down + 5 cockpit tabs, остальное скрыть за feature flag или вынести в web cabinet.

---

## 9. Готовность к следующим шагам

| Шаг                                          | Готов? |
|----------------------------------------------|--------|
| Backend smoke (healthz / OpenAPI / auth)     | ✅     |
| Mobile Expo рендерится и принимает запросы   | ✅     |
| Quick-login + session cookie работают        | ✅     |
| Money substrate + Payouts V2 daemons live    | ✅     |
| LLM (Claude / GPT / Gemini) через Emergent   | ✅     |
| `test_credentials.md` доступен testing-агенту| ✅     |
| Stripe / Resend / Cloudinary / Google OAuth  | ❌ (mock — нужны ключи) |
| Web cabinet (`/web`)                          | ❌ (не в репозитории eveveveve) |
| Production iOS/Android builds                | ❌ (через Publish)      |

---

## 10. Что я предлагаю на следующий шаг

Выберите направление — я готов:

1. **Привести D1 в порядок** — почистить admin до 5+8 экранов, спрятать лишнее за feature flag.
2. **Подключить live-интеграции** — Stripe / Resend / Cloudinary / Google OAuth (дайте ключи и я подниму `INTEGRATIONS_LIVE_ENABLED=1`).
3. **Восстановить ML-стек** — поднять `sentence-transformers` обратно (~2 ГБ), чтобы template embeddings работали.
4. **Новая фича** — что добавить в существующий flow (client / developer / tester / admin)?
5. **Bug-fix конкретного экрана** — пришлите скриншот или путь к экрану.
6. **End-to-end тест** через testing_agent — авто-прогон 743 endpoint'ов + критических Expo flow.
