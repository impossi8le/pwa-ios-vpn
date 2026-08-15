# ANDROID.md — PWA → OpenVPN на Android: почему это работает + план

> **⚠️ DEPRECATED (авг 2026)** — см. [PRODUCT.md](PRODUCT.md) и [NATIVE-APPS.md](NATIVE-APPS.md).
> Форма продукта изменилась: на Android VPN-управление уходит в **нативное приложение** (WebView кабинета + OpenVPN-модуль). PWA на Android больше не управляет VPN. Этот документ оставлен как **исследование** intent-механики (фактологическая база, из которой выросло решение о нативном приложении).

> **Цель**: спроектировать логику взаимодействия PWA ↔ Android для того же продукта, что и на iOS (ТГ-бот продаёт OpenVPN-конфигурации, пользователь ставит PWA, видит купленные конфиги, жмёт «Установить» и «Включить/Выключить»).
>
> **Фокус**: только Android, только OpenVPN. Документ — теоретическая база (без кода), по аналогии с `PLAN.md` / `ARCHITECTURE.md` / `RISKS.md` для iOS.

---

## 0. Сравнение с iOS — главный вывод

| | iOS | Android |
|---|---|---|
| Профиль | `.mobileconfig` → системный профиль iOS | `.ovpn` → внутри приложения OpenVPN |
| Импорт | 3 шага (Allow → Install → Shortcut), лимит 8 мин | **1 тап** по ссылке `intent:` → приложение открывает профиль |
| Управление туннелем | `shortcuts://run-shortcut` → Set VPN → iOS 16.4+ | **нет системного аналога**. Chrome не может дергать произвольные intents |
| Диалог согласия | нет | **каждый раз** системный `com.android.vpndialogs/.ConfirmDialog` (обходится только MDM) |
| Тихое переключение | нет (Safari спрашивает «Открыть в Команды?») | нет (из браузера вообще нельзя) |

**Ключевое ограничение Android**: Chrome по `intent://` запускает только activity с категорией `android.intent.category.BROWSABLE`. Системный экран VPN (`android.settings.VPN_SETTINGS`) этой категории **не имеет**, поэтому из PWA нельзя даже открыть настройки VPN напрямую. Встроенной системной URL-схемы для VPN (аналога `shortcuts://run-shortcut`) на Android **не существует**.

---

## 1. Доказательная база

### 1.1. Цепочка официальных механизмов

| Шаг | Механизм | Источник |
|-----|----------|----------|
| PWA отдаёт `.ovpn` | MIME `application/x-openvpn-profile` → OpenVPN for Android открывает файл | [ics-openvpn](https://github.com/schwabe/ics-openvpn), [FAQ](https://ics-openvpn.blinkt.de/FAQ.html) |
| Импорт по ссылке `intent:` | Chrome `intent:#Intent;...` + `S.browser_fallback_url` | [Chrome Intents](https://developer.chrome.com/docs/android/intents) |
| Профиль внутри клиента | `.ovpn` импортируется в OpenVPN Connect / ics-openvpn (внутри приложения, НЕ системный профиль) | [OpenVPN — Import a Profile](https://openvpn.net/connect-docs/import-profile.html) |
| Управление извне (Connect) | Intent `net.openvpn.openvpn.CONNECT/DISCONNECT` + `AUTOSTART_PROFILE_ID` | [OpenVPN — Tasker](https://openvpn.net/connect-docs/how-to-use-tasker.html) |
| Управление извне (ics-openvpn) | Intent `de.blinkt.openvpn.api.ConnectVPN/DisconnectVPN` + `profileName` | [ics-openvpn FAQ — Remote API](https://ics-openvpn.blinkt.de/FAQ.html) |
| Автоматизация из веба | MacroDroid Webhook: `http://macrodroid.com/webhook?id=...` → макрос шлёт intent | [MacroDroid](https://www.guytec.com) |
| «Подключён всегда» | Always-on VPN (Настройки → Network & Internet → VPN → шестерёнка) | [Android VPN docs](https://developer.android.com/develop/connectivity/vpn) |

### 1.2. Честные ограничения

| Ожидание | Реальность |
|----------|-----------|
| «Одна кнопка как на iOS» | ❌ Полного аналога `shortcuts://run-shortcut` нет. Chrome блокирует произвольные intents; каждый VPN-старт требует системный диалог согласия. |
| «Установил и забыл» — без шагов | ⚠️ Установка `.ovpn` — 1 тап (открыть в OpenVPN). Дальше профиль внутри приложения, не в системных настройках. |
| «Кнопка Включить работает из PWA» | ❌ Из браузера нельзя дёрнуть intent-управление клиентом. Только через автоматизатор (Tasker/MacroDroid) с ручной настройкой. |
| «Показать статус VPN» | ❌ PWA не видит состояние туннеля (нет API из браузера). |

**Вывод**: механика установки на Android — **проще, чем на iOS** (1 тап вместо 3 шагов, нет лимита 8 минут). Но **переключение** — сложнее: одна кнопка достижима только через Always-on (решает «включено всегда») или через автоматизатор.

---

## 2. Механика по шагам

### Шаг A. Установка профиля (один раз, 1 тап)

```
[PWA] «Установить» → <a href="intent:#Intent;action=android.intent.action.VIEW;
      type=application/x-openvpn-profile;S.browser_fallback_url=https://site/install-help;end">
      (user gesture — только по тапу)
  → Chrome скачивает .ovpn → открывает chooser (OpenVPN for Android / Connect)
  → Импорт профиля внутри приложения
  → Профиль создан (имя берётся из файла)
```

Условия:
- MIME `application/x-openvpn-profile` (иначе Chrome просто скачает файл в Downloads)
- `S.browser_fallback_url` — если клиент не установлен, ведём на Play Store / инструкцию
- Нет 8-минутного лимита и нет Stolen Device Protection (это iOS)

Важно про **официальный OpenVPN Connect**: MIME-открытие `.ovpn` у него есть, но **raw-ссылка на импорт не задокументирована**. В приложении есть вкладка «URL» — но URL вводится **вручную внутри приложения**, не по клику из браузера.

### Шаг B. Дальше — 3 ветки (выбор UX)

```
                        ┌─────────────────────────────────────────────────────┐
  Android? ── установка ──► «Как включать VPN?»                              │
                          │                                                  │
                          ├─ 1) Always-on VPN (по умолчанию, рекомендую)    │
                          │     Настройки → Network & Internet → VPN →       │
                          │     шестерёнка → Always-on + Block connections   │
                          │     → «включено всегда», кнопка не нужна          │
                          │                                                  │
                          ├─ 2) Quick Settings Tile клиента (1 тап)          │
                          │     Добавить плитку в шторку уведомлений          │
                          │     → тап по плитке включает/выключает            │
                          │                                                  │
                          └─ 3) MacroDroid webhook (для опытных)             │
                                PWA-кнопка → http://macrodroid.com/webhook?   │
                                id=...&action=connect → макрос шлёт intent    │
                                → туннель поднят (требует ручную настройку)   │
                                                                             │
                              (Вариант 4: Tasker intent — не из браузера,    │
                               нужен .shortcut/виджет — не для масс)          │
                              └──────────────────────────────────────────────┘
```

### Шаг C. Intent-управление туннелем (не из браузера)

Работает **только из приложения-автоматизатора** (Tasker / MacroDroid), не из PWA:

**Официальный OpenVPN Connect** (3.3.2+), использует **Profile ID** (генерируется при импорте, не меняется при переименовании):
```java
// Connect
Action: net.openvpn.openvpn.CONNECT
Extra:  net.openvpn.openvpn.AUTOSTART_PROFILE_ID:{profile_id}
Extra:  net.openvpn.openvpn.AUTOCONNECT:true
Package: net.openvpn.openvpn
Class:   net.openvpn.unified.MainActivity
Target:  Activity

// Disconnect
Action: net.openvpn.openvpn.DISCONNECT
Extra:  net.openvpn.openvpn.STOP:true
Package: net.openvpn.openvpn
Class:   net.openvpn.unified.MainActivity
Target:  Activity
```
Для старых версий (< 3.3.2) и OVPN-профилей: `AUTOSTART_PROFILE_NAME` с префиксом `PC ` + имя профиля.

**ics-openvpn (OpenVPN for Android)** — проще, по имени профиля:
```java
// Connect
Action: android.intent.action.MAIN
Class:  de.blinkt.openvpn/.api.ConnectVPN
Extra:  de.blinkt.openvpn.api.profileName:{имя профиля}

// Disconnect
Action: android.intent.action.MAIN
Class:  de.blinkt.openvpn/.api.DisconnectVPN
```

---

## 3. Рекомендация для MVP

### 3.0. Какой клиент использовать

**ics-openvpn (OpenVPN for Android, пакет `de.blinkt.openvpn`), а НЕ официальный OpenVPN Connect.**

Причины:
- Документированный **импорт из внешних приложений** (секция «Controlling from external apps» в README) — подходит для `intent:`-ссылки из PWA
- Управление **по имени профиля** (`profileName`), а не по Profile ID — предсказуемо для нового пользователя
- Есть официальная remote API (intents + AIDL)

Официальный Connect требует **Profile ID** (непредсказуем для нового пользователя) — сценарий «одна кнопка для произвольного клиента» ломается.

### 3.1. PWA-сайт (Android-ветка)

- [ ] Детекция платформы: `/Android/i` vs `/iPhone|iPad|iPod/i` (parsing `navigator.userAgent`)
- [ ] Кнопка «Установить» → `intent:#Intent;...type=application/x-openvpn-profile` + fallback
- [ ] Кнопка «Как включать?» → инструкция Always-on VPN (по умолчанию)
- [ ] (Опц.) Кнопка MacroDroid webhook для опытных
- [ ] Честный UX: PWA не показывает статус туннеля (не может)

### 3.2. Проверка end-to-end (на реальном устройстве)

- [ ] `intent://` с `type=application/x-openvpn-profile` реально открывает ics-openvpn из Chrome/WebAPK
- [ ] Профиль импортируется, туннель поднимается вручную
- [ ] Always-on VPN держит туннель после перезагрузки
- [ ] (Опц.) MacroDroid webhook из PWA переключает туннель

---

## 4. Риски и митигация

| # | Риск | Вероятность | Влияние | Митигация |
|---|------|-------------|---------|-----------|
| 1 | Chrome по `intent://` не открывает клиент (нет BROWSABLE / фильтры) | средняя | кнопка «Установить» молчит | `S.browser_fallback_url` на Play Store + инструкция; тест на устройстве |
| 2 | Официальный Connect требует Profile ID | высокая | «одна кнопка» ломается | Использовать ics-openvpn (управление по имени) |
| 3 | Каждый VPN-старт — системный диалог согласия | 100% | UX (лишний тап) | Always-on VPN (согласие один раз) |
| 4 | PWA не показывает статус туннеля | 100% | нельзя проверить «включён ли» | Инструкция «смотри иконку VPN в шторке» |
| 5 | MacroDroid webhook на мобильной сети / задержка | средняя | кнопка может не сработать | Позиционировать «для опытных»; fallback — ручное включение |
| 6 | Android 13+ ограничения фоновых запусков | средняя | автоматизация из фона режется | Запуск только по user gesture (foreground) |
| 7 | Пользователь не видит профиль в системных настройках VPN | средняя | путаница «где мой VPN» | Объяснить: профиль внутри приложения, не в Настройках |
| 8 | MIME `.ovpn` не привязан в Chrome → файл в Downloads | высокая | лишний шаг, отвал | Давать `intent:`-ссылку, а не голый `<a href=".ovpn">` |

---

## 5. Открытые вопросы (нужен натурный тест)

1. Работает ли `intent://` c `component=net.openvpn.openvpn/.unified.MainActivity` и custom action `net.openvpn.openvpn.CONNECT` в актуальных версиях Chrome/WebAPK — **нет публичных кейсов**.
2. Точное поведение custom action `net.openvpn.openvpn.CONNECT` при implicit intent (без BROWSABLE) вне Tasker.
3. Совместимость intent API официального Connect и ics-openvpn: **НЕ совместимы** (разные package/action/extra; у Connect нет AIDL).
4. Поведение на Android 11+ (package visibility): какие `<queries>` задекларировать, если решим нативный TWA.
5. iOS-флоу (`.mobileconfig` + shortcuts) остаётся без изменений — детекция платформы разводит ветки.

---

## Заключение

**Установка на Android проще, чем на iOS** (1 тап через `intent:` + MIME `application/x-openvpn-profile`, без лимита 8 минут и без Stolen Device Protection). **Переключение сложнее**: полного аналога `shortcuts://run-shortcut` нет — Chrome блокирует произвольные intents, каждый VPN-старт требует системный диалог согласия.

Реалистичный UX для Android:
- **Установка** — 1 тап (`intent:` → ics-openvpn).
- **«Включено всегда»** — Always-on VPN (настраивается 1 раз).
- **Кнопка** — только через автоматизатор (MacroDroid webhook для опытных) или Quick Settings Tile.

Рекомендация: для MVP использовать **ics-openvpn (OpenVPN for Android)**, а не официальный OpenVPN Connect — документированный intent-импорт и управление по имени профиля.
