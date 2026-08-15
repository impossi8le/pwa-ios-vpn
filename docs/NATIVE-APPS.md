# NATIVE-APPS.md — Нативные приложения (реализовано: Flutter-клиент)

> **Статус**: нативное приложение уже реализовано — репозиторий [impossi8le/commuspaceVPN](https://github.com/impossi8le/commuspaceVPN) (Flutter). Этот документ — спецификация под **реально существующий** код, а не гипотетический Android-модуль. Опираемся на факты из репозитория.

---

## 1. Что реализовано (факты из commuspaceVPN)

- **Flutter-приложение** на базе плагина [`openvpn_flutter`](https://github.com/nizwar/openvpn_flutter) **v1.3.4** (форк). Единый код под Android и iOS.
- **Движки OpenVPN**:
  - Android → **ics-openvpn** (GPLv2)
  - iOS → **OpenVPNAdapter** (NETunnelProvider / NetworkExtension)
  - На desktop (Windows/Linux/macOS) — **stub** (`vpn_stub.dart`): реального туннеля нет, симуляция для демо.
- **Auth** сейчас: **n8n webhooks** (`http://194.87.252.181:5678/webhook`), JWT HS256, срок 30 дней. Эндпоинты: `/auth/register`, `/auth/login`, `/auth/validate`, `/auth/logout`.
- **Хранилище**: SharedPreferences (профиль, JWT, конфиги, выбранный конфиг, состояние VPN).
- **Фичи**: добавление `.ovpn` из файла и по URL, смена конфигурации на лету, **Quick Settings Tile**, **Windows runner**, iOS-стиль UI (Cupertino).
- **Сборка**: `flutter build apk --debug` → `example/build/app/outputs/flutter-apk/app-debug.apk`.

### Структура приложения (example/lib)
```
config/    auth_config.dart   # baseUrl n8n (env AUTH_API_BASE), таймауты, mock-режим
models/    auth_session.dart  # JWT + AuthApiException
           user_profile.dart  # профиль (мерж null-полей с локальным)
           vpn_config.dart    # .ovpn конфиг (id, name, serverRegion, configContent/path)
screens/   home_screen.dart   # главный: VPN + список конфигов + смена на лету
           login_screen.dart  # вход/регистрация
           profile_screen.dart# профиль, авто-выход при невалидной сессии
services/  auth_service.dart  # register/login/validate/logout
           data_store.dart    # SharedPreferences
           vpn_service.dart   # OpenVPN-движок (initialize/connect/disconnect, таймауты)
           vpn_tile_handler.dart # Quick Settings Tile (polling 500 мс)
theme/     app_theme.dart     # iOS-стиль (AppColors, AppTheme)
utils/     jwt_helper.dart    # декодирование JWT
```

---

## 2. Лицензии (реальность)

| Компонент | Лицензия | Следствие |
|---|---|---|
| `openvpn_flutter` (форк) | MIT | ок |
| **ics-openvpn** (Android-движок) | **GPLv2** | Приложение, линкующее движок, — **GPL-производное** |
| **OpenVPNAdapter** (iOS-движок) | GPLv2 (форк ss-abramchuk) | То же для iOS-сборки |

**Вывод**: приложение фактически **открытое (GPL)**. Это не ошибка, а факт — репозиторий уже публичный, лицензия GPLv3 в `LICENSE`. Проектировать «проприетарный клиент» больше не нужно: весь клиент уже open-source по построению.

**Практика**: раз исходники публичны и лицензия GPL — можно линковать движки напрямую (как сейчас). Защита коммерческой логики — на стороне **бэкенда** (API, подписки), а не клиента.

---

## 3. iOS: режим тестирования (TestFlight) — потому что в App Store не публикуем

> Пользователь указал: **не сможем выложить на iOS/macOS App Store**. Значит iOS-сборка распространяется **вне сторов** — через TestFlight (или ad-hoc). Ниже — как это организовать.

### 3.1. Почему TestFlight

- Без App Store нельзя поставить приложение на iPhone: только **TestFlight** (Apple, до 10 000 тестировщиков / 90 дней на сборку) или **ad-hoc** (до 100 UDID).
- TestFlight — «бесплатный деплой»: добавляем тестировщиков по Apple ID / email, они получают TestFlight и ставят приложение без джейлбрейка.
- **Ограничение TestFlight**: сборка живёт **90 дней**, затем нужно залить новую. Приложение просит обновить — это норм для внутреннего тестирования.

### 3.2. Что нужно от Apple

1. **Apple Developer Program** ($99/год, личная или организация) — обязателен.
2. **App Store Connect** → приложение создаётся (без публикации в сторе).
3. **App ID** с capability **Network Extensions** (NETunnelProvider) + **App Groups**.
4. Сборка для **iOS** (`flutter build ios --no-codesign` → подпись в Xcode).

### 3.3. Добавление тестировщиков

**App Store Connect → Users and Access** (роли) и **TestFlight → Testers**:

```
App Store Connect
  ├─ My Apps → [App] → TestFlight
  │    ├─ Internal Testing (до 100 тестировщиков, роли в App Store Connect:
  │    │    Team Members с ролью — могут тестить сразу, без отзыва Apple)
  │    └─ External Testing (до 10 000; каждую сборку проверяет Apple,
  │         «Beta App Review» 1–2 дня)
  │
  ├─ Тester: + (добавить Apple ID / email) → отправка TestFlight invite
  │    └─ Тестировщик ставит приложение TestFlight → принимает инвайт → обновляет
  │
  └─ Для «добавлять людей в тестировщики»:
       · Internal: добавить в App Store Connect группу с ролью Developer/App Manager
       · External: добавить Apple ID в группу External Testers
```

**Как заливать сборки** (2 варианта):
1. **Xcode Organizer** — Archive → Distribute → TestFlight. Проще.
2. **CLI + fastlane** (рекомендуется для LLM-агента):
   ```bash
   # один раз: fastlane match для сертификатов/профилей
   fastlane add_plugin testflight
   fastlane run build_app
   fastlane run pilot upload
   # добавление тестировщика:
   fastlane pilot add_tester email:user@example.com app_identifier:com.commuspace.vpn
   ```
   `fastlane`-конфиг храним в репо (`fastlane/Fastfile`, `Appfile`).

### 3.4. macOS — та же логика

- Приложение собирается и для macOS (Flutter desktop), но в Mac App Store не публикуем.
- Распространение: **не-нотаризованный DMG/zip** (просто выкладываем файл) или **TestFlight для macOS** (также через App Store Connect, но для Mac).
- Пока нет macOS-деплоя — на macOS используется **PWA-кабинет** (см. [PRODUCT.md](PRODUCT.md) §3): `.mobileconfig` + меню-бар, статус по IP.

---

## 4. Текущий auth — переход с n8n на наш бэкенд

Сейчас приложение ходит в **n8n** (`/webhook/auth/...`). По [BACKEND.md](BACKEND.md) и [PRODUCT.md](PRODUCT.md) бэкенд — ядро, n8n — внешний. План перехода:

1. Поднять на нашем бэкенде те же 4 эндпоинта (`/api/auth/register|login|validate|logout`) — **совместимая сигнатура** (поля `token`, `expires_in`, `user`).
2. Сменить в приложении `AuthConfig.baseUrl` на наш API (`--dart-define=AUTH_API_BASE=https://api...`).
3. Дальше можно расширять: Telegram-привязка, подписки, app-токен для приложения.

Это даёт единую БД пользователей и плавный переход без переписывания приложения.

---

## 5. Что остаётся для интеграции с продуктом

- [ ] Смена `AuthConfig.baseUrl` на наш бэкенд (совместимый auth).
- [ ] Получение конфигов из кабинета: сейчас — файлом/URL руками; в продукте — список купленных конфигов из `GET /api/configs`.
- [ ] Репорт сессий: `POST /api/vpn/session` при `connected/disconnected` (событие `vpn_session_start/end`, source=`app`).
- [ ] Подписки/срок действия: при 401/403 — разрыв туннеля.
- [ ] iOS: fastlane + TestFlight-пайплайн, добавление тестировщиков.
- [ ] Windows: runner уже есть (stub VPN) — при необходимости подключить реальный `openvpn.exe` + management interface.

---

## 6. Стек (реализованный, август 2026)

| Слой | Технология | Версия |
|---|---|---|
| Фреймворк | Flutter (Dart) | SDK ≥**3.44**, Dart ≥**3.12** |
| Плагин VPN | `openvpn_flutter` (форк) | **1.3.4** |
| Движок Android | ics-openvpn | GPLv2 |
| Движок iOS | OpenVPNAdapter (ss-abramchuk) | GPLv2 |
| Auth (сейчас) | n8n webhooks + JWT HS256 | 30 дней |
| Хранение | SharedPreferences | — |
| Quick Settings Tile | плагин `openvpn_flutter` (vpn_tile_handler) | polling 500 мс |
| Desktop | stub (`vpn_stub.dart`), Windows runner | — |
| iOS-деплой | TestFlight (не App Store) | 90 дней на сборку |
| macOS-деплой | вне сторов (DMG) / PWA | — |

---

## 7. Acceptance (обновлённые проверки)

- [ ] Android APK собирается (`flutter build apk --debug`) и подключается к OpenVPN через ics-openvpn
- [ ] iOS-сборка подписывается, архивируется и заливается в TestFlight
- [ ] Тестировщик (Internal/External) получает инвайт, ставит приложение, подключается
- [ ] Auth переведён с n8n на наш бэкенд (совместимые эндпоинты)
- [ ] Конфиги приходят из кабинета (`GET /api/configs`), а не только файлом/URL
- [ ] Сессии репортятся в бэкенд (`POST /api/vpn/session`)
- [ ] При 401/403 (отзыв/истечение) приложение разрывает туннель
