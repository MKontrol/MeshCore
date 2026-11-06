# SETUP.md — сборка и поддержка моего форка MeshCore

Персональная шпаргалка. Форк основан на теге `companion-v1.18.0`,
ветка правок — `mk-clock-sync`.

## 1. Одноразовая настройка окружения (новый компьютер)

Репозиторий хранит код, но НЕ среду сборки. После `git clone` сделать:

1. Установить PlatformIO Core (`pio`) — https://docs.platformio.org/en/latest/core/installation.html
   Первая сборка сама скачает платформу Espressif и библиотеки (10–20 мин).
2. Добавить в `~/.bashrc` (Git Bash):

   ```bash
   export PATH="/c/Users/$USER/.platformio/penv/Scripts:$PATH"
   cp -n /c/Users/$USER/.platformio/penv/Scripts/python.exe \
         /c/Users/$USER/.platformio/penv/Scripts/python3.exe
   hash -r
   ```

   Зачем: PlatformIO кладёт только `python.exe`, а `build.sh` вызывает
   `python3`. Без копии bash подхватывает заглушку WindowsApps
   (`/c/Users/.../WindowsApps/python3`), которая печатает «Python» и молча
   выходит — сборка обрывается на первом шаге. `hash -r` чистит кэш путей bash,
   иначе после копирования он ещё раз запомнит старый путь.

Проверка окружения:

```bash
which pio && pio --version          # penv/Scripts/pio, version 6.x
which python3 && python3 --version  # penv/Scripts/python3, Python 3.x
```

## 2. Сборка прошивки

Каждое окно bash (переменная живёт до закрытия окна):

```bash
cd ~/MeshCore
export FIRMWARE_VERSION="v1.18.0.MK.0"
sh build.sh build-firmware heltec_v4_repeater
```

Примечания:
- `build.sh` сам подставляет хэш коммита и дату сборки
  (`FIRMWARE_BUILD_DATE=$(date '+%d-%b-%Y)`) — версия и дата в прошивку
  приходят через `-DFIRMWARE_VERSION` / `-DFIRMWARE_BUILD_DATE`,
  дефолты в `MyMesh.h` (`#ifndef`) перекрываются автоматически.
- `pio run` вызывается дважды — это норма: первый раз компиляция,
  второй раз только цель `mergebin` (инкрементально, секунды).
- Опционально `export DISABLE_DEBUG=1` перед сборкой — отключает
  отладочные флаги; для heltec_v4_repeater не влияет (MESH_DEBUG закомментирован).

Результат в `out/`:
- `heltec_v4_repeater-v1.18.0.MK.0-<hash>.bin`        — прошивка (адрес 0x10000)
- `heltec_v4_repeater-v1.18.0.MK.0-<hash>-merged.bin` — полный образ (адрес 0x0)

## 3. Прошивка

Обновление живого узла (ключи и настройки НЕ стираются):

```bash
pio run -t upload -e heltec_v4_repeater
# или вручную:
esptool.py --chip esp32s3 --port COM5 --baud 921600 write_flash 0x10000 out/<файл>.bin
```

Чистая установка / восстановление «кирпича» — merged-файл с адреса 0x0:

```bash
esptool.py --chip esp32s3 --port COM5 --baud 921600 write_flash 0x0 out/<файл>-merged.bin
```

Если загрузка не стартует: зажать PRG/BOOT, коротко нажать RST, отпустить PRG.

Проверка без прошивки (строка версии лежит в бине открытым текстом):

```powershell
$bytes = [IO.File]::ReadAllBytes(".pio\build\heltec_v4_repeater\firmware.bin")
[regex]::Match([Text.Encoding]::ASCII.GetString($bytes),
  "v1\.18\.0\.MK\.0-[0-9a-f]{7}").Value
```

На устройстве: `ver` → `v1.18.0.MK.0-<hash> (Build: DD-Mon-YYYY)`.

## 4. Ключи и настройки узла

Ключи живут на узле (NVS/FS), а не в прошивке — перепрошивка их не стирает.
Потерять можно только `erase` (serial-only) или `esptool erase_flash`.

Бэкап (по USB-консоли):
- `get prv.key`  → сохранить hex (64 символа) в надёжном месте
- `get public.key` → сверка pub (вычисляется из prv, отдельно не задаётся)

Восстановление:
```
set prv.key <hex>
reboot
get public.key   # сверить с сохранённым
```
`get prv.key` работает только по serial (sender_timestamp == 0).

## 5. Git: работа с форком

Remotes:
- `origin` → github.com/meshcore-dev/MeshCore  (чтение, апстрим)
- `mine`   → github.com/<логин>/MeshCore       (мой форк, запись)

Подтянуть правки апстрима и переложить свои коммиты поверх:

```bash
git fetch origin
git checkout mk-clock-sync
git rebase origin/companion-v1.18.0   # база меняется с выходом новых версий
# при конфликте: правим файл, git add <файл>, git rebase --continue
```

После rebase — пересборка и прошивка заново (шаг 2–3).

Отправить своё:

```bash
git push mine mk-clock-sync   # дальше достаточно `git push mine`
```

Новый компьютер: `git clone https://github.com/<логин>/MeshCore.git`,
затем шаг 1 (окружение).

## 6. Мои правки (что держать в голове)

- `src/helpers/CommonCLI.cpp`: допуск синхронизации времени ±3 с,
  подробные ответы clock sync/time, маркеры [S] (serial) / [T] (BT).
- Веб-конфигуратор flashmesh.ru шлёт `clock sync` БЕЗ метки времени
  (sender_timestamp = 0) → ответ «Recv epoch: 0» с огромной дельтой —
  это не баг прошивки. Обход: в консоли браузера
  `Math.floor(Date.now()/1000)`, затем `time <epoch>` в терминале.
- Официальный MCO показывает полный ответ, MCOA при успехе показывает
  своё уведомление (только время) — прошивка при этом отвечает полной строкой.

## 7. Типовые грабли

| Симптом | Причина | Лечение |
|---|---|---|
| `build.sh` печатает «Python» и молча выходит | заглушка WindowsApps вместо python3 | шаг 1: копия python3.exe + hash -r |
| `pio: command not found` в bash | PATH не содержит penv/Scripts | export PATH из шага 1 |
| Повторный export PATH не помогает | кэш путей bash | `hash -r` |
| `FIRMWARE_VERSION must be set` | не задана переменная | export FIRMWARE_VERSION=... |
| `time` → Unknown command | команда без аргумента | `time <epoch>` |
