# ATLAS DevOS / EVA-X — Аудит после полного redeploy

**Дата:** 2026-05-31
**Pod preview URL:** https://app-preview-expo-1.preview.emergentagent.com
**Источник:** https://github.com/svetlanaslinko057/evevua @ main `089a730`
**Тип:** полное восстановление кодовой базы на чистый pod + smoke-аудит

---

## 0. Что было сделано

1. Локальный `/app` содержал только initial-template (`README + backend/server.py заглушка + frontend/app/index.tsx`).
2. Из репозитория `svetlanaslinko057/evevua` (8 коммитов от Emergent E1 Agent) скопированы:
   - `backend/` (197 .py файлов, server.py = 28 351 строка)
   - `frontend/` (100 .tsx экранов, Expo SDK 54)
   - `memory/PRD.md`, `memory/test_credentials.md`
   - `tests/`, `test_reports/`, `test_result.md`
   - Существующие аудиты (`AUDIT_2026-05-31.md`, `AUDIT_REDEPLOY_2026.md`)
3. Сохранены protected env-переменные pod:
   - `backend/.env`: `MONGO_URL`, `DB_NAME` + добавлены `INTEGRATIONS_LIVE_ENABLED=0`, `EMERGENT_LLM_KEY` (LIVE).
   - `frontend/.env`: оригинальные `EXPO_PACKAGER_*` сохранены.
4. Установлены зависимости:
   - `pip install -r backend/requirements.txt` → 60+ новых пакетов
     (FastAPI, motor, emergentintegrations, stripe, resend, reportlab,
     google-genai, litellm-deps, python-socketio, pyotp, qrcode, slowapi, …)
   - `yarn install` во `frontend/` → expo-audio, expo-notifications,
     react-native-reanimated, socket.io-client и другие 50+ пакетов.
5. Все 3 supervisor-сервиса перезапущены и подняты в RUNNING.

---

## 1. Состояние сервисов

| Сервис   | Команда                                                  | Порт  | Статус   |
|----------|----------------------------------------------------------|-------|----------|
| backend  | `uvicorn server:app --host 0.0.0.0 --port 8001 --reload` | 8001  | RUNNING  |
| expo     | `yarn expo start --tunnel --port 3000`                   | 3000  | RUNNING  |
| mongodb  | `mongod --bind_ip_all`                                   | 27017 | RUNNING  |

**Smoke-проверки (все ✅):**
- `GET /api/healthz` → `{"status":"ok"}`
- `GET /api/` → `{"message":"Development OS API","version":"1.0.0"}`
- `GET /openapi.json` → **743 endpoints**
- `POST /api/auth/quick` для admin / john / client / tester → 200 для всех 4
- `GET /api/client/workspace` (с сессионной cookie) → 200
- `GET /api/integrations/manifest` → `capabilities/server_time/ttl_ms/version`
- Frontend `https://app-preview-expo-1.preview.emergentagent.com` → 200,
  рендерится лендинг **EVA-X**: «Build real products. Not tasks.»,
  SEQ-01/02/03, language toggle EN/UK, CTA «See my product plan».

---

## 2. Backend (FastAPI · MongoDB)

| Метрика                  | Значение      |
|--------------------------|---------------|
| `server.py`              | **28 351 строка** |
| Всего `.py` файлов       | **197**       |
| API endpoints (paths)    | **743**       |
| Коллекций MongoDB        | **44**        |
| Документов в `users`     | 12 (1 admin + 11 demo) |
| Документов в `modules`   | 99            |
| Документов в `qa_decisions` | 105        |
| Документов в `projects`  | 3             |

### Топология endpoint-групп (top-15)
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
/api/projects/*                    8
/api/validation/*                  8
/api/provider/*                    8
```

### Boot-последовательность (verified в логах)
- DEV POOL seed: 6 dev, 89 modules, 81 QA decisions, 6 canonical money states
- MOCK SEED: 2 проекта, 7 модулей, 6 earnings, 6 invoices, 2 deliverables,
  3 tickets, 3 notifications, 7 cognition_actions
- 11 demo-пользователей (admin/client/dev/multi/tester + 6 pool) — см. `memory/test_credentials.md`
- L0/L1 backfill: 12 users + 1 admin
- TESTER SEED: 5 validations + 1 issue → tester@atlas.dev
- Indexes ensured: money_ledger, payouts_v2, validation_campaigns, competitor_cache
- Daemons started:
  - **MODULE MOTION** (15s)
  - **AUTO GUARDIAN** (120s)
  - **CONTRACT REMINDER** (21600s / 6h)
  - **OPERATOR SCHEDULER** (5 min)
  - **PAY-V2 worker** (5s, batch=10, lease=60s, max_attempts=5)
  - **PAY-V2 reaper** (30s)
  - **PAY-V2 mock advancer** (5s, delay=2s)
  - **PAY-V2 reconcile loop** (900s)
  - **EVENT ENGINE** (15 min)
- Scope templates: 4 seeded.

### Money substrate
Запечатан (Phase 2B PR-1 + Phase 2C B4.5 Divergence Observer passive).
Single source of truth: `domains/money/service.py`.
Bridges: escrow / earnings / payout.

### Известные warning-и (не блокирующие)
- `Embedding error for template Online Marketplace: No module named 'sentence_transformers'`
  — повторяется для 4 шаблонов. Сейчас работает текстовый fallback. Активация
  требует `pip install sentence-transformers` (~1.5 GB модели).

---

## 3. Mobile (Expo SDK 54)

| Метрика              | Значение |
|----------------------|----------|
| Всего `.tsx` файлов в `app/` | **100** |
| Modules в bundle     | **1564 → 1565** (web target) |
| Bundle time (cold)   | ~4s (теплый кеш `.metro-cache`) |
| Tunnel               | ngrok через `EXPO_TUNNEL_SUBDOMAIN=app-preview-expo-1` |

### Структура экранов

| Роль       | Экранов | Заметка                                       |
|------------|---------|-----------------------------------------------|
| admin      | **21**  | ⚠️ Расхождение D1: scope-freeze требует 5+8=13 |
| client     | **20**  |                                               |
| developer  | **18**  |                                               |
| tester     | **6**   | Stage 4 готов                                 |
| lead       | **2**   | Conversion surface                            |
| operator   | **1**   |                                               |
| прочее (auth/profile/settings/chat/inbox/contract/help/portfolio/project/hub/...) | 32 | |
| **итого**  | **100** |                                               |

### Установленные плагины Expo
`expo-router`, `expo-splash-screen`, `expo-web-browser`, `expo-audio`,
`expo-secure-store`, `expo-asset`, `expo-notifications`, `expo-image`,
`expo-image-picker`, `expo-location`, `expo-haptics`, `expo-clipboard`,
`expo-symbols`, `expo-blur`, `expo-document-picker`, `expo-device`,
`expo-crypto`, `expo-auth-session`, `expo-system-ui`, `expo-status-bar`,
`expo-font`, `expo-linking`, `expo-constants`, `react-native-reanimated`,
`react-native-gesture-handler`, `react-native-screens`,
`react-native-safe-area-context`, `react-native-webview`,
`react-native-worklets`, `socket.io-client`, `axios`.

Permissions (app.json):
- iOS: `NSMicrophoneUsageDescription` = «Record voice briefs and messages»
- Android: `RECORD_AUDIO`

### i18n
`frontend/src/i18n.tsx` — **1929 EN / 1929 UK пар** (по PRD; v5 коммит).
Реалізовано: StatusPill labels (invoice/stage/risk/mode/ticket), auth/welcome/
2FA/describe/estimate-result, developer/admin/tester/operator surfaces.
EN — default, переключатель пилюль EN/UK в welcome (`testID welcome-lang-uk`).

---

## 4. Integrations boundary

| Категория | Провайдер             | Режим в pod | Активация                              |
|-----------|-----------------------|-------------|----------------------------------------|
| AI / LLM  | emergentintegrations  | **LIVE**    | `EMERGENT_LLM_KEY=sk-emergent-***`     |
| Payment   | Stripe / WayForPay    | MOCK        | `STRIPE_SECRET_KEY` / `WAYFORPAY_*` пусты |
| Mail      | Resend                | MOCK        | `RESEND_API_KEY` пусто                  |
| Storage   | Cloudinary            | MOCK (local)| `CLOUDINARY_*` пусто                    |
| OAuth     | Google                | MOCK        | `GOOGLE_CLIENT_ID/SECRET` пусто         |

`INTEGRATIONS_LIVE_ENABLED=0` → весь external boundary через mocks.
Flip на `1` + ключі → live adapters без изменений бизнес-логики.

---

## 5. Тестовые аккаунты

См. `memory/test_credentials.md` (12 demo accounts для testing-agent):
- `admin@atlas.dev` — Atlas Admin (role=admin)
- `john@atlas.dev` — developer
- `client@atlas.dev` — client
- `multi@atlas.dev` — мульти-роль
- `tester@atlas.dev` — Stage 4 валидации
- + 6 dev pool аккаунтов с canonical money states
- + lead-демо

Auth через `POST /api/auth/quick` (email-only, OTP-bypass для демо).

---

## 6. Расхождения vs scope-freeze D1

⚠️ **admin: 21 экран** против ожидаемых **13** (5 cockpit + 8 drill-down).
Это унаследовано из репо. Решение: либо обновить D1 в PRD, либо
выделить «дополнительные» admin-экраны как experimental.

---

## 7. Готовность к работе

✅ Полный redeploy завершён.
✅ Backend (743 endpoints), frontend (100 экранов) и MongoDB (44 коллекции) RUNNING.
✅ Все 4 ключевих demo-аккаунти автентифікуються.
✅ Welcome-лендинг рендериться, language toggle EN/UK активний.
✅ Daemons (9 фонових петель) працюють.
✅ EMERGENT_LLM_KEY активований — AI-фічі в LIVE-режимі.

**Готово до продовження розробки.**

---

## Quick reference

- Frontend: `https://app-preview-expo-1.preview.emergentagent.com`
- Backend health: `https://app-preview-expo-1.preview.emergentagent.com/api/healthz`
- OpenAPI: `https://app-preview-expo-1.preview.emergentagent.com/openapi.json` (743 paths)
- Logs: `/var/log/supervisor/{backend,expo}.{out,err}.log`
- Restart all: `sudo supervisorctl restart backend expo mongodb`
