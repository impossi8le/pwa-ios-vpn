# PWA + Нативные приложения → OpenVPN

Продукт: **ТГ-бот продаёт OpenVPN-конфигурации**. Пользователь заходит на **PWA-сайт** (лендинг + личный кабинет), покупает подписку, устанавливает VPN-профиль и включает туннель. VPN-управление в PWA — на **iOS и macOS**; на **Android** — через нативное приложение (встроенный кабинет + OpenVPN-модуль); Windows/Linux — кабинет в PWA + `.ovpn` вручную (нативные приложения позже).

> **Реализация**: весь продукт (бэкенд, PWA, аналитика) реализуется LLM-агентом по самодостаточным спецификациям ниже — см. [IMPLEMENTATION.md](docs/IMPLEMENTATION.md).

## Форма продукта

```
PWA
├── /            Лендинг (публичный): оффер, тарифы, FAQ, «Купить в боте» / «Войти»
└── /app         Кабинет (авторизованный)
      ├── iOS / macOS   → VPN-экран первым (установка профиля, вкл/выкл, статус)
      └── Android/Win/Linux → кабинет без VPN-экрана: подписка, оплата, «Скачать приложение»

Нативное приложение (Android, MVP)
├── WebView → /app (кабинет, авторизация по app-токену)
└── Нативный OpenVPN-модуль: подключение, локальный статус, репорт сессий в бэкенд
```

**Правило роутинга**: VPN-экран открывается первым, только если `(iOS | macOS) ∧ авторизован ∧ активная подписка`. Подробно — [PRODUCT.md](docs/PRODUCT.md).

## Содержимое

| Файл | О чём |
|------|-------|
| [PRODUCT.md](docs/PRODUCT.md) | **Форма продукта**: лендинг + кабинет + нативные приложения, правило роутинга, аккаунты, оплата |
| [PLAN.md](docs/PLAN.md) | iOS: полное техническое обоснование (почему это работает) + пошаговый план |
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | iOS: схема взаимодействия компонентов и механика по шагам |
| [RISKS.md](docs/RISKS.md) | iOS: ограничения и риски (Stolen Device Protection, 8 минут, Shortcuts, App Store 5.4, РФ) |
| [ANDROID.md](docs/ANDROID.md) | **DEPRECATED** — исследование intent-механики PWA↔OpenVPN (заменено нативным приложением) |
| [NATIVE-APPS.md](docs/NATIVE-APPS.md) | **Нативные приложения**: Android (WebView кабинета + OpenVPN-модуль), лицензии, Windows/Linux позже |
| [BACKEND.md](docs/BACKEND.md) | Бэкенд: состояние VPN (IP-сравнение + CN), аккаунты/app-токены, стек, модульность, n8n как внешний API |
| [ANALYTICS.md](docs/ANALYTICS.md) | Полная аналитика: события, KPI, воронка лендинга, хранение (PostHog/ClickHouse+Metabase) |
| [IMPLEMENTATION.md](docs/IMPLEMENTATION.md) | **Самодостаточный план для LLM-агента**: Prisma, API, фазы, DoD, критерии готовности |

## Состояние VPN

- **iOS/macOS (PWA)** — из браузера статус не прочитать: PWA спрашивает бэкенд, тот видит реальный IP клиента и сверяет с IP VPN-сервера. «Чей конфиг активен» — по **Common Name** из status-файла OpenVPN.
- **Android (нативное приложение)** — статус известен локально и точно (VpnService/ConnectivityManager); приложение репортит сессии в бэкенд (`POST /api/vpn/session`).
- Подробно — [BACKEND.md](docs/BACKEND.md) и [NATIVE-APPS.md](docs/NATIVE-APPS.md).

## Стек (актуальные версии, август 2026)

| Слой | Технология | Версия |
|---|---|---|
| Рантайм | Node.js | **24 (Active LTS)** |
| PWA-фронт | React + Vite + vite-plugin-pwa | React **19.2**, Vite **8.2**, vite-plugin-pwa **1.3** |
| БД | PostgreSQL | **18** |
| ORM | Prisma | **7.8** |
| Очереди | BullMQ + Redis | BullMQ **6.0**, Redis **8.10** |
| Бот | Telegram Bot API | **10.2** |
| Логирование | pino | **10.3** |
| Трассировка/метрики | OpenTelemetry | GA (graduated) |
| Аналитика | PostHog (Cloud/self-host) | events, воронки, retention, SQL |
| Android-приложение | Kotlin + Jetpack Compose + openvpn (отдельный процесс) | minSdk 24, targetSdk 36 |
| Автоматизации | **n8n 2.29+ (внешний API)** | не в ядре, вызываем по требованию |

## Ключевой вывод

**iOS/macOS**
```
PWA (HTTPS) → .mobileconfig (com.apple.vpn.managed, VPNSubType) → системный профиль
  iOS:   shortcuts://run-shortcut → Set VPN (iOS 16.4+) → туннель
  macOS: меню-бар / OpenVPN Connect (экшена Set VPN на macOS нет)
```

**Android (нативное приложение)**
```
WebView кабинета → скачивание .ovpn → OpenVPN-модуль (VpnService + отдельный процесс openvpn)
  → локальный статус → репорт сессий в бэкенд
```

Реалистичный UX:
- **iOS**: установка — 3 действия 1 раз (Allow → Install → Shortcut), ежедневно — 1 тап.
- **macOS**: установка — как на iOS, включение — меню-бар / клиент, статус — по IP.
- **Android**: вход в приложение → подключение одной кнопкой, статус точный.

## Как открыть план

Прямая ссылка на README в GitHub: https://github.com/impossi8le/pwa-ios-vpn
