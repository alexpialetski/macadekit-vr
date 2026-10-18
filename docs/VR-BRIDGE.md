# PCVR на Mac: план VR-моста для MacadeKit

Рабочий документ ветки `vr-bridge` в форке [alexpialetski/macadekit-vr](https://github.com/alexpialetski/macadekit-vr).
Здесь всё, что решили и выяснили 18.10.2026, и пошаговый план для продолжения на другом Mac (в том числе для
новой сессии Claude Code, у которой нет контекста прошлой).

> Статус на 18.10.2026: фаза 0 почти закрыта. Движок 2.0.90 и компоненты BasaltVR собираются в GitHub Actions
> (раздел 8.3a). Ничего ещё не запускалось на Mac и в шлеме; фикс `pipe2` проверен только по исходникам, не
> сборкой с SDK 27. Всё, что помечено «не проверено», — выводы из чтения кода и документации.

---

## 1. Цель

Играть в Windows PCVR-игры (Steam/Epic/GOG) на Apple Silicon Mac в шлеме Meta Quest 3 — с трекингом головы
(6DoF), контроллерами, вибрацией и звуком. Через MaPC, без Windows-ПК и без виртуальных машин.

Минимальный результат («первая игра в шлеме»): одна D3D11 VR-игра (план — Beat Saber) запускается в
движке из этого форка и стримится на Quest 3.

Железо для тестов: MacBook M5 Pro, 48 ГБ, Quest 3.

## 2. Почему это вообще возможно

Проблема PCVR на Mac — не стриминг и не Quest. На macOS нет VR-рантайма: SteamVR для Mac убран (в 2020,
ветка `macos_default` удалена в декабре 2025), а Meta Quest Link есть только под Windows. Windows-игре под Wine
некому отдать кадры и не у кого спросить позу головы.

Решение — та же схема, что работает на Linux (Proton + WiVRn/Monado), но с macOS-частями:

```
Windows VR-игра (D3D11, OpenVR или OpenXR)
  └─ OpenComposite (только для OpenVR-игр: OpenVR → OpenXR), Windows-DLL внутри Wine
      └─ OpenXR-рантайм / мост внутри Wine
          └─ кадры: D3D11 → Metal-текстура (IOSurface)
              └─ нативный macOS-процесс: композиция + VideoToolbox H.264/HEVC
                  └─ Wi-Fi / USB
                      └─ клиент на Quest 3 (стандартный ALVR) → трекинг обратно
```

Встраивание в MaPC без исходников приложения: MaPC умеет брать движок из своего манифеста
(`MACADE_ENGINE_MANIFEST=file:///…/engine-manifest.json`, см. README репозитория). Значит, достаточно
собрать **свой движок** (Wine + графика + VR-компоненты) и указать на него. Настройки Wine (переопределения
DLL, ключи VR) можно класть в реестр бутылки — так уже делает MaPC (`Bottle.swift`, `WineRunner.swift:164`).

## 3. Что уже существует (разведка 18.10.2026)

| Проект | Лицензия | Что даёт | Зрелость |
|---|---|---|---|
| [BasaltVR](https://github.com/frenchbaguetteman/BasaltVR) | MIT | **Почти вся цепочка в открытом виде**: OpenXR-рантайм как Windows PE-DLL внутри Wine; форк DXMT v0.80 с экспортом D3D11-текстур в IOSurface (`runtime/dxmt-fork`, `runtime/dxmt-patches`, тег `v0.80-basalt.1`); нативный Swift-стример `basalt-streamer` (Metal-композиция, VideoToolbox H.264, транспорт ALVR); стандартный ALVR-клиент на Quest | Молодой (8 звёзд). Проверен только Beat Saber: M3 Max + Quest 3, 72 fps, трекинг, вибрация, звук (BlackHole). OpenXR-рантайм не conformant, `hello_xr` проходит. Пины: Wine 11.10 + DXMT v0.80. С D3DMetal (WhiskyWine 7.7) не работает |
| [MoltenVR](https://github.com/112cxyz/MoltenVR-Preview) | закрытый (Patreon) | Та же архитектура, что у BasaltVR: свой OpenXR/OpenVR-рантайм в бутылке, DXMT, ALVR-клиент v20.14.0, BlackHole. Заявлены Half-Life: Alyx, Beat Saber | Превью, только бинарники. Полезен как эталон «что возможно» |
| [OXRSys](https://github.com/demonixis/oxrsys) (бывш. OpenXR-OSX) | MPL-2.0 | Нативный OpenXR-рантайм для macOS: dylib в процессе приложения, Metal (`XR_KHR_metal_enable`) и Vulkan через MoltenVK (`XR_KHR_vulkan_enable/2`), VideoToolbox H.264/H.265, свой Quest/Pico-клиент. Есть x86_64-срез и arm64-хелпер кодирования для Rosetta | ~5 мес., 107 звёзд. Vulkan-путь и Rosetta-путь на реальном шлеме **не квалифицированы**. Требует `VK_EXT_metal_objects` для Vulkan |
| [monofunc/wineopenxr](https://github.com/monofunc/wineopenxr) | — (проверить) | `wineopenxr`, портированный на macOS: D3D11 OpenXR-игры → любой x86_64 рантайм с `XR_KHR_metal_enable` (т.е. OXRSys). Запускал Beat Saber и Unity XR в симуляторе OXRSys. См. [OXRSys issue #4](https://github.com/demonixis/oxrsys/issues/4) | Эксперимент |
| Proton `wineopenxr` + GE-Proton 10-33 wineopenvr | LGPL (Wine) | Мост OpenXR из Wine в нативный рантайм. D3D11 через интероп DXVK (`IDXGIVkInteropDevice`, игра рисует прямо в VkImage рантайма) | Работает на Linux. Подробности в разделе 6 |
| [DXMT](https://github.com/3Shain/dxmt) | ≤ v0.80: MIT; main: LGPL-2.1 | D3D11 → Metal напрямую (быстрее DXVK→MoltenVK). Своих средств интеропа с IOSurface нет — их добавляет форк BasaltVR | Активен. `winemetal.dll` и unix-часть связаны по версии |
| [OpenComposite](https://gitlab.com/znixian/OpenOVR) | GPLv3 | OpenVR → OpenXR, Windows-DLL (`vrclient_x64.dll`/`openvr_api.dll`). Нужен для OpenVR-игр (Alyx, большинство Steam VR) | Зрелый. xrizer не подходит — только нативный Linux |
| [ALVR](https://github.com/alvr-org/ALVR) | MIT | Клиент на Quest (v20.14.0 используют и MoltenVR, и BasaltVR) | Зрелый. Сервер под macOS официально не поддерживается |

Прочее:
- [ALVR issue #2047](https://github.com/alvr-org/ALVR/issues/2047): пользователь **reecelikesramen** (26.05.2026)
  сообщил о прототипе — пропатченные ALVR + CrossOver + DXMT, Half-Life: Alyx и The Lab на 90 fps,
  M4 Pro 24 ГБ + Quest 3S по Wi-Fi 6 ГГц. Код обещан, но на 18.10.2026 не опубликован.
- Негативный результат (почему Virtual Desktop/SteamVR через CrossOver не работают): заметки
  blog.nicholas.clooney.io (май 2026).

## 4. Принятые решения

1. **Форк**, а не PR сразу. Репозиторий — Apache 2.0. Название форка без «MaPC» (товарный знак Steale LLC).
   Upstream остаётся remote `origin`, форк — remote `fork`.
2. **Основной путь — Metal, по схеме BasaltVR** (путь B): DXMT v0.80 + IOSurface-патчи + OpenXR-рантайм в
   Wine + нативный стример + ALVR-клиент. Причина: единственный путь, который уже работает целиком.
3. **Запасной путь — Vulkan** (путь A): DXVK → `wineopenxr` (порт из GE-Proton) → OXRSys. Понадобится для
   Vulkan/DX12-игр или если путь B упрётся. Разведка показала, что он слабее: Vulkan-путь OXRSys не проверен,
   DXVK должен создавать устройство с расширениями рантайма.
4. **База движка — CrossOver-сборка (`engine/build-engine-d3dmetal.sh`, Wine 11.0 из CrossOver 26.3.0)**.
   Под неё уже есть `--with-vulkan --with-mingw` и шаблон добавления mingw-файлов (steamwebhelper-wrapper).
   Открытый вопрос: DXMT/BasaltVR проверены на Wine 11.10, совместимость с CrossOver 26.3 — **не проверена**.
   Если не заработает — собирать VR-движок на стоковом Wine 11.10, как у BasaltVR.
5. **Сначала один тест — Beat Saber**, потому что он уже работает у BasaltVR. Потом OpenVR-игры через
   OpenComposite, потом Alyx.
6. **Фикс `pipe2` отправить в upstream отдельным PR** — он нужен всем, независимо от VR.
7. **Сборку и запуск делать только на личном Mac.** Сборка Wine производит сотни Windows-бинарников
   (`wineconsole.exe`, `srvsvc.dll`…), и EDR-агенты помечают их как подозрительные. На машинах с
   корпоративной защитой не собирать.

### Жёсткие правила

- Никаких файлов D3DMetal / Apple Game Porting Toolkit в репозитории (запрет из README, лицензия Apple).
- Никакого обхода лицензии MaPC (Pro-функции) — ни в коде, ни в документации.
- Каждая загрузка в скриптах сборки — с пином SHA-256 (правило репозитория).
- GPL-компоненты (OpenComposite) — отдельными файлами, с текстом лицензии и ссылкой на исходники в
  `engine/licenses`. DXMT брать от тега v0.80 (MIT), не от main (LGPL).

## 5. Фаза 0 — фикс `pipe2` (сделано в коде, нужна проверка сборкой)

### Симптом
Движок 2.x (CrossOver) падает при любом запуске: игра «нажимается» и сразу гаснет. В
`~/Library/Logs/DiagnosticReports/wine-*.ips` — падение в `server_pipe` / `init_thread_pipe` в `ntdll.so`.

### Причина (проверено по исходникам и SDK)
- SDK macOS 27 объявляет `pipe2()` (и `dup3()`) с `__API_AVAILABLE(macos(27.0))`
  (`MacOSX.sdk/usr/include/sys/unistd.h:216-219`) и экспортирует символ в `libSystem.B.tbd`.
- `configure` Wine проверяет `pipe2` линковкой (`configure.ac`, список `AC_CHECK_FUNCS`) → `HAVE_PIPE2`.
- Движок собирается с `MACOSX_DEPLOYMENT_TARGET=14.0`, поэтому символ слабый. В macOS 26.x его нет
  (`nm -gU /usr/lib/system/libsystem_kernel.dylib | grep pipe2` пусто) → указатель нулевой.
- `dlls/ntdll/unix/server.c:1280` (`server_pipe`) вызывает `pipe2` без проверки на NULL → SIGSEGV.
  Запасной путь через `pipe()` + `FD_CLOEXEC` в коде есть.
- Из всех функций, которые проверяет `configure`, только `pipe2` — новая в macOS 27.

### Фикс (в этой ветке)
`engine/build-engine-d3dmetal.sh`:
- `ac_cv_func_pipe2=no` в окружении `configure` → Wine не использует `pipe2`, сразу `pipe()`.
- После сборки проверка: если `ntdll.so` импортирует `_pipe2` — сборка падает с ошибкой.

Исходники Wine не меняются, так что формулировка «built unmodified» в NOTICE остаётся верной.

## 6. Справка: устройство `wineopenxr` (для пути A)

Из разбора Proton master и коммита GE-Proton `a6a635e`:
- PE `wineopenxr.dll` + unix `wineopenxr.so`. Экспортирует `xrNegotiateLoaderRuntimeInterface` — Windows
  OpenXR-loader считает его рантаймом.
- ~34,7 тыс. строк, из них ~29 тыс. сгенерированы `make_openxr` из спецификации; руками ~3 тыс.
  (`openxr.c` 475, `openxr_loader.c` 2010).
- `XR_KHR_D3D11_enable` переписывается в `XR_KHR_vulkan_enable`; D3D11 — через `IDXGIVkInteropDevice`
  (DXVK): рантайм выделяет VkImage, мост оборачивает их `CreateTexture2DFromVkImage`. В DXVK из CrossOver
  26.3 нужный интерфейс есть (тот же GUID); побайтово vtable не сверялась.
- CrossOver 26.3 содержит `include/wine/vulkan_driver.h` (версия 47, как в Wine 11.0) — `openxr.c`
  компилируется почти без изменений.
- Что поменять для macOS: линковка с `libopenxr_loader.dylib`; `VK_KHR_xlib_surface` → `VK_EXT_metal_surface`;
  время `CLOCK_MONOTONIC` → `mach_absolute_time`/`CLOCK_UPTIME_RAW`; проброс `XR_*` env (включая
  `XR_RUNTIME_JSON`) в `ntdll/unix/env.c`; убрать ожидание реестра vrclient (как в GE).
- Межпроцессного шаринга не нужно: на macOS unix-часть Wine живёт в том же процессе, что и игра.

## 7. План работ

Оценка до первой игры в шлеме с переиспользованием BasaltVR: **~3–7M токенов** (было 7–20M до того, как
нашли BasaltVR). Узкое место — не токены, а итерации тестов в шлеме (их делает человек) и пересборки Wine
(20–60 минут каждая).

### Фаза 0. Основа ← мы здесь
- [x] Форк, ветка `vr-bridge`
- [x] Фикс `pipe2` в `build-engine-d3dmetal.sh`
- [x] Сборка движка и BasaltVR в GitHub Actions (раздел 8.3a)
- [ ] Сборка движка на личном Mac с SDK 27 — единственная настоящая проверка фикса `pipe2` (раздел 8.3)
- [ ] Проверить, что движок запускается в MaPC через свой манифест (обычная не-VR игра)
- [ ] PR фикса `pipe2` в upstream (отдельная ветка от `origin/main`, только этот коммит)

### Фаза 1. Повторить BasaltVR как есть (без MaPC)
Цель — убедиться, что цепочка работает на M5 Pro + Quest 3, прежде чем что-то встраивать.
- [ ] Прочитать `docs/ARCHITECTURE.md`, `docs/DXMT-INTEROP.md`, `runtime/README.md` в BasaltVR
- [ ] Собрать/поставить BasaltVR по его инструкции (его Wine 11.10 + DXMT v0.80-basalt.1)
- [ ] Поставить ALVR-клиент v20.14.0 на Quest 3
- [ ] Beat Saber в шлеме: зафиксировать fps, задержку, проблемы
- [ ] Если не работает — разобраться, прежде чем идти дальше

### Фаза 2. VR-движок для MaPC
- [ ] Решить базу: CrossOver 26.3 (наш) или Wine 11.10 (как BasaltVR). Начать с проверки, собирается ли
      DXMT v0.80-basalt.1 против нашего Wine и совпадают ли индексы unix-вызовов `winemetal`
- [ ] `engine/build-engine-vr.sh`: Wine + DXMT (форк BasaltVR, пин SHA-256) + OpenXR-рантайм BasaltVR (PE)
      + OpenComposite; `ENGINE.json` с бэкендом `dxmt` и признаком VR; лицензии в `engine/licenses`
- [ ] Настройки бутылки: переопределения DLL (`d3d11`, `dxgi` → DXMT; OpenXR-рантайм в
      `HKLM\Software\Khronos\OpenXR\1\ActiveRuntime`), env `DXMT_BASALT_IOSURFACE=1`
- [ ] Стример (`basalt-streamer`) — запускается отдельно, вне движка

### Фаза 3. Интеграция в MacadeKit
- [ ] Бэкенд `dxmt` в `GraphicsBackend` (`Sources/MacadeKit/Wine/GraphicsBackend.swift`) с тестами
- [ ] Флаг VR в `LaunchProfile` (`Sources/MacadeKit/Wine/LaunchProfile.swift`): переопределения + env в
      `WineRunner.environment()` (`Sources/MacadeKit/Wine/WineRunner.swift:49-85`), проверка, что движок
      содержит VR-компоненты (`ENGINE.json`)
- [ ] `mapc run --vr` (`Sources/MapcCommand/Options.swift`) для тестов без приложения
- [ ] `SteamVR` сейчас скрыт из библиотеки (`Sources/MacadeKit/Stores/Steam/SteamLibrary.swift:7`) — оставить так

### Фаза 4. Совместимость
- [ ] OpenVR-игры через OpenComposite (Beat Saber Steam-версия, The Lab)
- [ ] Half-Life: Alyx
- [ ] 90 fps / HEVC / USB-режим
- [ ] Бэкенд DXMT как отдельный PR в upstream (полезен и без VR)

### Запасной путь A (если B не взлетит)
- [ ] Порт `dlls/wineopenxr` из GE-Proton 10-33 в дерево CrossOver (раздел 6)
- [ ] OXRSys как рантайм; Windows-сборка `hello_xr` (Vulkan) → затем D3D11-игра → OpenComposite

## 8. Пошагово на личном Mac

### 8.1. Требования
- Apple Silicon, macOS 15+, Xcode 16+ (или Command Line Tools), Rosetta 2:
  `softwareupdate --install-rosetta --agree-to-license`
- Homebrew: `brew install bison pkgconf freetype gnutls sdl2 mingw-w64`
- MaPC установлен и открыт хотя бы раз
- ~10 ГБ свободного места (исходники 338 МБ, дерево сборки ~2+ ГБ, плюс движок)

### 8.2. Код
```bash
git clone https://github.com/alexpialetski/macadekit-vr.git
cd macadekit-vr
git switch vr-bridge
git remote add upstream https://github.com/abduzeedo/mapc-engine.git   # для PR и синхронизации
swift test                                                             # юнит-тесты, без Wine
```

### 8.3. Сборка движка
Версию бери заметно выше официальной, чтобы не пересечься с движками MaPC (MaPC выбирает новейший по
числовому сравнению версий):
```bash
engine/build-engine-d3dmetal.sh 2.0.90 2>&1 | tee /tmp/engine-build.log
```
- Идёт 20–60 минут. Лог configure: `engine/.cache/cx/build/configure.out`; лог make:
  `engine/.cache/cx/build/make.out`.
- Если упало после исправления — продолжить без пересборки с нуля: `RESUME=1 engine/build-engine-d3dmetal.sh 2.0.90`.
- Результат: `engine/dist/macade-engine-2.0.90.tar.xz` и `engine/dist/engine-manifest.json`.

Проверка фикса:
```bash
grep -n "pipe2" engine/.cache/cx/build/configure.out     # ожидается: checking for pipe2... (cached) no
grep -n "HAVE_PIPE2" engine/.cache/cx/build/include/config.h   # ожидается: /* #undef HAVE_PIPE2 */
nm -u engine/.cache/cx/stage/lib/wine/x86_64-unix/ntdll.so | grep pipe2   # ожидается: пусто
```

### 8.3a. Или взять готовое из GitHub Actions
Собирать локально не обязательно — в форке два workflow (вкладка Actions, запуск вручную или по push в
`vr-bridge`):

| Workflow | Что даёт | Время |
|---|---|---|
| `Build engine` (`.github/workflows/build-engine.yml`) | Движок 2.x + манифест + логи сборки | ~40 мин |
| `Build BasaltVR` (`.github/workflows/build-basalt.yml`) | Форк DXMT (IOSurface), `basalt_openxr.dll`, тестовые программы для Wine, `basalt-streamer`/`basaltctl`/`BasaltVR` | ~30 мин (LLVM потом из кеша) |

Первые успешные сборки (18.10.2026): движок 2.0.90 — run `39140324519`, sha256
`92b53dcfd5899321e57f070c28e1be3aaee3a06a5912152780edbf4b9fe6cadd`; BasaltVR `6116e59` — run `39140512430`.
Артефакты хранятся 90 дней.

Раннер — macOS 15.7, Xcode 16.4, SDK 15.5: в этом SDK `pipe2` ещё нет, так что CI-движок не затронут
проблемой из раздела 5 и проверка там проходит тривиально. Сам фикс проверяется только сборкой с SDK 27.

Скачать и переписать манифест на локальный путь (манифест из CI указывает внутрь раннера):
```bash
mkdir -p ~/macade-engines && cd ~/macade-engines
gh run download 39140324519 -R alexpialetski/macadekit-vr -n macade-engine-2.0.90
F=macade-engine-2.0.90.tar.xz
python3 ~/path/to/macadekit-vr/engine/manifest-add.py engine-manifest.json 2.0.90 \
  "file://$PWD/$F" "$(shasum -a 256 $F | cut -d' ' -f1)" "$(stat -f%z $F)" d3dMetal
# BasaltVR:
gh run download 39140512430 -R alexpialetski/macadekit-vr -D basalt
```

### 8.4. Запуск движка в MaPC
По README репозитория MaPC берёт движок из `MACADE_ENGINE_MANIFEST`. Как именно передать переменную
приложению — **не проверено**; варианты:
```bash
# вариант 1: запустить бинарник приложения напрямую с переменной
MACADE_ENGINE_MANIFEST="file://$PWD/engine/dist/engine-manifest.json" /Applications/MaPC.app/Contents/MacOS/MaPC
# вариант 2: переменная для всех GUI-приложений до перезагрузки
launchctl setenv MACADE_ENGINE_MANIFEST "file://$PWD/engine/dist/engine-manifest.json"
```
После этого в MaPC должен появиться/скачаться движок 2.0.90. Проверить: `mapc engines`.

Тест без приложения:
```bash
swift build --product mapc
.build/debug/mapc engines
.build/debug/mapc run --bottle epic --verbose 'C:\path\to\Game.exe'
```
Чтобы не трогать настоящие бутылки, указывай `MACADE_HOME` на отдельную папку.

### 8.5. Диагностика
| Что | Где |
|---|---|
| Лог запуска игры | `~/Library/Application Support/MaPC/Logs/<store>-<id>/last.log` |
| Логи интерфейса MaPC | `~/Library/Application Support/MaPC/Logs/mapc/` |
| Краши Wine | `ls -lt ~/Library/Logs/DiagnosticReports/wine-*.ips` → смотреть `Crashed Thread` |
| Бутылки | `~/Library/Application Support/MaPC/Bottles/<store>/` (реестр — `user.reg`) |
| Движки | `~/Library/Application Support/MaPC/Engines/<version>/` |
| Пропали `dxgi.dll`/`d3d11.dll` в бутылке | `WINEPREFIX=<bottle> <engine>/wine/bin/wine wineboot -u` |
| Зависшие процессы | `pgrep -il "wine\|mapc"`, `pkill -9 -f wine` |

### 8.6. PR фикса `pipe2` в upstream
```bash
git fetch upstream
git switch -c fix-pipe2 upstream/main
git cherry-pick <коммит с фиксом pipe2>
git push -u origin fix-pipe2        # origin здесь — твой форк
gh pr create --repo abduzeedo/mapc-engine --base main
```
В описании: симптом, причина (раздел 5), на каком Mac и macOS проверено, какой игрой.

## 9. Для новой сессии Claude Code

Если продолжаешь с Claude Code на другом Mac — дай ему прочитать этот файл. Договорённости:
- Рутинную работу (сборки, разбор логов, поиск по коду и в интернете, механические правки) отдавать
  сабагентам на Sonnet (совсем простое — Haiku). Архитектуру, интероп графики и неочевидные падения — в
  основной сессии на Opus. Большие логи в основной контекст не тащить.
- Тесты в шлеме делает человек: собирать изменения пачками на одну итерацию теста.
- Соблюдать «жёсткие правила» из раздела 4.
- Перед утверждениями про BasaltVR, OXRSys, monofunc/wineopenxr — перепроверять по их актуальному коду:
  данные выше получены 18.10.2026 разведкой и не все проверены вручную.
