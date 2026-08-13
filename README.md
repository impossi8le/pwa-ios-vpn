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
