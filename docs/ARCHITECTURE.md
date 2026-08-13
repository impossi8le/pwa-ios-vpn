# ARCHITECTURE.md — Схема взаимодействия компонентов

## Общая схема

```
[ НАЧАЛО: пользователь открывает PWA с экрана «Домой» ]
       │
       ▼
 [ 1. Установка профиля (один раз) ]
       │  PWA отдаёт .mobileconfig (application/x-apple-aspen-config)
       ▼
 [ Safari: «This website is trying to download a configuration profile. Allow?» ]
       │  Allow
       ▼
 [ Настройки iOS → Профиль загружен → Install ]
       │  Профиль привязан к OpenVPN Connect (VPNSubType net.openvpn.connect.app)
       ▼
 [ 2. Добавление Shortcut (один раз) ]
       │  OpenVPN Connect → Edit profile → Create Shortcut
       │  ИЛИ «Команды» → Set VPN → Connect/Disconnect → выбрать профиль
       ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. ЕЖЕДНЕВНОЕ ИСПОЛЬЗОВАНИЕ                                 │
│                                                             │
│  [Кнопка в PWA: «Включить»]                                 │
│         │ window.location.href = "shortcuts://run-shortcut"  │
│         ▼                                                    │
│  [Safari: «Открыть в "Команды"?»] ← каждое нажатие           │
│         │  Открыть                                            │
│         ▼                                                    │
│  [«Команды» (Shortcuts) — открываются на передний план]      │
│         │  действие Set VPN → Connect                        │
│         ▼                                                    │
│  [NEVPNManager / NETunnelProviderManager]                    │
│         │                                                   │
│         ▼                                                   │
│  [OpenVPN Connect — туннель поднят]                          │
│         │                                                   │
│         ▼                                                   │
│  [ VPN УСПЕШНО ВКЛЮЧЕН ]                                     │
└─────────────────────────────────────────────────────────────┘
```

## Два уровня интеграции

### Уровень 1 — «Системный профиль» (рекомендуемый)

`com.apple.vpn.managed` + `VPNSubType = net.openvpn.connect.app`:
- Профиль виден в Настройках iOS как **системный VPN**
- OpenVPN Connect читает директивы из `VendorConfig`
- Управление: действие Set VPN (iOS 16.4+) **и** встроенные Siri-Shortcuts

### Уровень 2 — «Просто импорт» (упрощённый, без .mobileconfig)

Отдать `.ovpn` по URL (MIME `application/x-openvpn-profile`):
- Пользователь открывает в OpenVPN Connect → профиль внутри приложения
- **Без** системного профиля: действие Set VPN может не увидеть его
- Управление только встроенными Siri-Shortcuts приложения

> Для сценария «две кнопки и забыл» — уровень 1.

## Ключевые точки

| Точка | Технология |
|-------|-----------|
| Скачивание профиля | `<a href=".mobileconfig">`, MIME `application/x-apple-aspen-config`, HTTPS |
| Структура профиля | plist XML, `VPNType=VPN`, `VPNSubType=net.openvpn.connect.app`, `VendorConfig` (key-value) |
| Системная регистрация | Network Extension: `NETunnelProviderManager` |
| Управление | Shortcuts Set VPN (iOS 16.4+) / Siri-Shortcuts OpenVPN Connect |
| Запуск из PWA | `shortcuts://run-shortcut?name=...` (по жесту) |
| Возврат | `shortcuts://x-callback-url/run-shortcut?name=...&x-success=<url>` |

## Поток данных конфигурации

```
БД ТГ-бота (mail_accounts / конфиги)
   │  (позже: n8n-вебхуки)
   ▼
[ PWA: личный кабинет — список купленных конфигов ]
   │
   ├─ «Установить» → GET /configs/<id>.mobileconfig
   │      → конвертация .ovpn → VendorConfig → plist XML
   │      → ответ с Content-Type: application/x-apple-aspen-config
   │
   └─ «Включить/Выключить» → shortcuts://run-shortcut
```

## Замечание про PWA

PWA **не может** сам поднять VPN-туннель (песочница браузера). Туннель всегда строит OpenVPN Connect. PWA лишь:
1. Раздаёт конфигурацию (`.mobileconfig` / `.ovpn`)
2. Проверяет установлен ли OpenVPN Connect (`openvpn://`)
3. Запускает Shortcut через URL-схему
4. Ведёт пользователя по шагам установки
