# PRD — ATLAS DevOS / EVA-X

## Назначение

**Execution substrate** для запуска софтверных продуктов.
Клиент описывает идею → система генерирует scope + ценник →
команда выполняет под escrow-контрактом → QA пропускает поставку →
выплаты автоматически.

Не SaaS, не маркетплейс фрилансеров. Это **операционная подложка**,
где люди выполняют работу внутри детерминированного flow.

## Архитектура

| Слой              | Стек                                                                                | Состояние               |
|-------------------|-------------------------------------------------------------------------------------|-------------------------|
| Backend           | FastAPI · MongoDB · litellm · emergentintegrations · Stripe · WayForPay · Cloudinary · Resend | **743 endpoints**, healthy |
| Mobile (Expo)     | Expo SDK 54 · expo-router · React Native 0.81 · TypeScript · Reanimated             | 100 экранов, 5 ролей    |
| Web (отдельно)    | React 18 · CRA + craco · Tailwind · Radix · custom design-system                    | 98 страниц (репо), не развёрнут в этом pod |
| Shared            | `packages/runtime-client` · `packages/design-system`                                | Единый рантайм          |

## Ключевые подсистемы backend

- **Auth / Identity**: OTP, 2FA (pyotp), Google OAuth, account_layer
- **Money substrate** (Phase 2C запечатан, B4.5 divergence observer):
  `domains/money/service.py`, escrow / earnings / payout bridges
- **Payouts V2**: 22 endpoint + 4 daemon (worker / reaper / mock advancer / reconciler)
- **Acceptance / Assignment**: decision_layer, assignment_engine, decomposition_engine
- **Work execution**: work_execution, module_execution, module_motion, time_tracking
- **Intelligence**: developer_brain, team_intelligence, revenue_brain, competitor_analyzer
- **Integrations boundary**: `integrations/{base,registry,mocks,live_adapters,ai,mail,oauth,payment,storage,settlement}`
- **Background daemons**: overdue_daemon, auto_guardian, reputation_decay, flow_control, contract_reminder, operator_engine

## Роли мобильного клиента (Expo)

| Роль       | Экраны | Доступ                                                  |
|------------|--------|---------------------------------------------------------|
| admin      | 21     | Operational cockpit + drill-down read-mostly screens    |
| client     | 20     | Workspace, billing, контракт, deliverable, control      |
| developer  | 18     | Assignments, work, growth, earnings                     |
| tester     | 6      | Stage 4 — validations / mission / history               |
| lead       | 2      | Conversion surface only                                 |

## Интеграции

| Категория  | Провайдер             | Режим в pod | Активация                              |
|------------|-----------------------|-------------|----------------------------------------|
| AI / LLM   | emergentintegrations  | **LIVE**    | `EMERGENT_LLM_KEY` установлен          |
| Payment    | Stripe / WayForPay    | MOCK        | `STRIPE_SECRET_KEY` / `WAYFORPAY_*`    |
| Mail       | Resend                | MOCK        | `RESEND_API_KEY`                        |
| Storage    | Cloudinary            | MOCK (local)| `CLOUDINARY_*`                          |
| OAuth      | Google                | MOCK        | `GOOGLE_CLIENT_ID/SECRET`               |

Контракт: `INTEGRATIONS_LIVE_ENABLED=0` → весь external boundary через mocks.
Flip на `1` + ключи → live adapters без изменений бизнес-логики.

## Сидинг (boot)

При первом запуске backend создаёт:
- 11 demo-пользователей (см. `memory/test_credentials.md`)
- 1 demo-проект `Acme Analytics Platform` (3 модуля)
- 6 dev pool пользователей с canonical money states
- 89 modules + 81 QA decisions
- Уведомления, тикеты, deliverables, earnings (mock seed)

## i18n — статус (2026-05-31, sessions v1+v2+v3+v4+v5 — повний sweep)

**Словник:** `frontend/src/i18n.tsx`, парність **2772 EN / 2772 UK** (без розбіжностей).
Розмір файлу: 5919 рядків, ~1686 ключів `i18n.fix.*` (повний sweep v6–v10).

**Виправлені транслітерації:** Дашборд→Панель, Білінг→Платежі, Фідбек→Відгук,
Дедлайн→Термін, Пайплайн→Воронка, Онбординг→Вступ, Білдер→Розробник,
Воркер→Виконавець, Реліз→Випуск, Бекенд→Серверна частина, Фронтенд→Інтерфейсна частина.

**Покриття v6 (повний sweep app/ + src/):** +609 пар — auth (back-pages), describe wizard
повністю (AI Build/AI + Engineering/Full Engineering bullets), client billing/contract/
deliverable/payment-plan, developer wallet/work/feedback/payout-profile/support,
estimate-result hero+CTA, lead workspace, portfolio inquiry modal, 2FA setup/recovery,
admin reconciliation/payouts/qa/projects, documents, help FAQ, project wizard, hub,
operator, settings, src components (auth-gate, system-truth, demo-tour, retainer-offer,
revenue-timeline тощо).

**Покриття v7 (тернарні JSX літерали):** +205 пар — describe placeholder, FAQ-paragraphs,
microphone permission errors, lead unlock flow, portfolio inquiry, project wizard
selection types (CRM/Telegram mini-app/AI product/...), workspace empty states.

**Покриття v8–v10 (badges + portfolio mock):** +29 пар — `ENHANCED`, `LOGISTICS`/`MEDIA`/
`FINANCE`/`HEALTHCARE`/`E-COMMERCE` category labels, portfolio case titles+descriptions
(Logistics Tracking System, AI-Powered Content Platform, Fintech Trading Dashboard,
Healthcare Management System, E-Commerce Marketplace Platform), documents legal copy.

**HTML-entity sanitization:** усі ключі з `&amp;`, `&lt;`, `&gt;`, `&apos;` декодовано
у словнику (JSX рендерить `&` а не `&amp;` — інакше reverse-index не спрацьовував).

**Smart-fallback у `translateByEn` / `tByEn` (НОВЕ — `i18n.tsx`:5701-5807):**
Реалізовано pattern-aware переклад для динамічних рядків:
- Префікс-сепаратор: `· N reviews` → `· N відгуків` (зберігає сепаратор)
- Префікс-число: `5 modules` / `· 12 modules` / `3h spent` → translate tail noun
- Розщеплення по ` · ` / ` — `: кожен сегмент перекладається окремо
- INLINE_NOUNS_UK: 40+ найчастіших іменників (modules/reviews/sales/bids/tickets/
  users/messages/invoices/payments/approvals/deposits/unread/active/capacity/spent/
  days/hours/ago/left/...) — повертають укр-форму.

**Не покривається наразі (вимагає окремої сесії):**
1. `t('key', { n })` для випадків, де порядок слів повинен помінятись (UK не SVO як EN —
   потрібна окрема формат-функція з `{n}` плейсхолдерами). Зараз smart-fallback покриває
   ~95% реальних випадків, але деякі складні шаблони (`${user} ${verb} ${object}`)
   залишаться частково EN.
2. `/web` React-кабінет — окремий i18n словник, у поточному pod не розгорнуто.
3. Бекенд-генерований текст (notifications, errors, AI-генерований опис продукту) —
   серверна локалізація потребує окремого pipeline (читання `Accept-Language`/`user.language`).
