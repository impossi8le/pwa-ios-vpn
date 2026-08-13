# ANALYTICS.md — Полная аналитика продукта

> Продукт: ТГ-бот продаёт OpenVPN-конфигурации, PWA (iOS + Android) отдаёт конфиг и включает VPN. Здесь — вся аналитика: какие события собираем, какие метрики считаем, как храним и визуализируем. Документ — **самодостаточная спецификация для LLM-реализации** (что реализовывать — описано явно).

---

## 1. Принципы аналитики

1. **События — основа.** Всё считается из событий (product analytics), не из счётчиков. Одна воронка, один источник правды.
2. **События из бэкенда (server-side) — источник правды.** Клиентские события (PWA) отправляем на наш бэкенд, бэкенд валидирует и пишет в аналитику. Клиент может врать — сервер нет.
3. **Идентификация по `user_id`** (Telegram user_id) и `session_id` (uuid, создаётся в PWA при первом заходе).
4. **Метрики — из событий через SQL**, единый словарь (имена событий и свойств фиксированы).
5. **PII-минимизация.** Не пишем реальный IP клиента в события; храним `anon_id` (хэш). IP — только в транзакционных таблицах БД.

---

## 2. Словарь событий (event dictionary)

Формат события — JSON:

```json
{
  "event": "order_paid",
  "distinct_id": "tg:123456789",
  "session_id": "5f0c...",
  "timestamp": "2026-08-13T10:00:00Z",
  "properties": {
    "platform": "ios",
    "plan": "monthly",
    "price_usd": 5.99,
    "config_id": "cfg_7f3a"
  }
}
```

### 2.1. Бэкенд-события (server-side, надёжные)

| Событие | Свойства | Когда |
|---|---|---|
| `bot_start` | `platform` | юзер запустил бота |
| `order_created` | `plan`, `price_usd`, `payment_method` | создан заказ |
| `order_paid` | `plan`, `price_usd`, `payment_method`, `config_id` | оплачен заказ |
| `order_failed` | `plan`, `payment_method`, `reason` | оплата не прошла |
| `config_issued` | `config_id`, `cn`, `expires_at` | выдан конфиг |
| `config_revoked` | `config_id`, `cn`, `reason` | отозван конфиг |
| `subscription_renewed` | `config_id`, `plan` | продлена подписка |
| `subscription_expired` | `config_id` | истекла подписка |
| `vpn_session_start` | `cn`, `config_id` | клиент подключился (из status-файла) |
| `vpn_session_end` | `cn`, `config_id`, `bytes`, `duration_s` | клиент отключился |
| `vpn_status_checked` | `platform`, `result` | PWA запросил статус (IP-сравнение) |

### 2.2. Клиентские события (PWA, через бэкенд)

| Событие | Свойства | Когда |
|---|---|---|
| `pwa_visit` | `platform`, `screen` | открыл PWA |
| `pwa_install_click` | `platform` | нажал «Установить» |
| `pwa_config_downloaded` | `platform`, `config_id` | скачал конфиг |
| `pwa_vpn_toggle` | `platform`, `action` (`on`/`off`), `method` | нажал «Включить/Выключить» |
| `pwa_status_poll` | `platform`, `result` | опросил статус |
| `pwa_error` | `platform`, `error_code` | ошибка в PWA |

---

## 3. Метрики (KPI) — как считать

Все метрики — SQL-запросы по событиям.

### 3.1. Активность (базовая)
- **DAU/WAU/MAU** — уникальные `distinct_id` за день/неделю/месяц (по любому событию).
- **Установки PWA** — `pwa_visit` впервые для `session_id`.
- **Активные VPN-сессии** — `vpn_session_start` минус `vpn_session_end` (мгновенно).

### 3.2. Продажи
- **Конверсия заказ → оплата** — `order_paid` / `order_created` по `payment_method`.
- **ARPU / ARPPU** — выручка / (все юзеры | платящие).
- **MRR, Churn** — по `subscription_renewed` / `subscription_expired`.
- **LTV** — сумма `order_paid.price_usd` по `distinct_id` до ухода.

### 3.3. Продукт (функциональность)
- **% установивших конфиг** — `pwa_config_downloaded` / `pwa_visit`.
- **% «включивших VPN»** — `vpn_session_start` / `order_paid` (активация).
- **Средняя длительность сессии VPN** — avg `duration_s`.
- **Частота включения** — `pwa_vpn_toggle` на юзера/день.
- **Доставка конфига (time-to-first-VPN)** — время от `order_paid` до первого `vpn_session_start`.

### 3.4. Инфраструктура
- **% успешных `vpn_status_checked`** — доля `result=ok`.
- **Ошибки генерации конфига** — `config_issued` / `config_issued`+`order_paid`.
- **Latency API** — p50/p95 (из логов pino / OTel).

---

## 4. Хранение и инструменты

### 4.1. Рекомендация для MVP: PostHog

- **PostHog** — product analytics из коробки: события, воронки, retention, дашборды, SQL (HogQL), **веб-аналитика PWA** (pageviews, авто-трекинг), и **MCP-доступ** (можно отдавать сводки в LLM-агентов).
- Режим для старта: **PostHog Cloud** (проще, бесплатно до 1 млн событий/мес). Self-host — Postgres 16 + Redis 7 + ClickHouse + ZooKeeper; **не масштабируется** за ~сотни тысяч событий/мес и требует ~6-8 ч/мес поддержки — берите self-host только если критичен compliance (данные не покидают сервер).
- SDK: `posthog-js` для PWA (или без него — события шлём через свой бэкенд) + `posthog-node` на сервере.

### 4.2. Альтернатива (если self-host по compliance)
- **ClickHouse** (OLAP-хранилище событий) + **Metabase** (BI, актуальная 63 / LTS 58) для дашбордов. События пишем в ClickHouse напрямую из бэкенда; дашборды в Metabase.
- Метрики те же (SQL по событиям), но воронки/retention — руками.

### 4.3. Инфраструктурные метрики (отдельно)
- **Prometheus + Grafana** — метрики сервера: CPU/RAM/диск, status OpenVPN (число клиентов), Redis, очередь BullMQ (задержка/длина), HTTP latency.

### 4.4. Версии (август 2026)
| Инструмент | Версия | Примечание |
|---|---|---|
| PostHog | Cloud / self-host (Postgres 16 + Redis 7 + ClickHouse + ZooKeeper) | self-host не для больших объёмов |
| Metabase | 63 (LTS 58) | BI-альтернатива |
| Grafana | 11.x | инфраструктурные дашборды |
| Prometheus | 3.x | сбор метрик |

---

## 5. Поток данных (server-side first)

```
PWA ──/api/events──► Бэкенд (валидация, distinct_id)
   │
   ├──► PostgreSQL: транзакционные данные (users, orders, configs, sessions)
   │
   └──► PostHog (Cloud/self): события → воронки, retention, дашборды
          (или ClickHouse + Metabase)

OpenVPN status-файл ──► Worker (BullMQ) ──► vpn_session_start/end ──► PostHog
```

**Правило:** все события проходят через наш бэкенд. Никаких прямых вызовов PostHog из PWA (клиент ненадёжен), кроме базовой pageview-аналитики (если решим через `posthog-js`).

---

## 6. Что реализовывать (checklist для LLM)

### MVP
- [ ] Эндпоинт `POST /api/events` (валидация, `distinct_id`, `session_id`, запись в PostHog)
- [ ] Словарь событий (раздел 2) как константы/типы
- [ ] Эмиттер событий в бэкенде (`services/analytics.ts`) — обёртка для `order_paid`, `config_issued`, `vpn_session_start/end`
- [ ] Worker «status-parser» → пишет `vpn_session_start/end` в аналитику
- [ ] PWA: отправка `pwa_visit`, `pwa_install_click`, `pwa_vpn_toggle`, `pwa_status_poll`
- [ ] PostHog: проект, дашборды «Продажи», «Продукт», «VPN-активность»
- [ ] Метрики 3.1–3.3 как сохранённые запросы/инсайты в PostHog

### Later
- [ ] Метрики инфраструктуры (Prometheus + Grafana): число активных клиентов OpenVPN, задержка очередей
- [ ] Retention-отчёт «установили → включили → продлили»
- [ ] Алерты: падение конверсии оплат, рост ошибок генерации конфигов

---

## 7. Открытые вопросы

1. **PostHog Cloud vs self-host** — решаем при выборе хостинга (РФ-аудитория может требовать self-host по закону о персональных данных).
2. **Приватность**: хранить ли `tg:user_id` открытым в событиях или хэшировать (по GDPR/152-ФЗ).
3. **Нужна ли pageview-аналитика через `posthog-js`** или хватает server-side событий.
4. **Каналы атрибуции** — откуда пришёл юзер (реферальные ссылки бота, QR, сайт).
