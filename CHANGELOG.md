# Changelog

Изменения этого форка относительно [Slava-Shchipunov/awg-openwrt](https://github.com/Slava-Shchipunov/awg-openwrt).

Точка ответвления — коммит `9742aa5` (`docs: update README`).

---

## 2026-08-25 — AmneziaWG 3.1

Автор изменений: [@koshelevnv](https://github.com/koshelevnv) совместно с [Claude](https://claude.com/claude-code) (Anthropic).

### `dce9b6b` — обновление до 3.1

**`kmod-amneziawg/Makefile`**

```diff
-PKG_VERSION:=1.0.20260611
+PKG_VERSION:=3.1.20260812
```

**`amneziawg-tools/Makefile`**

```diff
-PKG_VERSION:=1.0.20260618
+PKG_VERSION:=3.1.20260812
-# TODO: при следующем обновлении убрать -2
-PKG_SOURCE_VERSION:=v$(PKG_VERSION)-2
+PKG_SOURCE_VERSION:=v$(PKG_VERSION)
```

Суффикс `-2` существовал только для тега `v1.0.20260618-2`; для 3.1 такого тега нет, и с ним сборка падает на этапе checkout.

**`amneziawg-tools/files/amneziawg.sh`** — proto-скрипт netifd. Апстрим Amnezia его не поставляет, он живёт здесь, поэтому одного поднятия версии пакетов недостаточно: без правки этого файла новые параметры невозможно задать через uci. Добавлены девять опций (объявление в `proto_amneziawg_init_config`, `config_get` и запись в конфиг для `awg setconf`):

| Опция uci | Ключ в конфиге AmneziaWG |
|---|---|
| `awg_header_protection_key` | `HeaderProtectionKey` |
| `awg_content_padding_addition` | `ContentPaddingAddition` |
| `awg_rekey_after_time` | `RekeyAfterTime` |
| `awg_rekey_timeout` | `RekeyTimeout` |
| `awg_reject_after_time` | `RejectAfterTime` |
| `awg_keepalive_timeout` | `KeepaliveTimeout` |
| `awg_max_handshake_attempts` | `MaxHandshakeAttempts` |
| `awg_random_trailers` | `RandomTrailers` |
| `awg_disable_cookies` | `DisableCookies` |

Список ключей сверен с парсером `src/config.c` апстрим-тега `v3.1.20260812`. Параметра `Itime` в 3.1 нет.

### `12fc77d` — права на создание релиза

**`.github/workflows/build-module.yml`**

```diff
 jobs:
   build:
+    permissions:
+      contents: write
```

У форков `default_workflow_permissions` = `read`, поэтому `softprops/action-gh-release` падал с `Resource not accessible by integration`. Сборка при этом проходила полностью, и пакеты терялись вместе с раннером — около 33 минут впустую.

### `da37af2` — поддержка 3.1 в веб-интерфейсе

**`luci-proto-amneziawg`**, версия `2.0.4` → `3.1.0`.

Симптом до правки: импорт конфига 3.1 через LuCI отвергался с ошибкой

```
Cannot parse configuration: PersistentKeepAlive setting is invalid
```

Причина — не в `awg`: он разбирает `PersistentKeepalive = 25-35` штатно, через `u16_range_from_string()`. Ошибку выдавал JS-валидатор формы, проверявший значение как номер порта. При этом даже исправленный keepalive не спасал: форма знала только параметры 2.0 и молча отбрасывала всё остальное, создавая секцию, с которой хендшейк не проходит.

Изменения в `htdocs/luci-static/resources/protocol/amneziawg.js`:

- девять новых полей на вкладке AmneziaWG — текстовые для ключа и диапазонов, выпадающие списки on/off для `RandomTrailers` и `DisableCookies`;
- хелперы `isAwgRange()` / `validateAwgRange()` / `isAwgBool()` — принимают как одиночное число, так и диапазон `lo-hi` с проверкой `lo <= hi <= 65535`;
- `parseConfig()` — разбор и валидация новых ключей, `PersistentKeepalive` переведён на `isAwgRange`;
- `handleApplyConfig()` — перенос новых значений в форму при импорте;
- `createPeerConfig()` — вывод новых параметров при экспорте конфига и генерации QR-кода;
- поле `persistent_keepalive` у пира: `datatype` с `range(0,65535)` заменён на строку с валидатором диапазона.

### `43ccefc` — install-скрипт указывает на этот форк

**`amneziawg-install.sh`**

```diff
-BASE_URL="https://github.com/Slava-Shchipunov/awg-openwrt/releases/download/"
+BASE_URL="https://github.com/koshelevnv/awg-openwrt/releases/download/"
```

Адрес был захардкожен, поэтому запуск скрипта из форка скачивал пакеты 2.0 из исходного репозитория — то есть откатывал установку 3.1.

---

## Обратная совместимость

Проверено по исходникам апстрим-тега `v3.1.20260812` и на живом роутере: конфиги 2.0 работают без изменений.

Все возможности 3.x закрыты условиями и включаются только явным заданием параметров:

- `awg_header_protection_init()` возвращает `false`, пока не задан `HeaderProtectionKey` (флаг `has_protection`), и заголовок уходит незашифрованным;
- `ContentPaddingAddition` и `RandomTrailers` при нулевых значениях уводят обработку в прежнюю ветку `else`;
- при приёме длина пакета сверяется строгим равенством, если `random_trailers` выключен;
- новые таймеры при отсутствии значения откатываются на константы WireGuard: `!u16_range_is_zero(...) ? u16_range_pick_one(...) : KEEPALIVE_TIMEOUT` (аналогично `REKEY_TIMEOUT`, `REJECT_AFTER_TIME`, `MAX_TIMER_HANDSHAKES`).

Параметры хранятся в `struct wg_device`, то есть привязаны к интерфейсу, а не к модулю: туннели 2.0 и 3.1 работают на одном роутере одновременно.

## Тестовый стенд

Cudy WR3000P v1 (`cudy,wr3000p-v1`), OpenWrt 25.12.5 `r33051-f5dae5ece4`, ядро 6.12.94, `aarch64_cortex-a53`, target `mediatek/filogic`, совместно с podkop 0.7.22 / sing-box.

Проверено: обновление поверх рабочей 2.0 без правки конфигов, одновременная работа трёх туннелей (2×2.0 + 1×3.1), импорт конфига 3.1 через LuCI со всеми параметрами, сквозная проверка внешнего IP.

Остальные платформы собраны, но на оборудовании не тестировались.

---

## Сборка под все платформы

25.08.2026 запущена полная сборка для OpenWrt 25.12.5: 93 пары target/subtarget, 94 задачи, все завершились успешно. Релиз [v25.12.5](https://github.com/koshelevnv/awg-openwrt/releases/tag/v25.12.5) содержит 372 пакета — по четыре на платформу.

Запуск: Actions → «Create Release on Tag» → Run workflow, версия `25.12.5`, поля target и subtarget оставлены пустыми — тогда `index.js` разбирает список платформ с `downloads.openwrt.org` и строит матрицу целиком.

Для форка обязателен `permissions: contents: write` в задаче `build` (коммит `12fc77d`): без него сборка проходит, но публикация релиза падает с `Resource not accessible by integration`, и пакеты теряются вместе с раннером.
