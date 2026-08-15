# NATIVE-APPS.md — Нативные приложения (Android — MVP, Windows/Linux — позже)

> Платформы Android/Windows/Linux не могут управлять OpenVPN из PWA (браузер не даёт). Поэтому для них VPN-управление живёт в **нативном приложении**: внутри — кабинет (WebView → наш `/app`) + нативный OpenVPN-модуль. Документ — самодостаточная спецификация для LLM-реализации.

---

## 1. Почему нативное приложение

| Проблема PWA на Android | Решение нативным приложением |
|---|---|
| Chrome не может дёргать произвольные intents (только BROWSABLE) | Нативный код — полный доступ к intents / сервисам |
| Каждый VPN-старт — системный диалог согласия `com.android.vpndialogs` | Единственный диалог при первом подключении (обычное поведение VpnService) |
| PWA не видит статус туннеля | **VpnService** + `ConnectivityManager` дают точный статус локально |
| Always-on / MacroDroid — хрупкий UX | Управление из своего UI, опционально Always-on |
| [ANDROID.md](ANDROID.md) intent-механика | Больше не нужна (остаётся как исследование) |

---

## 2. MVP: Android (Kotlin)

### 2.1. Архитектура приложения

```
[Android app: package com.<brand>.vpn]
│
├── WebView-активность (кабинет)
│     · WebView → https://<site>/app?embed=app
│     · авторизация: app-токен (инжектится в заголовок / через JS bridge)
│     · JS-мост (bridge) → вызовы нативного модуля из JS (вкл/выкл, статус)
│
├── OpenVPN-модуль
│     · VpnService + OpenVPN (см. §2.2)
│     · конфиг: скачивается из кабинета (API), сохраняется в app-private storage
│     · управление: start/stop/status
│     · события: connected/disconnected/bytes → POST /api/vpn/session (репорт в бэкенд)
│
├── Авторизация
│     · экран входа в приложении (email+пароль) — если не авторизован
│     · app-токен в Android Keystore (encryptedSharedPreferences)
│     · отзыв токена из кабинета → приложение разлогинивается
│
└── Уведомления
      · foreground service (поддержка подключения в фоне)
      · push (FCM) для «подписка истекает», «VPN отключился» — опционально
```

### 2.2. OpenVPN-модуль: лицензионно-безопасный выбор

**Проблема лицензий:**
- **ics-openvpn** (OpenVPN for Android) — **GPLv2**: встраивание библиотекой в проприетарное приложение требует открыть исходники приложения под GPL.
- **OpenVPN3** (openvpn3-android / openvpn3-linux) — **AGPLv3**: встраивание библиотекой → сетевые пользователи вправе требовать исходники.

**Решение для MVP (лицензионно чистое):**
- **Запускать `openvpn` как отдельный процесс** (не линковать код): приложение управляет через CLI/management interface. Само приложение остаётся проприетарным.
  - Android: `openvpn` собранный как отдельный бинарь (из исходников OpenVPN 2.6, GPL) — запуск через `ProcessBuilder`/`exec` с привилегиями VpnService (`--dev tun` + `--ifconfig` на интерфейс от VpnService). Это стандартный подход [OpenVPN for Android](https://github.com/schwabe/ics-openvpn) (он сам так работает: свой процесс + VpnService).
- Либо: **обернуть ics-openvpn через его AIDL-API** (remote API) — приложение при этом тоже становится GPL-производным, НЕ рекомендуется.
- Либо: **принять открытость** (решение о лицензии приложения) — тогда можно линковать напрямую.

**Рекомендация:** «openvpn как отдельный процесс + VpnService» — проприетарное приложение, минимальные риски, стандартная практика.

### 2.3. Статус VPN в приложении

- Локальный статус: `VpnService` состояния + `ConnectivityManager.getNetworkCapabilities()` (`TRANSPORT_VPN`) — точный и мгновенный.
- Репорт в бэкенд: `POST /api/vpn/session` (`start/stop`, `bytes_in/out`, `config_id`) — событие `vpn_session_start/end` пишется **клиентом приложения**, а не парсингом status-файла (для Android-пользователей это точнее).
- WebView показывает статус через JS-мост: `getStatus()` → `{ connected: bool, config: cn }`.

### 2.4. Конфиг

- `GET /api/configs/:id/download` → `.ovpn` (JSON/мгновенно из API), сохраняется в app-private storage (не в Downloads).
- Автоподключение после установки — опционально.
- Срок действия проверяет и бэкенд (401/403 при отзыве) — приложение при получении ошибки разрывает туннель.

### 2.5. Стек Android (август 2026)

| Слой | Технология |
|---|---|
| Язык | Kotlin 2.x |
| UI | Jetpack Compose (или простой WebView-only) |
| WebView | Android System WebView |
| Хранение | EncryptedSharedPreferences / DataStore |
| OpenVPN | бинарь OpenVPN 2.6 (отдельный процесс) |
| Сеть | OkHttp / Retrofit |
| Пуш | FCM (опционально) |
| minSdk / targetSdk | minSdk 24, targetSdk актуальный (авг 2026 — 36) |

---

## 3. Windows и Linux — позже

- **Windows**: Electron + встроенный WebView кабинета + запуск `openvpn.exe` (openvpn2.6 installer). Управление и статус — через management interface (`--management 127.0.0.1:7505`), репорт в бэкенд.
- **Linux**: Electron + `openvpn` (system `openvpn` package) + management interface.
- Пока эти приложения не готовы: на Windows/Linux пользователь использует **кабинет в PWA** (подписка, оплата) и **скачивает `.ovpn`** для ручного подключения (Tunnelblick/OpenVPN GUI).

---

## 4. Проверка end-to-end (acceptance)

- [ ] Приложение: вход (email+пароль) → кабинет в WebView → скачивание конфига → подключение
- [ ] VpnService: туннель поднят, системный диалог согласия — 1 раз
- [ ] Статус в приложении точный (VpnService), WebView показывает «включён/выключен»
- [ ] `POST /api/vpn/session` отправляет start/stop + bytes; в PostHog идут `vpn_session_start/end`
- [ ] Отзыв подписки → серверный 401 → приложение разрывает туннель
- [ ] App-токен в Keystore, отзыв из кабинета разлогинивает приложение
- [ ] Фон: туннель держится (foreground service), Android 13+ — без резания фона
