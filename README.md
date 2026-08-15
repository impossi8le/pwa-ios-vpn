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

Нативный клиент (Flutter, commuspaceVPN)
├── Android: ics-openvpn (подключение, локальный статус, репорт в бэкенд)
└── iOS: OpenVPNAdapter — распространяется через TestFlight (не App Store)
```

**Правило роутинга**: VPN-экран открывается первым, только если `(iOS | macOS) ∧ авторизован ∧ активная подписка`. Подробно — [PRODUCT.md](docs/PRODUCT.md).

## Содержимое

| Файл | О чём |
|------|-------|
| [PRODUCT.md](docs/PRODUCT.md) | **Форма продукта**: лендинг + кабинет + нативные приложения, правило роутинга, аккаунты, оплата |
| [PLAN.md](docs/PLAN.md) | iOS: полное техническое обоснование (почему это работает) + пошаговый план |
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | iOS: схема взаимодействия компонентов и механика по шагам |
| [RISKS.md](docs/RISKS.md) | iOS: ограничения и риски (Stolen Device Protection, 8 минут, Shortcuts, App Store 5.4, РФ) |
| [ANDROID.md](docs/ANDROID.md) | **DEPRECATED** — исследование intent-механики PWA↔OpenVPN (заменено Flutter-клиентом) |
| [NATIVE-APPS.md](docs/NATIVE-APPS.md) | **Нативные приложения**: реализованный Flutter-клиент (commuspaceVPN), лицензии GPL, iOS TestFlight (не App Store), переход auth с n8n |
| [BACKEND.md](docs/BACKEND.md) | Бэкенд: состояние VPN (IP-сравнение + CN), аккаунты/app-токены, стек, модульность, n8n как внешний API |
| [ANALYTICS.md](docs/ANALYTICS.md) | Полная аналитика: события, KPI, воронка лендинга, хранение (PostHog/ClickHouse+Metabase) |
| [IMPLEMENTATION.md](docs/IMPLEMENTATION.md) | **Самодостаточный план для LLM-агента**: Prisma, API, фазы, DoD, критерии готовности |

## Состояние VPN

- **iOS/macOS (PWA)** — из браузера статус не прочитать: PWA спрашивает бэкенд, тот видит реальный IP клиента и сверяет с IP VPN-сервера. «Чей конфиг активен» — по **Common Name** из status-файла OpenVPN.
- **Android (Flutter-клиент)** — статус известен локально и точно (ics-openvpn / VpnService); приложение репортит сессии в бэкенд (`POST /api/vpn/session`).
- **iOS (Flutter-клиент / TestFlight)** — локальный статус через OpenVPNAdapter; либо PWA-флоу (`.mobileconfig` + Shortcuts).
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
| Нативный клиент | **Flutter** + `openvpn_flutter` 1.3.4 | SDK ≥3.44, Dart ≥3.12; ics-openvpn (Android), OpenVPNAdapter (iOS) |
| iOS-деплой | **TestFlight** (не App Store) | добавляем тестировщиков, 90 дней на сборку |
| Автоматизации | **n8n 2.29+ (внешний API)** | не в ядре, вызываем по требованию |

## Ключевой вывод

**iOS/macOS**
```
PWA (HTTPS) → .mobileconfig (com.apple.vpn.managed, VPNSubType) → системный профиль
  iOS:   shortcuts://run-shortcut → Set VPN (iOS 16.4+) → туннель
  macOS: меню-бар / OpenVPN Connect (экшена Set VPN на macOS нет)
```

**Android (нативный клиент, Flutter)**
```
Flutter-клиент (commuspaceVPN) → openvpn_flutter → ics-openvpn
  → локальный статус → репорт сессий в бэкенд
```

**iOS (не App Store)**
```
Flutter-клиент → OpenVPNAdapter (NETunnelProvider) → сборка в TestFlight
  → добавление тестировщиков (Internal/External) → установка без джейлбрейка
```

Реалистичный UX:
- **iOS**: PWA `.mobileconfig` + Shortcuts ИЛИ Flutter-клиент из TestFlight.
- **macOS**: установка — как на iOS, включение — меню-бар / клиент, статус — по IP.
- **Android**: Flutter-клиент — вход → конфиг → подключение одной кнопкой, статус точный.

## Как открыть план

Прямая ссылка на README в GitHub: https://github.com/impossi8le/pwa-ios-vpn
