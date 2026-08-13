# PLAN.md — PWA → iOS VPN: почему это работает + план

> **Цель**: убедиться, что связка **PWA + OpenVPN Connect + Shortcuts** реально работает на iOS, и дать пошаговый план. Документ — теоретическая база (без кода).

**Сценарий**: ТГ-бот продаёт VPN-конфигурации. Пользователь заходит на сайт → ставит PWA → видит купленные конфиги → «Установить» → iOS ставит профиль → в PWA кнопка «Включить/Выключить» → Shortcuts всё делает сам.

**Фокус**: только iOS. Взаимодействие PWA ↔ iOS. Авторизация и бэкенд (n8n) — позже.

---

## Часть 1. Доказательная база

### 1.1. Цепочка официальных механизмов

| Шаг | Механизм | Источник |
|-----|----------|----------|
| PWA отдаёт `.mobileconfig` | MIME `application/x-apple-aspen-config` → Safari предлагает установку | [mobile-config-installer](https://github.com/rohit-chouhan/mobile-config-installer), [Apple support 102400](https://support.apple.com/en-us/102400) |
| iOS ставит профиль | `com.apple.vpn.managed`, `supervised: false`, `allowmanualinstall: true` | [apple/device-management](https://github.com/apple/device-management/blob/release/mdm/profiles/com.apple.vpn.managed.yaml) |
| Профиль → OpenVPN Connect | `VPNSubType = net.openvpn.connect.app`, параметры в `VendorConfig` | [OpenVPN mobileconfig docs](https://openvpn.net/connect-docs/mobileconfig-profile.html) |
| Система видит туннель | OpenVPN Connect через `NETunnelProviderManager` | [kean.blog](https://kean.blog/post/vpn-configuration-manager), [Apple NETunnelProviderManager](https://developer.apple.com/documentation/networkextension/netunnelprovidermanager) |
| Управление из Shortcuts | Действие **Set VPN** (iOS 16.4+), Connect/Disconnect/Toggle/On Demand | [Apple support 101583](https://support.apple.com/en-us/101583), [vninja.net](https://vninja.net/2023/07/17/ios-shortcut-toggle-vpn) |
| PWA запускает Shortcut | `shortcuts://run-shortcut?name=...` — официальная схема | [Apple Shortcuts URL scheme](https://support.apple.com/guide/shortcuts/run-a-shortcut-from-a-url-apd624386f42/ios) |

**6 звеньев, каждое — документированный API. Ни одного «костыля».**

### 1.2. Подтверждения

1. **`.mobileconfig` для OpenVPN Connect — официально** (не хак): OpenVPN документирует профиль `com.apple.vpn.managed` + `VPNSubType net.openvpn.connect.app` + директивы в `VendorConfig`, ручную установку без MDM. Инструмент `ovpnmcgen.rb` генерирует такие профили.

2. **iOS видит OpenVPN Connect как родное подключение**: реализация на Network Extension (`NETunnelProviderManager`) → профиль появляется в Настройки → VPN.

3. **Shortcuts переключает VPN**: действие Set VPN (iOS 16.4+) работает через NEVPNManager. Двойная страховка — OpenVPN Connect v3.3+ сам создаёт Siri-Shortcuts (Connect/Disconnect) для профиля.

4. **PWA инициирует всё**: HTTPS-сайт, ссылка на `.mobileconfig` (Safari сам предлагает), `window.location.href = "shortcuts://run-shortcut?name=..."` в обработчике клика.

### 1.3. Честные ограничения

| Ожидание | Реальность |
|----------|-----------|
| «Команда в фоне полсекунды» | ❌ Shortcuts **открывается на передний план**, виден баннер. Safari каждый раз спрашивает «Открыть в "Команды"?». Тихого фона нет. |
| «Установил и забыл» — без шагов | ❌ Установка `.mobileconfig` = Allow в Safari → Настройки → Install (в течение 8 минут). iOS 17.3+ требует отключить Stolen Device Protection. |
| «Одна кнопка навсегда» | ⚠️ После установки — включение = 1 тап + подтверждение Safari. Возврат в PWA через x-callback. |

**Вывод**: механика работает, но UX — «3 действия при установке, 1 тап + подтверждение при каждом включении». Это стандартный путь VPN-сервисов (Proton, Passepartout) и радикально лучше ручной инструкции.

---

## Часть 2. Механика по шагам

### Шаг A. `.mobileconfig` → iOS

```
[PWA] «Установить» → <a href="https://site/config.mobileconfig">
      Content-Type: application/x-apple-aspen-config, HTTPS
  → Safari: «This website is trying to download a configuration profile. Allow?»
  → Allow → iOS скачивает → «Профиль загружен» (Настройки)
  → Настройки → Профиль загружен → Install → Install
  → Профиль привязан к OpenVPN Connect
```

Условия:
- MIME `application/x-apple-aspen-config` (иначе Safari покажет XML)
- Установить в течение **8 минут** (иначе авто-удаление)
- iOS 17.3+: отключить **Stolen Device Protection** до установки

Структура профиля (key-value в `VendorConfig`, НЕ `.ovpn` текстом):

```xml
<dict>
  <key>PayloadType</key><string>com.apple.vpn.managed</string>
  <key>VPNType</key><string>VPN</string>
  <key>VPNSubType</key><string>net.openvpn.connect.app</string>
  <key>RemoteAddress</key><string>vpn.example.com</string>
  <key>AuthenticationMethod</key><string>Password</string>
  <key>VPNUserDefined</key>
  <dict>
    <key>VendorConfig</key>
    <dict>
      <key>remote</key><string>vpn.example.com 1194 udp</string>
      <key>client</key><string>NOARGS</string>
      <key>dev</key><string>tun</string>
      <key>auth-user-pass</key><string>NOARGS</string>
      <key>vpn-on-demand</key><string>0</string>
    </dict>
  </dict>
</dict>
```

Правила конвертации `.ovpn` → `VendorConfig`:
- без аргумента → `NOARGS`
- повторы `remote` → `remote.1`, `remote.2`, …
- многострочные блоки `<ca>/<cert>/<key>/<tls-auth>` → одна строка с `\n`
- обязательна `vpn-on-demand: 0`
- подписать профиль (идеально), неподписанный — с предупреждением

### Шаг B. Создание Shortcut (одноразово)

1. **Рекомендуемый — встроенные Siri-Shortcuts**: OpenVPN Connect v3.3+ → Edit profile → Create Shortcut (Connect) / Settings → Create Disconnect Shortcut.
2. **Действие Set VPN** (iOS 16.4+): «Команды» → «+» → Set VPN → Connect/Disconnect → выбрать профиль.
3. **Автоматический**: PWA отдаёт `.shortcut` файл через `shortcuts://import-shortcut?url=<.shortcut>&silent=true`.

### Шаг C. Ежедневное включение

```js
document.getElementById('vpn-toggle').onclick = () => {
  window.location.href = "shortcuts://run-shortcut?name=Включить_VPN";
};
```

Safari → «Открыть в "Команды"?» → «Команды» → Set VPN → туннель. Возврат: `shortcuts://x-callback-url/run-shortcut?name=...&x-success=<pwa-url>`.

---

## Часть 3. План реализации

### 3.0. Предпосылки
- [ ] OpenVPN-сервер → `.ovpn` (client, dev tun, proto udp, remote, auth-user-pass, inline `<ca>/<cert>/<key>`)
- [ ] iPhone iOS 16.4+, OpenVPN Connect из App Store
- [ ] HTTPS-хостинг (обязателен для PWA)
- [ ] Проверить доступность OpenVPN Connect в регионе (РФ — риски)

### 3.1. Генератор `.mobileconfig`
- [ ] Конвертер `.ovpn` → `VendorConfig` (NOARGS / remote.N / `\n` / vpn-on-demand)
- [ ] Сборка XML `com.apple.vpn.managed` + `VPNSubType net.openvpn.connect.app`
- [ ] (Опц.) Подпись профиля

### 3.2. PWA-сайт
- [ ] manifest.json + service worker → «Добавить на экран Домой»
- [ ] Личный кабинет: список купленных конфигов (позже — из n8n)
- [ ] Кнопка «Установить» → `.mobileconfig` (aspen-config)
- [ ] Кнопка «Включить/Выключить» → `shortcuts://run-shortcut`
- [ ] Хелперы: проверка OpenVPN Connect (`openvpn://`), подсказки про SDP и 8 минут

### 3.3. Shortcuts
- [ ] Способ: встроенные Siri-Shortcuts / Set VPN / авто-импорт `.shortcut`
- [ ] x-callback возврат в PWA

### 3.4. Проверка end-to-end
- [ ] Установка `.mobileconfig` (Allow → Настройки → Install)
- [ ] Профиль виден в Настройки → VPN
- [ ] Shortcut включает/выключает туннель
- [ ] Кнопка в PWA работает с экрана «Домой» (standalone)

---

## Часть 4. Риски и митигация

| Риск | Митигация |
|------|-----------|
| Stolen Device Protection (iOS 17.3+) | Инструкция «отключить до установки, вернуть после» |
| Профиль удаляется через 8 минут | PWA ведёт по шагам сразу |
| Shortcuts открывает «Команды» (не фон) | Честный UX + x-success возврат |
| App Store правило 5.4 «config profiles» | PWA — не приложение; туннель строит одобренный OpenVPN Connect |
| РФ: недоступность OpenVPN Connect | Проверить регион; запасной план — свой NEVPNManager-апп от организации |
| Safari блокирует схему без жеста | Переходы только по тапу |

---

## Заключение

**Да, работает.** Вся цепочка — официальные механизмы: `.mobileconfig` (aspen-config, com.apple.vpn.managed), Network Extension (`NETunnelProviderManager`), Shortcuts Set VPN (iOS 16.4+), URL-схема `shortcuts://run-shortcut`. Тот же путь используют Proton VPN, Passepartout.

UX: **установка** (1 раз) — 3 действия; **ежедневно** — 1 тап + подтверждение. Радикально лучше ручной инструкции из ТГ-бота.

**Пограничный случай для теста на реальном устройстве**: работает ли действие Set VPN именно с профилями OpenVPN Connect (механизм подтверждён, прямого теста на OpenVPN Connect не нашли — есть страховка встроенными Siri-Shortcuts приложения).
