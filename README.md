# PWA → iOS VPN

Теоретическое обоснование и план: **PWA на iOS** устанавливает VPN-конфигурацию через Apple-профиль (`.mobileconfig`) и управляет туннелем через **Shortcuts (Команды)** + приложение **OpenVPN Connect**.

> Продукт: ТГ-бот продаёт VPN-конфигурации. Пользователь заходит на сайт, устанавливает PWA, видит купленные конфиги, жмёт «Установить» — и затем включает/выключает VPN одной кнопкой. Без ручной возни с `.ovpn`-файлами.

## Содержимое

| Файл | О чём |
|------|-------|
| [PLAN.md](docs/PLAN.md) | Полное техническое обоснование (почему это работает) + пошаговый план реализации |
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | Схема взаимодействия компонентов и механика по шагам |
| [RISKS.md](docs/RISKS.md) | Ограничения и риски (Stolen Device Protection, 8 минут, Shortcuts на передний план, App Store 5.4, РФ) |

## Ключевой вывод

Механика **рабочая** — все звенья цепочки документированы Apple и OpenVPN:

```
PWA (HTTPS)
  └─ .mobileconfig (application/x-apple-aspen-config) → Safari → Настройки iOS → Профиль
       (com.apple.vpn.managed, VPNSubType = net.openvpn.connect.app)
  └─ shortcuts://run-shortcut?name=...  →  «Команды» → действие Set VPN (iOS 16.4+)
       → NEVPNManager → OpenVPN Connect (NETunnelProviderManager) → туннель
```

Реалистичный UX: **установка** — 3 действия 1 раз (Allow → Install → добавить Shortcut), **ежедневно** — 1 тап + подтверждение Safari. Сильно лучше ручной инструкции.

## Как открыть план

Прямая ссылка на README в GitHub: https://github.com/impossi8le/pwa-ios-vpn
