# Создание портативной версии Telegram Desktop для Windows

## Обзор

Telegram Desktop поддерживает портативный режим работы для Windows, который позволяет хранить все пользовательские данные в папке с приложением вместо стандартного расположения в AppData.

**Ключевой факт:** Портативная версия работает автоматически - приложение по умолчанию пытается сохранить данные рядом с исполняемым файлом. Никаких специальных папок создавать не нужно!

## Механизм работы портативного режима

### 1. Автоматическое определение рабочей директории

**Файл:** `Telegram/SourceFiles/logs.cpp:354-362`

В Windows (релизная сборка, не UWP) приложение использует следующую логику:

```cpp
if (cWorkingDir().isEmpty()) {
    cForceWorkingDir(cExeDir());  // Пытается использовать папку с .exe
    if (!LogsData->openMain()) {   // Если не может создать файл лога
        cForceWorkingDir(psAppDataPath());  // Переключается на %APPDATA%
    }
}
```

**Это означает:**
- По умолчанию приложение **пытается работать портативно** из папки с exe
- Если не может создать файлы (нет прав записи), переключается в `%APPDATA%\Telegram Desktop\`
- Никакой специальной папки `TelegramForcePortable` не требуется!

### 2. Папка TelegramForcePortable (устаревший механизм)

**Файл:** `Telegram/SourceFiles/core/launcher.cpp:231-284`

Функция `CheckPortableVersionFolder()` проверяет папку `TelegramForcePortable`, но это **устаревший механизм** для альфа/бета версий:

```cpp
const auto portable = cExeDir() + u"TelegramForcePortable"_q;
```

Этот код используется только для:
- Миграции старых альфа версий (`TelegramAlpha_data` → `TelegramForcePortable`)
- Миграции старых бета версий (`TelegramBeta_data` → `TelegramForcePortable`)
- Хранения приватного ключа альфа-версий

**Для обычных пользователей портативной версии эта папка не нужна!**

### 3. Структура портативной версии

Портативный ZIP-архив содержит только:

```
Telegram/
  ├── Telegram.exe
  └── modules/
      └── [архитектура]/
          └── d3d/
              └── d3dcompiler_47.dll
```

Где `[архитектура]` может быть:
- `x86` - для 32-битной версии
- `x64` - для 64-битной версии
- `arm64` - для ARM версии (без модуля d3dcompiler_47.dll)

**При первом запуске создается структура:**

```
Telegram/
  ├── Telegram.exe
  ├── modules/...
  └── tdata/              ← создается автоматически
      ├── settings0
      ├── key_data
      └── ...
```

**Файлы НЕ включаются в портативный архив:**
- `Updater.exe` - остается в папке deploy, но не архивируется
- `tdata/` - создается при первом запуске
- Файлы PDB (символы отладки)
- Установщик (.exe)

## Процесс сборки портативной версии

### 1. Сборка приложения

**Файл:** `Telegram/build/build.bat:187-194`

Сначала собирается релизная версия:

```batch
call configure.bat
cd "%SolutionPath%"
call cmake --build . --config Release --target Telegram
```

### 2. Подготовка файлов для портативной версии

**Файл:** `Telegram/build/build.bat:313-338`

После успешной сборки:

1. Создается структура папок для развертывания:
```batch
mkdir "%DeployPath%\%BinaryName%\modules\%Platform%\d3d"
```

2. Копируются необходимые файлы:
```batch
move "%ReleasePath%\%BinaryName%.exe" "%DeployPath%\%BinaryName%\"
xcopy "%ReleasePath%\modules\%Platform%\d3d\d3dcompiler_47.dll" "%DeployPath%\%BinaryName%\modules\%Platform%\d3d\"
move "%ReleasePath%\Updater.exe" "%DeployPath%\"
```

Обратите внимание: `Updater.exe` перемещается в `%DeployPath%`, но НЕ в подпапку `%BinaryName%\`!

3. Создается ZIP-архив портативной версии с помощью 7-Zip:
```batch
cd "%DeployPath%"
7z a -mx9 %PortableFile% %BinaryName%\
```

Команда `7z a -mx9 %PortableFile% %BinaryName%\` архивирует **только папку Telegram\**, не включая `Updater.exe`.

4. После создания архива структура очищается:
```batch
move "%DeployPath%\%BinaryName%\%BinaryName%.exe" "%DeployPath%\"
rmdir "%DeployPath%\%BinaryName%"
```

Это перемещает `Telegram.exe` обратно в корень deploy и удаляет пустую папку.

### 3. Имена файлов портативных версий

**Файл:** `Telegram/build/build.bat:110-124`

Имена файлов зависят от архитектуры:

- **32-бит:** `tportable.%AppVersionStrFull%.zip`
- **64-бит:** `tportable-x64.%AppVersionStrFull%.zip`
- **ARM64:** `tportable-arm64.%AppVersionStrFull%.zip`

Где `%AppVersionStrFull%` имеет формат:
- Стабильная версия: `X.X.X` (например, `4.14.2`)
- Бета версия: `X.X.X.beta` (например, `4.14.2.beta`)
- Альфа версия: `X.X.X_ALPHANUMBER` (например, `4.14.2_1001234`)

### 4. Подписывание файлов

**Файл:** `Telegram/build/build.bat:213-226`

Перед упаковкой выполняется цифровая подпись:

```batch
:sign1
call "%SignPath%" "%BinaryName%.exe"
if %errorlevel% neq 0 (
  timeout /t 3
  goto sign1
)

:sign2
call "%SignPath%" "Updater.exe"
```

## Процесс релиза портативных версий

**Файл:** `Telegram/build/release.py:202-236`

Скрипт релиза определяет следующие портативные версии для загрузки на GitHub:

```python
files.append({
  'local': 'tportable.' + version_full + '.zip',
  'remote': 'tportable.' + version_full + '.zip',
  'backup_folder': 'tsetup',
  'mime': 'application/zip',
  'label': 'Windows 32 bit: Portable',
})
files.append({
  'local': 'tportable-x64.' + version_full + '.zip',
  'remote': 'tportable-x64.' + version_full + '.zip',
  'backup_folder': 'tx64',
  'mime': 'application/zip',
  'label': 'Windows 64 bit: Portable',
})
files.append({
  'local': 'tportable-arm64.' + version_full + '.zip',
  'remote': 'tportable-arm64.' + version_full + '.zip',
  'backup_folder': 'tarm64',
  'mime': 'application/zip',
  'label': 'Windows on ARM: Portable',
})
```

## Использование портативной версии

### Для конечного пользователя

После распаковки архива получается структура:

```
Telegram/
  ├── Telegram.exe
  └── modules/...
```

### Автоматический портативный режим

**Просто запустите `Telegram.exe` - приложение автоматически работает в портативном режиме!**

При запуске происходит:

1. Приложение пытается создать папку `tdata` рядом с `Telegram.exe`
2. Если получается - работает портативно (данные рядом с exe)
3. Если нет прав записи - переключается в `%APPDATA%\Telegram Desktop\`

**Типичные сценарии:**

#### Сценарий 1: Портативный режим (обычный случай)
```
Telegram/
  ├── Telegram.exe
  ├── modules/...
  └── tdata/           ← создается автоматически при первом запуске
      ├── settings0
      ├── key_data
      └── ...
```

Данные хранятся рядом с exe. Можно перенести всю папку на USB или в другое место.

#### Сценарий 2: Режим %APPDATA% (если нет прав записи)
Если папка находится в `C:\Program Files` или другом защищенном месте:
- Данные сохраняются в `%APPDATA%\Telegram Desktop\tdata`
- Приложение работает как обычная установленная версия

### Принудительный портативный режим

Чтобы гарантировать портативный режим:
1. Распакуйте архив в папку с правами записи (не Program Files)
2. Запустите `Telegram.exe`
3. Папка `tdata` создастся автоматически

### Перенос данных

Чтобы перенести существующий профиль в портативную версию:

1. Закройте Telegram
2. Скопируйте папку `%APPDATA%\Telegram Desktop\tdata` в папку с `Telegram.exe`
3. Запустите портативную версию

## Различия между установщиком и портативной версией

### Установщик (setup.iss)

**Что устанавливается:**
- `Telegram.exe`
- `Updater.exe` ← включен в установщик
- `modules/[arch]/d3d/d3dcompiler_47.dll`

**Характеристики:**
- Устанавливается в `%USERAPPDATA%\Telegram Desktop`
- Создает ярлыки в меню Пуск и на рабочем столе
- Регистрируется в списке программ Windows
- Данные хранятся в `%USERAPPDATA%\Telegram Desktop\tdata`
- Поддержка автообновлений через `Updater.exe`

### Портативная версия (ZIP)

**Что в архиве:**
- `Telegram/Telegram.exe`
- `Telegram/modules/[arch]/d3d/d3dcompiler_47.dll`
- **НЕТ** `Updater.exe` ← важное отличие!

**Характеристики:**
- Не требует установки - просто распаковать
- Может запускаться с USB-накопителя
- Данные могут быть:
  - В `%APPDATA%\Telegram Desktop\tdata` (стандартный режим)
  - В `TelegramForcePortable\tdata` (если создана папка)
- Не регистрируется в системе
- **Автообновления не работают** (нет Updater.exe)

### Роль Packer.exe

**Файл:** `Telegram/SourceFiles/_other/packer.cpp`

`Packer.exe` НЕ создает портативную ZIP-версию. Он создает файл обновления:
- `tupdate%VERSION%` (32-бит)
- `tx64upd%VERSION%` (64-бит)
- `tarm64upd%VERSION%` (ARM64)

Эти файлы используются `Updater.exe` для автоматического обновления установленной версии.

## Особенности реализации

### Поддержка альфа/бета версий

**Файл:** `Telegram/SourceFiles/core/launcher.cpp:236-283`

Для альфа-версий используется специальный механизм с ключом:

```cpp
QFile key(portable + u"/tdata/alpha"_q);
if (cAlphaVersion()) {
    cForceWorkingDir(portable);
    QDir().mkpath(cWorkingDir() + u"tdata"_q);
    cSetAlphaPrivateKey(QByteArray(AlphaPrivateKey));
    // Запись приватного ключа
}
```

### Миграция старых папок

**Файл:** `Telegram/SourceFiles/core/launcher.cpp:201-229`

Функция `MoveLegacyAlphaFolder()` мигрирует старые папки:
- `TelegramAlpha_data` → `TelegramForcePortable`
- `TelegramBeta_data` → `TelegramForcePortable`

## Требования к окружению сборки

### Windows Build Environment

**Файл:** `Telegram/build/build.bat:34-66`

Сборка требует правильной среды Visual Studio 2022:

- **32-бит:** x86 Native Tools Command Prompt for VS 2022
- **64-бит:** x64 Native Tools Command Prompt for VS 2022
- **ARM64:** ARM64 Native Tools Command Prompt for VS 2022

### Необходимые инструменты

1. **7-Zip** - для создания ZIP-архива
2. **cmake** - для сборки проекта
3. **Visual Studio 2022** - компилятор
4. **Inno Setup** - для создания установщика (не используется для портативной версии)
5. **Инструмент подписи** - для цифровой подписи исполняемых файлов

## Пути к финальным файлам

**Файл:** `Telegram/build/build.bat:126-133`

После сборки портативные версии сохраняются в:

```
out/Release/deploy/%AppVersionStrMajor%/%AppVersionStrFull%/
```

И затем копируются в финальное расположение для резервного копирования:
- Для 64-бит: `%FinalReleasePath%/%AppVersionStrMajor%/%AppVersionStrFull%/tx64`
- Для ARM64: `%FinalReleasePath%/%AppVersionStrMajor%/%AppVersionStrFull%/tarm64`
- Для 32-бит: `%FinalReleasePath%/%AppVersionStrMajor%/%AppVersionStrFull%/tsetup`

## Заключение

Портативная версия Telegram Desktop для Windows работает следующим образом:

1. **Автоматический портативный режим**: Приложение по умолчанию пытается работать портативно, сохраняя данные в папке `tdata` рядом с exe-файлом

2. **Fallback в %APPDATA%**: Если не может создать файлы (нет прав записи), автоматически переключается в `%APPDATA%\Telegram Desktop\`

3. **Простота использования**: Пользователю нужно просто распаковать ZIP-архив и запустить - никаких дополнительных настроек

4. **Сборка**: Портативный ZIP создается командой `7z a -mx9` и содержит только папку `Telegram\` с exe и библиотеками

5. **TelegramForcePortable**: Это устаревший механизм для альфа/бета версий, обычным пользователям не нужен

Портативная версия полностью функциональна, но не включает `Updater.exe`, поэтому автоматические обновления не работают.
