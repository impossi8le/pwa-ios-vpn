# IMPLEMENTATION.md — План реализации (самодостаточный, для LLM-агентов)

> Документ — **полная спецификация для LLM-реализации**: содержит все входные данные, правила и критерии готовности. LLM-агент, читающий только этот файл (+ связанные `PLAN/ARCHITECTURE/RISKS/ANDROID/BACKEND/ANALYTICS`), может реализовать продукт без устных пояснений. Продукт: ТГ-бот продаёт OpenVPN-конфигурации, PWA раздаёт конфиги и включает VPN на iOS/Android, бэкенд — ядро.

---

## 0. Цель и скоуп MVP

**MVP**: ТГ-бот → оплата → генерация OpenVPN-конфига → PWA показывает купленные конфиги → пользователь устанавливает и включает VPN → бэкенд показывает статус (включён/выключен/чей конфиг). Полная аналитика.

**Вне MVP**: n8n-автоматизации (внешний API, подключается позже), нативная обёртка TWA, поддержка WireGuard, многоязычность, реселлеры.

**Архитектура**: [BACKEND.md](BACKEND.md) (стек, модульность, состояние VPN) + [ANDROID.md](ANDROID.md) + [PLAN.md](PLAN.md) + [ANALYTICS.md](ANALYTICS.md).

---

## 1. Стек и версии (обязательно для LLM)

| Слой | Технология | Версия |
|---|---|---|
| Рантайм | Node.js | **24 LTS** |
| Язык | TypeScript | **5.x** (strict) |
| Фреймворк бэкенда | Fastify | **5.x** |
| PWA | React + Vite | React **19**, Vite **8** |
| PWA-плагин | vite-plugin-pwa | **1.3** |
| БД | PostgreSQL | **18** |
| ORM | Prisma | **7** |
| Очереди | BullMQ | **6** |
| Кэш/брокер | Redis | **8.10** |
| Бот | Telegram Bot API | **10.2** |
| Логи | pino | **10** |
| Трассировка | OpenTelemetry | GA |
| Аналитика | PostHog | Cloud (или self-host) |
| Автоматизации | n8n | 2.29+ (внешний, не трогаем) |

---

## 2. Дерево проекта

```
/
  /backend
    /src
      /api            # Fastify routes + auth
      /services       # бизнес-логика (orders, configs, vpn-status, billing)
      /openvpn        # коннектор к серверу (status-файл, management, easyrsa)
      /telegram       # бот: webhook/long polling, команды, платежи
      /queues         # BullMQ: worker'ы (config.generate, cert.revoke, status.poll, notify.expiry)
      /n8n            # HTTP-клиент к внешнему n8n (по требованию)
      /workers        # фоновые задачи
      /analytics      # эмиттер событий в PostHog
      /config, /logger, /db, /metrics
    /prisma           # schema.prisma, миграции
    /.env.example
  /pwa
    /src              # React + Vite + vite-plugin-pwa
  /infra              # docker-compose, OpenVPN-конфиги, systemd
  /docs               # этот и связанные документы
```

---

## 3. Схема данных (Prisma)

```prisma
model User {
  id          String   @id @default(cuid())
  tgId        BigInt   @unique                 // Telegram user_id
  tgUsername  String?
  email       String?
  createdAt   DateTime @default(now())
  orders      Order[]
  sessions    VpnSession[]
}

model Order {
  id            String    @id @default(cuid())
  userId        String
  plan          String    // daily | monthly | quarterly | yearly
  priceUsd      Decimal
  paymentMethod String    // stars | crypto | fiat
  status        String    // created | paid | failed | refunded
  paidAt        DateTime?
  config        Config?
  createdAt     DateTime  @default(now())
}

model Config {
  id        String    @id @default(cuid())
  orderId   String    @unique
  cn        String    @unique   // Common Name = cert name = имя в CCD
  ovpn      String?             // содержимое .ovpn
  mobileconfig String?          // содержимое .mobileconfig (для iOS)
  expiresAt DateTime
  revokedAt DateTime?
  createdAt DateTime  @default(now())
}

model VpnSession {
  id         String   @id @default(cuid())
  cn         String
  configId   String?
  realIp     String?
  connectedAt DateTime
  disconnectedAt DateTime?
  bytesIn    BigInt?
  bytesOut   BigInt?
}

// events → PostHog (не в БД; при необходимости — таблица events для резерва)
```

**Правила:**
- `cn` = уникальный идентификатор конфига (CN сертификата), привязан к `Order`.
- Срок действия = срок подписки (`expiresAt`). Отзыв: `revokedAt` + `easyrsa revoke` + CRL + `kill` активных.

---

## 4. API (Fastify, версии)

### 4.1. Публичные (auth: Telegram initData)
```
POST /api/events                    # приём аналитических событий из PWA
GET  /api/configs                   # список купленных конфигов юзера
GET  /api/configs/:id/download      # отдать .ovpn / .mobileconfig
GET  /api/vpn/status                # статус: включён/выключен + чей конфиг + сессии
POST /api/pwa/log                   # ошибки PWA
```

### 4.2. Бот (webhook или long polling)
```
/start, /help                        # приветствие, инструкция
/buy <plan>                          # создать заказ
/pay <order_id>                      # оплата (Telegram Stars)
/my                                  # мои конфиги
/status                              # статус подписки
/renew, /cancel
```

### 4.3. Ответы — JSON, ошибки — RFC 7807 (`{ "error": { "code", "message" } }`).

---

## 5. Состояние VPN (повтор ключевого механизма)

- **«Включён ли VPN»**: `GET /api/vpn/status` — бэкенд берёт IP клиента из соединения (или XFF с trusted_proxies), сравнивает с IP VPN-сервера. Равны → включён.
- **«Чей конфиг»**: из status-файла OpenVPN (`--status /var/run/openvpn-status.log 5`, формат `status 3`), парсим `Common Name` → сопоставляем с `Config.cn` → `Order` → `User`.
- **Сессии**: Worker `status.poll` (BullMQ, раз в 5–30 с) парсит status-файл, пишет `VpnSession` + события `vpn_session_start/end`.
- **Пограничные случаи**: IPv6 (проверять v4+v6), split tunneling, CDN/XFF, двойной VPN → статус «unconfirmed». Подробно — [BACKEND.md](BACKEND.md) §2.3.

---

## 6. События аналитики (что эмитить)

См. [ANALYTICS.md](ANALYTICS.md) §2. Минимум для MVP:
`order_paid`, `config_issued`, `vpn_session_start`, `vpn_session_end`, `vpn_status_checked`, `pwa_visit`, `pwa_install_click`, `pwa_vpn_toggle`, `pwa_status_poll`.

Эмиттер — `services/analytics.ts`, пишет в PostHog через `posthog-node`. Server-side first.

---

## 7. Правила реализации (DoD — Definition of Done)

1. **Каждый модуль** покрыт юнит-тестами (Vitest) + критические пути — интеграционными.
2. **Логирование**: pino, структурированное. Каждый запрос — `req_id`. Ключевые события бизнес-логики логируются (`order_paid`, `config_issued`, `vpn_session_*`).
3. **Ошибки**: Fastify-обработчики, RFC 7807, централизованный логер, retry с backoff в BullMQ.
4. **Безопасность**: валидация входов (zod), auth через Telegram initData (HMAC-SHA256), секреты в env, никаких секретов в git.
5. **Схема БД**: миграции через Prisma, idempotent.
6. **PWA**: офлайн (service worker), установка на экран Домой, детект iOS/Android.
7. **OpenVPN**: конфиг с `push "redirect-gateway ipv6"` (закрыть IPv6-утечку), `crl-verify`, уникальный CN на заказ.
8. **Аналитика**: все события из §6 эмитятся, дашборды PostHog созданы.
9. **CI**: `npm run lint`, `npm run test`, `npm run build` проходят.
10. **Документация**: ключевые решения зафиксированы в `docs/` (обновить при изменении архитектуры).

---

## 8. План по фазам (для LLM-агента)

### Фаза 0 — Инфраструктура (0.5 дня)
- [ ] docker-compose: Postgres 18, Redis 8.10, (PostHog — Cloud или self)
- [ ] Fastify-скелет + pino + zod + Prisma (подключение, миграция)
- [ ] OpenVPN-сервер: статус-файл, management, easyrsa, CRL, IPv6

### Фаза 1 — Бот + оплата (1.5 дня)
- [ ] Telegram bot: webhook/long polling, команды `/start /buy /pay /my /status /renew /cancel`
- [ ] Оплата Telegram Stars (или крипта/фиат по решению)
- [ ] `order_paid` → генерация конфига (Worker `config.generate`)

### Фаза 2 — Генерация конфигов (1 день)
- [ ] Конвертер `.ovpn` → `.mobileconfig` (VendorConfig, VPNSubType) для iOS
- [ ] `.ovpn` с уникальным CN, `auth-user-pass`, сроком действия
- [ ] Отзыв/продление (Worker `cert.revoke`, `notify.expiry`)

### Фаза 3 — PWA (2 дня)
- [ ] React + Vite + vite-plugin-pwa (manifest, service worker)
- [ ] Личный кабинет: список конфигов, кнопки «Установить» (iOS `.mobileconfig`, Android `intent:`)
- [ ] «Включить/Выключить» (iOS `shortcuts://`, Android Always-on инструкция / MacroDroid)
- [ ] Статус VPN: `GET /api/vpn/status`, поллинг раз в 10 с

### Фаза 4 — Состояние VPN (1 день)
- [ ] Worker `status.poll`: парсер status-файла → `VpnSession` + события
- [ ] `GET /api/vpn/status`: IP-сравнение + CN-маппинг + пограничные случаи (IPv6, unconfirmed)

### Фаза 5 — Аналитика (1 день)
- [ ] `POST /api/events` + эмиттер в PostHog
- [ ] События §6, дашборды «Продажи», «Продукт», «VPN-активность»
- [ ] Метрики: конверсия оплат, активация, MRR/churn (базово)

### Фаза 6 — Запуск (0.5 дня)
- [ ] HTTPS (Caddy/Traefik), env-конфиг, systemd/docker для бота
- [ ] Смоук-тест на реальных устройствах (iOS 16.4+, Android 13+)
- [ ] Алерты (логи ошибок, метрики)

---

## 9. Критерии готовности MVP (acceptance)

- [ ] Юзер покупает конфиг в боте → получает `.ovpn` + `.mobileconfig` автоматически
- [ ] PWA показывает конфиги, кнопка «Установить» работает на iOS и Android
- [ ] Кнопка «Включить» запускает VPN (iOS через Shortcuts, Android через Always-on/автоматизатор)
- [ ] `GET /api/vpn/status` корректно показывает «включён/выключен/чей конфиг» (тест на реальном сервере)
- [ ] Аналитика: события из §6 идут в PostHog, дашборды построены
- [ ] Пограничные случаи состояния VPN (§5) учтены: IPv6, двойной VPN → «unconfirmed»
- [ ] CI зелёный, DoD из §7 соблюдены

---

## 10. Открытые вопросы (решить перед реализацией)

1. **Оплата**: Telegram Stars (просто, но комиссия) vs крипта vs фиат. Влияет на `Order.paymentMethod`.
2. **PostHog Cloud vs self-host** (РФ-аудитория — 152-ФЗ).
3. **Long polling vs webhook** для бота (нужен ли публичный HTTPS для webhook; для MVP подходит long polling).
4. **Продление подписки**: `easyrsa renew` vs перевыпуск конфига.
5. **Лимит одновременных подключений** на один конфиг: `--duplicate-cn off` (default) или CCD.
