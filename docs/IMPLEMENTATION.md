# IMPLEMENTATION.md — План реализации (самодостаточный, для LLM-агентов)

> Документ — **полная спецификация для LLM-реализации**: содержит все входные данные, правила и критерии готовности. LLM-агент, читающий только этот файл (+ связанные `PLAN/ARCHITECTURE/RISKS/PRODUCT/NATIVE-APPS/BACKEND/ANALYTICS`), может реализовать продукт без устных пояснений. Продукт: ТГ-бот продаёт OpenVPN-конфиги; PWA = лендинг + кабинет (VPN-флоу в PWA — только iOS/macOS); Android — Flutter-клиент; бэкенд — ядро.

---

## 0. Цель и скоуп MVP

**Продукт** (см. [PRODUCT.md](PRODUCT.md) — эталон формы): ТГ-бот продаёт OpenVPN-конфиги; **PWA** = публичный лендинг + личный кабинет; VPN-флоу в PWA — только **iOS/macOS**; **Android** — Flutter-клиент (commuspaceVPN); Windows/Linux — кабинет в PWA + `.ovpn` вручную. Бэкенд — ядро. Полная аналитика.

**MVP**: лендинг → регистрация/вход (email+пароль, привязка к Telegram) → оплата (Stars в боте + веб-оплата в кабинете) → генерация OpenVPN-конфига → установка/включение (iOS/macOS в PWA; Android в Flutter-клиенте) → статус VPN → аналитика.

**Вне MVP**: n8n-автоматизации (внешний API, позже), Windows/Linux нативные приложения, WireGuard, многоязычность, реселлеры.

**Архитектура**: [PRODUCT.md](PRODUCT.md) + [BACKEND.md](BACKEND.md) + [NATIVE-APPS.md](NATIVE-APPS.md) + [PLAN.md](PLAN.md) + [ANDROID.md](ANDROID.md) (deprecated, исследование) + [ANALYTICS.md](ANALYTICS.md).

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
| Автоматизации | n8n | 2.29+ (внешний) |
| Нативный клиент | **Flutter** + `openvpn_flutter` | SDK ≥**3.44**, Dart ≥**3.12**, плагин **1.3.4** |
| Движки OpenVPN | ics-openvpn (Android) / OpenVPNAdapter (iOS) | GPL; уже в [commuspaceVPN](https://github.com/impossi8le/commuspaceVPN) |
| iOS-деплой | **TestFlight** (не App Store) | 90 дней на сборку, до 10k тестировщиков |

---

## 2. Дерево проекта

```
/
  /backend
    /src
      /api            # Fastify routes + auth
      /services       # бизнес-логика (orders, configs, vpn-status, billing, accounts)
      /openvpn        # коннектор к серверу (status-файл, management, easyrsa)
      /telegram       # бот: webhook/long polling, команды, платежи, привязка
      /queues         # BullMQ: worker'ы (config.generate, cert.revoke, status.poll, notify.expiry)
      /n8n            # HTTP-клиент к внешнему n8n (по требованию)
      /workers        # фоновые задачи
      /analytics      # эмиттер событий в PostHog
      /config, /logger, /db, /metrics
    /prisma           # schema.prisma, миграции
    /.env.example
  /pwa
    /src              # React + Vite + vite-plugin-pwa
      /pages          # лендинг (/), кабинет (/app), VPN-экран (iOS/macOS)
  /client             # НАТИВНЫЙ КЛИЕНТ — уже реализован (commuspaceVPN, Flutter)
    /example/lib      # openvpn_flutter: screens, services, models
    /ios              # Runner + VPNExtension (OpenVPNAdapter)
    /android          # ics-openvpn
    /example/windows  # Windows runner (stub VPN)
    /fastlane         # (добавить) TestFlight: Fastfile, Appfile
  /infra              # docker-compose, OpenVPN-конфиги, systemd
  /docs               # этот и связанные документы
```

---

## 3. Схема данных (Prisma)

```prisma
model User {
  id          String   @id @default(cuid())
  email       String?  @unique                 // основной логин (email+пароль)
  passwordHash String?                         // argon2, nullable пока нет пароля
  phone       String?                          // зарезервировано (2FA/поддержка)
  tgId        BigInt?  @unique                 // Telegram user_id (не у всех)
  tgUsername  String?
  createdAt   DateTime @default(now())
  orders      Order[]
  sessions    VpnSession[]
  appTokens   AppToken[]
}

model AppToken {                               // для нативных приложений
  id        String   @id @default(cuid())
  userId    String
  tokenHash String   @unique                   // храним только hash
  name      String?                            // «Android (Pixel 8)»
  revokedAt DateTime?
  createdAt DateTime @default(now())
}

model Order {
  id            String    @id @default(cuid())
  userId        String
  plan          String    // daily | monthly | quarterly | yearly
  priceUsd      Decimal
  paymentMethod String    // stars | crypto | fiat
  channel       String?   // bot | web
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
  mobileconfig String?          // содержимое .mobileconfig (для iOS/macOS)
  expiresAt DateTime
  revokedAt DateTime?
  createdAt DateTime  @default(now())
}

model VpnSession {
  id         String   @id @default(cuid())
  cn         String
  configId   String?
  realIp     String?
  source     String?  // status | app
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

### 4.1. Auth (email+пароль + Telegram)
```
POST /api/auth/register               # email + пароль → создаёт User, JWT
POST /api/auth/login                  # email + пароль → JWT (access + refresh)
POST /api/auth/tg-login               # Telegram Login Widget (HMAC) → вход/привязка
POST /api/auth/link-tg                # код привязки из бота → привязка tg к аккаунту
POST /api/auth/refresh                # refresh → новый access
```

### 4.2. Кабинет (auth: JWT / app-токен)
```
POST /api/events                    # приём аналитических событий из PWA
GET  /api/configs                   # список купленных конфигов юзера
GET  /api/configs/:id/download      # отдать .ovpn / .mobileconfig
GET  /api/vpn/status                # статус: включён/выключен + чей конфиг + сессии
POST /api/pwa/log                   # ошибки PWA
POST /api/orders                    # создать заказ из кабинета (веб-оплата)
GET  /api/orders/:id/payurl         # ссылка/QR веб-оплаты (crypto/fiat)
```

### 4.3. Нативное приложение (auth: app-токен)
```
POST /api/app/token                 # создать app-токен (из кабинета, с паролем)
POST /api/vpn/session               # репорт сессии из приложения: start/stop, bytes
GET  /api/app/me                    # профиль для приложения
```

### 4.4. Бот (webhook или long polling)
```
/start, /help                        # приветствие, инструкция
/buy <plan>                          # создать заказ
/pay <order_id>                      # оплата (Telegram Stars)
/my                                  # мои конфиги
/status                              # статус подписки
/renew, /cancel
/link                                # сгенерировать код привязки к PWA-аккаунту
```

### 4.5. Ответы — JSON, ошибки — RFC 7807 (`{ "error": { "code", "message" } }`).

---

## 5. Состояние VPN (повтор ключевого механизма)

- **«Включён ли VPN»**: `GET /api/vpn/status` — бэкенд берёт IP клиента из соединения (или XFF с trusted_proxies), сравнивает с IP VPN-сервера. Равны → включён.
- **«Чей конфиг»**: из status-файла OpenVPN (`--status /var/run/openvpn-status.log 5`, формат `status 3`), парсим `Common Name` → сопоставляем с `Config.cn` → `Order` → `User`.
- **Сессии**: Worker `status.poll` (BullMQ, раз в 5–30 с) парсит status-файл, пишет `VpnSession` + события `vpn_session_start/end`.
- **Пограничные случаи**: IPv6 (проверять v4+v6), split tunneling, CDN/XFF, двойной VPN → статус «unconfirmed». Подробно — [BACKEND.md](BACKEND.md) §2.3.
- **Нативное приложение (Flutter)**: локальный статус через движок (ics-openvpn / OpenVPNAdapter) + репорт `POST /api/vpn/session`. Подробно — [NATIVE-APPS.md](NATIVE-APPS.md).

### 5.5. Роутинг в PWA (кто что видит)

```
/                  лендинг (публичный): оффер, тарифы, FAQ, CTA «Купить в боте» / «Войти»
/app               кабинет (auth): подписка, конфиги, оплата, статистика
/app/vpn           VPN-экран — первый, если (iOS|macOS) ∧ авторизован ∧ активная подписка
                     · iOS:   «Установить» .mobileconfig + «Включить/Выключить» (shortcuts://)
                     · macOS: «Установить» .mobileconfig + инструкция меню-бар + «Проверить статус»
                   иначе: домашняя страница кабинета
Внутри нативного приложения (?embed=app): скрыть «Скачать приложение», «Установить PWA», «Войти»
```

Правило подробно — [PRODUCT.md](PRODUCT.md) §2. Детекция платформы по userAgent + ручной переключатель в настройках.

---

## 6. События аналитики (что эмитить)

См. [ANALYTICS.md](ANALYTICS.md) §2. Минимум для MVP:
`landing_visit`, `auth_register`, `auth_login`, `tg_linked`, `order_created`, `order_paid`, `config_issued`, `vpn_session_start`, `vpn_session_end`, `vpn_status_checked`, `pwa_vpn_toggle`, `pwa_status_poll`, `app_install`, `app_vpn_connect`.

Эмиттер — `services/analytics.ts`, пишет в PostHog через `posthog-node`. Server-side first. События приложения (Android) приходят через `POST /api/vpn/session`/`POST /api/events` и перенаправляются в PostHog.

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
- [ ] Telegram bot: webhook/long polling, команды `/start /buy /pay /my /status /renew /cancel /link`
- [ ] Оплата Telegram Stars (или крипта/фиат по решению)
- [ ] `order_paid` → генерация конфига (Worker `config.generate`)
- [ ] Код привязки аккаунта к Telegram (`/link` → код → ввод в кабинете)

### Фаза 2 — Генерация конфигов (1 день)
- [ ] Конвертер `.ovpn` → `.mobileconfig` (VendorConfig, VPNSubType) для iOS/macOS
- [ ] `.ovpn` с уникальным CN, `auth-user-pass`, сроком действия
- [ ] Отзыв/продление (Worker `cert.revoke`, `notify.expiry`)

### Фаза 3 — PWA: лендинг + кабинет (3 дня)
- [ ] **Лендинг** `/`: статический, SEO (meta/OG), тарифы, FAQ, CTA «Купить в боте» / «Войти»
- [ ] **Аккаунты**: регистрация/вход email+пароль (argon2), JWT, refresh
- [ ] **Привязка Telegram**: Telegram Login Widget + код из бота
- [ ] **Кабинет** `/app`: подписка, конфиги, оплата, статистика
- [ ] **Веб-оплата** в кабинете (crypto/fiat) — `POST /api/orders`, `GET payurl`
- [ ] **Роутинг**: VPN-экран первым для iOS/macOS при активной подписке; ручной переключатель платформы; `?embed=app` скрывает установочные блоки
- [ ] React + Vite + vite-plugin-pwa (manifest, service worker, установка на Home)

### Фаза 4 — PWA: VPN-экран iOS/macOS (1 день)
- [ ] «Установить»: `.mobileconfig` (iOS) / инструкция меню-бар (macOS)
- [ ] «Включить/Выключить»: iOS `shortcuts://run-shortcut` (Set VPN) — только iOS; macOS — инструкция + «Проверить статус»
- [ ] Статус VPN: `GET /api/vpn/status`, поллинг раз в 10 с

### Фаза 5 — Состояние VPN (1 день)
- [ ] Worker `status.poll`: парсер status-файла → `VpnSession` + события
- [ ] `GET /api/vpn/status`: IP-сравнение + CN-маппинг + пограничные случаи (IPv6, unconfirmed)

### Фаза 6 — Аналитика (1 день)
- [ ] `POST /api/events` + эмиттер в PostHog
- [ ] События §6, дашборды «Продажи», «Продукт», «VPN-активность», «Лендинг»
- [ ] Метрики: конверсия оплат, активация, MRR/churn (базово)

### Фаза 7 — Нативный клиент + iOS TestFlight (2 дня, MVP-параллельно)
> Клиент уже реализован ([commuspaceVPN](https://github.com/impossi8le/commuspaceVPN), Flutter). Здесь — интеграция и iOS-деплой вне сторов.
- [ ] **Auth на наш бэкенд**: поднять совместимые `/api/auth/register|login|validate|logout` → сменить `AuthConfig.baseUrl` (`--dart-define=AUTH_API_BASE`)
- [ ] **Конфиги из кабинета**: `GET /api/configs` вместо ручного импорта файлом/URL
- [ ] **Репорт сессий**: `POST /api/vpn/session` при connected/disconnected (`vpn_session_start/end`, source=`app`)
- [ ] **401/403 → разрыв туннеля** (отзыв/истечение подписки)
- [ ] **iOS TestFlight**: Apple Developer, App ID (Network Extensions + App Groups), `flutter build ios`, archive → TestFlight
- [ ] **Добавление тестировщиков**: Internal (роли в App Store Connect) / External (Apple ID); автоматизация через **fastlane pilot**
- [ ] Подробно — [NATIVE-APPS.md](NATIVE-APPS.md) §3–§5

### Фаза 8 — Запуск (0.5 дня)
- [ ] HTTPS (Caddy/Traefik), env-конфиг, systemd/docker для бота
- [ ] Смоук-тест на реальных устройствах (iOS 16.4+, macOS, Android 13+)
- [ ] Алерты (логи ошибок, метрики)

---

## 9. Критерии готовности MVP (acceptance)

- [ ] Лендинг `/` открывается, SEO-теги на месте, CTA ведут в бота и на регистрацию
- [ ] Регистрация/вход (email+пароль), привязка Telegram (виджет + код `/link`)
- [ ] Юзер покупает конфиг (бот Stars ИЛИ веб-оплата в кабинете) → получает `.ovpn` + `.mobileconfig`
- [ ] Роутинг: iOS/macOS с активной подпиской → VPN-экран первым; остальные платформы → кабинет
- [ ] iOS: «Установить» (.mobileconfig) + «Включить» (shortcuts://) работают
- [ ] iOS-клиент: TestFlight-сборка залита, тестировщики добавлены (Internal/External), подключение работает
- [ ] macOS: «Установить» (.mobileconfig) + инструкция меню-бар + статус по IP
- [ ] Flutter-клиент (Android): вход через наш бэкенд → конфиги из кабинета → подключение через ics-openvpn → локальный статус → репорт сессии
- [ ] `GET /api/vpn/status` корректно показывает «включён/выключен/чей конфиг» (тест на реальном сервере)
- [ ] Аналитика: события из §6 идут в PostHog, дашборды построены
- [ ] Пограничные случаи состояния VPN (§5) учтены: IPv6, двойной VPN → «unconfirmed»
- [ ] CI зелёный, DoD из §7 соблюдены

---

## 10. Открытые вопросы (решить перед реализацией)

1. **Веб-оплата**: CryptoBot vs ЮKassa vs другой провайдер (для кабинета). Влияет на `Order.paymentMethod` (`crypto`/`fiat`).
2. **PostHog Cloud vs self-host** (РФ-аудитория — 152-ФЗ).
3. **Long polling vs webhook** для бота (нужен ли публичный HTTPS для webhook; для MVP подходит long polling).
4. **Продление подписки**: `easyrsa renew` vs перевыпуск конфига.
5. **Лимит одновременных подключений** на один конфиг: `--duplicate-cn off` (default) или CCD.
6. **Лицензия клиента**: приложение GPL по построению (линкует ics-openvpn/OpenVPNAdapter). Репозиторий уже публичный, LICENSE GPLv3 — принято как факт; коммерческая логика — на бэкенде (см. [NATIVE-APPS.md](NATIVE-APPS.md) §2).
7. **Хранение паролей**: email+пароль (argon2) — необходимость подтверждения email / восстановления пароля на старте.
8. **iOS-деплой**: TestFlight (90 дней на сборку) — нужно регулярно заливать новые сборки; или ad-hoc (до 100 UDID). Apple Developer $99/год.
