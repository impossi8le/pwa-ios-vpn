# PWA → VPN (iOS + Android)

Теоретическое обоснование и план: **PWA** устанавливает VPN-конфигурацию на **iOS** через Apple-профиль (`.mobileconfig`) и управляет туннелем через **Shortcuts (Команды)** + приложение **OpenVPN Connect**; на **Android** — через `intent:`-ссылку на `.ovpn` (MIME `application/x-openvpn-profile`) + **OpenVPN for Android (ics-openvpn)**, включение — через Always-on VPN или автоматизатор (MacroDroid webhook).

> Продукт: ТГ-бот продаёт VPN-конфигурации. Пользователь заходит на сайт, устанавливает PWA, видит купленные конфиги, жмёт «Установить» — и затем включает/выключает VPN одной кнопкой. Без ручной возни с `.ovpn`-файлами.

## Содержимое

| Файл | О чём |
|------|-------|
| [PLAN.md](docs/PLAN.md) | iOS: полное техническое обоснование (почему это работает) + пошаговый план реализации |
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | iOS: схема взаимодействия компонентов и механика по шагам |
| [RISKS.md](docs/RISKS.md) | iOS: ограничения и риски (Stolen Device Protection, 8 минут, Shortcuts на передний план, App Store 5.4, РФ) |
| [ANDROID.md](docs/ANDROID.md) | Android: логика взаимодействия PWA ↔ OpenVPN (intent://, MIME, intent API, Always-on, 3 ветки UX, риски) |
| [BACKEND.md](docs/BACKEND.md) | Бэкенд: считывание состояния VPN (IP-сравнение + CN), стек (версии авг 2026), модульность, очереди/логирование, n8n как внешний API |

## Состояние VPN

Из чистого браузера определить статус VPN нельзя — единственный кросс-платформенный способ: **PWA спрашивает свой бэкенд, тот видит реальный IP клиента и сверяет с IP VPN-сервера**. «Чей конфиг активен» определяется **по Common Name** из status-файла OpenVPN. Подробнее — [BACKEND.md](docs/BACKEND.md).

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
| Автоматизации | **n8n 2.29+ (внешний API)** | не в ядре, вызываем по требованию |

## Ключевой вывод

Механика **рабочая** — все звенья цепочки документированы Apple и OpenVPN:

**iOS**
```
PWA (HTTPS)
  └─ .mobileconfig (application/x-apple-aspen-config) → Safari → Настройки iOS → Профиль
       (com.apple.vpn.managed, VPNSubType = net.openvpn.connect.app)
  └─ shortcuts://run-shortcut?name=...  →  «Команды» → действие Set VPN (iOS 16.4+)
       → NEVPNManager → OpenVPN Connect (NETunnelProviderManager) → туннель
```

**Android**
```
PWA (HTTPS)
  └─ intent:#Intent;type=application/x-openvpn-profile → Chrome → OpenVPN for Android (ics-openvpn)
       → импорт .ovpn профиля (1 тап, без лимита 8 минут)
  └─ включение: Always-on VPN (настраивается 1 раз) — «включено всегда»
       └─ или кнопка через автоматизатор (MacroDroid webhook) → intent net.openvpn.openvpn.CONNECT
```

Реалистичный UX:
- **iOS**: установка — 3 действия 1 раз (Allow → Install → добавить Shortcut), ежедневно — 1 тап + подтверждение Safari.
- **Android**: установка — 1 тап (`intent:` → ics-openvpn); включение — Always-on VPN (1 раз настроить) или кнопка через автоматизатор.

Сильно лучше ручной инструкции.

## Как открыть план

Прямая ссылка на README в GitHub: https://github.com/impossi8le/pwa-ios-vpn
