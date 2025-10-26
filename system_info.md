# Сбор и отправка системной информации Telegram Desktop


## 📋 Оглавление
1. [Краткий обзор](#краткий-обзор)
2. [Что собирается](#что-собирается)
3. [Откуда берется информация](#откуда-берется-информация)
4. [Куда и как отправляется](#куда-и-как-отправляется)
5. [Структура данных](#структура-данных)
6. [Код и файлы](#код-и-файлы)
7. [Можно ли изменить](#можно-ли-изменить)

---

## Краткий обзор

Telegram Desktop при каждом подключении к серверу отправляет метод MTProto `initConnection`, который содержит детальную информацию о вашей системе и устройстве.

**Когда отправляется**: При каждой инициализации сессии (подключении к серверу)
**Куда отправляется**: На серверы Telegram через MTProto протокол
**Можно ли отключить**: Нет, это обязательная часть протокола

---

## Что собирается

### 1. API ID (Идентификатор приложения)
**Значение**: `2040` (стандартная desktop версия)

**Варианты**:
- `2040` - Telegram Desktop (стандартный)
- `17349` - Тестовый API ID
- `611335` - Snap версия

**Файл**: `Telegram/SourceFiles/config.h:90`

---

### 2. Device Model (Модель устройства)

#### Windows
**Источник**: Реестр Windows
- **Путь**: `HKEY_LOCAL_MACHINE\HARDWARE\DESCRIPTION\System\BIOS`
- **Ключи** (в порядке приоритета):
  1. `SystemProductName` - например "HP EliteBook 840 G7"
  2. `SystemFamily` + `BaseBoardProduct` - комбинация
  3. `BaseBoardProduct` - только модель материнской платы
  4. `SystemFamily` - только семейство системы

**Примеры**:
- `"HP EliteBook 840"`
- `"ASUS ROG Strix"`
- `"Desktop"` (если ничего не найдено)

**Код**: `Telegram/lib_base/base/platform/win/base_info_win.cpp:154-186`

```cpp
QString DeviceModelPretty() {
    const auto bios = QSettings(
        "HKEY_LOCAL_MACHINE\\HARDWARE\\DESCRIPTION\\System\\BIOS",
        QSettings::NativeFormat);

    const auto systemProductName = bios.value("SystemProductName").toString();
    // ... обработка и возврат значения
}
```

#### Linux
**Источник**: DMI/SMBIOS данные в sysfs
- **Путь**: `/sys/class/dmi/id/`
- **Файлы** (в порядке приоритета):
  1. `product_name` - название продукта
  2. `product_family` + `board_name` - комбинация
  3. `board_name` - только материнская плата
  4. `product_family` - только семейство

**Дополнительно**:
- Если DMI недоступен, использует `systemd-detect-virt` (для виртуальных машин)
- Определяет тип chassis: Desktop, Laptop, Tablet, Server и т.д.

**Примеры**:
- `"ThinkPad X1 Carbon Gen 9"`
- `"Dell XPS 13"`
- `"Laptop"` (по chassis type)
- `"Desktop"` (fallback)

**Код**: `Telegram/lib_base/base/platform/linux/base_info_linux.cpp:87-136`

#### macOS
**Источник**: system_profiler и sysctl
- **Команда**: `system_profiler -json SPHardwareDataType`
- **Поля JSON**: `machine_name`, `chip_type`
- **Fallback**: sysctl `hw.model` (например "MacBookAir10,1")

**Примеры**:
- `"MacBook Air M1"`
- `"MacBook Pro M2 Max"`
- `"Mac Book Air"` (парсинг из hw.model)
- `"Mac"` (fallback)

**Код**: `Telegram/lib_base/base/platform/mac/base_info_mac.mm:121-166`

**Ограничения**:
- Максимальная длина: 15 символов (обычные модели)
- Максимальная длина: 32 символа (для хороших моделей типа HP)
- Удаляются подчеркивания, лишние пробелы

---

### 3. System Version (Версия системы)

#### Windows
**Определение версии**:
- Использует Windows API: `IsWindows11OrGreater()`, `IsWindows10OrGreater()` и т.д.
- Определяет архитектуру: `IsWow64Process2()` для x64/arm64

**Примеры**:
- `"Windows 11 x64"`
- `"Windows 10 arm64"`
- `"Windows 8.1"`
- `"Windows 7"`

**Код**: `Telegram/lib_base/base/platform/win/base_info_win.cpp:188-235`

```cpp
QString SystemVersionPretty() {
    QStringList resultList;

    if (IsWindows11OrGreater()) {
        resultList << "Windows 11";
    } else if (IsWindows10OrGreater()) {
        resultList << "Windows 10";
    }
    // ... определение архитектуры

    switch (nativeMachine) {
    case IMAGE_FILE_MACHINE_AMD64:
        resultList << "x64";
        break;
    case IMAGE_FILE_MACHINE_ARM64:
        resultList << "arm64";
        break;
    }

    return resultList.join(' ');
}
```

#### Linux
**Собирается**:
- Имя ядра через `uname()` (например "Linux")
- Версия ядра (например "6.5.0")
- Desktop Environment из `XDG_CURRENT_DESKTOP` (KDE, GNOME, XFCE и т.д.)
- Window manager: X11, Wayland, Xwayland
- Версия glibc

**Примеры**:
- `"Linux 6.5.0 KDE Wayland glibc 2.38"`
- `"Linux 5.15.0 GNOME X11 glibc 2.35"`

**Код**: `Telegram/lib_base/base/platform/linux/base_info_linux.cpp:138-183`

#### macOS
**Определение**:
- Версия macOS через системные API
- Формат зависит от версии (OS X vs macOS)

**Примеры**:
- `"macOS 14.0"` (Sonoma)
- `"macOS 13.0"` (Ventura)
- `"OS X 10.15"` (Catalina)

**Код**: `Telegram/lib_base/base/platform/mac/base_info_mac.mm:168-179`

---

### 4. App Version (Версия приложения)

**Формат**: `<версия> [архитектура] [источник]`

**Базовая версия**: Из `Telegram/SourceFiles/core/version.h`
```cpp
constexpr auto AppVersionStr = "6.2.4";
```

**Архитектура**:
- Windows x64: добавляется `" x64"`
- Windows x86: не добавляется
- Другие платформы: из `QSysInfo::buildCpuArchitecture()`

**Источник установки**:
- Mac App Store: `" Mac App Store"`
- Microsoft Store: `" Microsoft Store"`
- Flatpak: `" Flatpak"`
- Snap: `" Snap"`
- Обычная установка: ничего не добавляется

**Примеры**:
- `"6.2.4 x64"` - Windows 64-bit стандарт
- `"6.2.4"` - Windows 32-bit стандарт
- `"6.2.4 aarch64 Flatpak"` - Linux ARM в Flatpak
- `"6.2.4 Mac App Store"` - macOS из App Store
- `"6.2.4 x86_64"` - Linux x64 стандарт

**Код**: `Telegram/SourceFiles/mtproto/session_private.cpp:79-99`

```cpp
QString ComputeAppVersion() {
    #if defined Q_OS_WIN && defined Q_PROCESSOR_X86_64
        const auto arch = u" x64"_q;
    #elif (defined Q_OS_WIN && defined Q_PROCESSOR_X86_32) || defined Q_PROCESSOR_X86_64
        const auto arch = QString();
    #else
        const auto arch = ' ' + QSysInfo::buildCpuArchitecture();
    #endif

    return QString::fromLatin1(AppVersionStr) + arch + ([] {
        #if defined OS_MAC_STORE
            return u" Mac App Store"_q;
        #elif defined OS_WIN_STORE
            return u" Microsoft Store"_q;
        #else
            return KSandbox::isFlatpak()
                ? u" Flatpak"_q
                : KSandbox::isSnap()
                ? u" Snap"_q
                : QString();
        #endif
    })();
}
```

---

### 5. System Language Code (Язык системы)

**Источник**: Системные настройки ОС

**Формат**: ISO 639-1 + ISO 3166-1 (язык-регион)

**Примеры**:
- `"en-US"` - Английский (США)
- `"ru-RU"` - Русский (Россия)
- `"de-DE"` - Немецкий (Германия)
- `"zh-CN"` - Китайский (КНР)
- `"pt-BR"` - Португальский (Бразилия)

---

### 6. Language Pack Name (Название языкового пакета)

**Значение**: `"tdesktop"` (для Telegram Desktop)

**Описание**: Идентификатор языкового пакета, используемого приложением

**Другие варианты**:
- `"android"` - для Android версии
- `"ios"` - для iOS версии
- `"macos"` - для macOS версии

---

### 7. Language Code (Код языка в приложении)

**Источник**: Настройки языка в самом Telegram

**Формат**: ISO 639-1 (двухбуквенный код)

**Примеры**:
- `"en"` - English
- `"ru"` - Русский
- `"uk"` - Українська
- `"de"` - Deutsch
- `"es"` - Español

**Отличие от System Language Code**: Это язык, выбранный пользователем в настройках Telegram, а не язык системы.

---

### 8. Timezone Offset (Смещение часового пояса)

**Формат**: JSON параметр в секундах

**Вычисление**:
1. Берется текущее время в локальной и UTC зонах
2. Вычисляется разница в секундах
3. Учитывается unixtime shift
4. Округляется до ближайших 15 минут (900 секунд)
5. Ограничивается диапазоном от -12 до +14 часов

**Примеры**:
- `{"tz_offset": 0}` - UTC (Лондон зимой)
- `{"tz_offset": 10800}` - UTC+3 (Москва)
- `{"tz_offset": -18000}` - UTC-5 (Нью-Йорк)
- `{"tz_offset": 43200}` - UTC+12 (Новая Зеландия)

**Код**: `Telegram/SourceFiles/mtproto/session_private.cpp:549-570`

```cpp
MTPVector<MTPJSONObjectValue> SessionPrivate::prepareInitParams() {
    const auto local = QDateTime::currentDateTime();
    const auto utc = QDateTime(local.date(), local.time(), Qt::UTC);
    const auto shift = base::unixtime::now() - (TimeId)::time(nullptr);
    const auto delta = int(utc.toSecsSinceEpoch()) - int(local.toSecsSinceEpoch()) - shift;

    auto sliced = delta;
    while (sliced < -12 * 3600) sliced += 24 * 3600;
    while (sliced > 14 * 3600) sliced -= 24 * 3600;

    const auto sign = (sliced < 0) ? -1 : 1;
    const auto rounded = base::SafeRound(std::abs(sliced) / 900.) * 900 * sign;

    return MTP_vector<MTPJSONObjectValue>(
        1,
        MTP_jsonObjectValue(
            MTP_string("tz_offset"),
            MTP_jsonNumber(MTP_double(rounded))));
}
```

---

### 9. Proxy Information (Информация о прокси) - опционально

**Отправляется только если**: Используется MTProto прокси

**Содержит**:
- Хост прокси-сервера
- Порт прокси-сервера

**Формат**:
```cpp
MTP_inputClientProxy(
    MTP_string(_options->proxy.host),  // "proxy.example.com"
    MTP_int(_options->proxy.port))      // 443
```

---

## Откуда берется информация

### Windows
1. **Реестр Windows**:
   - `HKEY_LOCAL_MACHINE\HARDWARE\DESCRIPTION\System\BIOS`
   - Ключи: `SystemProductName`, `SystemFamily`, `BaseBoardProduct`

2. **Windows API**:
   - `IsWindows11OrGreater()`, `IsWindows10OrGreater()` - версия ОС
   - `IsWow64Process2()` - архитектура процессора
   - `GetCurrentProcess()` - информация о процессе

### Linux
1. **DMI/SMBIOS в sysfs**:
   - `/sys/class/dmi/id/product_name`
   - `/sys/class/dmi/id/product_family`
   - `/sys/class/dmi/id/board_name`
   - `/sys/class/dmi/id/chassis_type`

2. **Системные утилиты**:
   - `systemd-detect-virt` - определение виртуализации
   - `uname()` - информация о ядре

3. **Переменные окружения**:
   - `XDG_CURRENT_DESKTOP` - desktop environment
   - `WAYLAND_DISPLAY` / `DISPLAY` - window manager

4. **Системные библиотеки**:
   - `gnu_get_libc_version()` - версия glibc

### macOS
1. **system_profiler**:
   - Команда: `system_profiler -json SPHardwareDataType`
   - Поля: `machine_name`, `chip_type`

2. **sysctl**:
   - `hw.model` - модель устройства (например "MacBookAir10,1")

3. **Системные API**:
   - Версия macOS/OS X

### Все платформы
1. **Qt Framework**:
   - `QSysInfo::buildCpuArchitecture()` - архитектура CPU
   - `QSysInfo::prettyProductName()` - название ОС (fallback)
   - `QDateTime` - для вычисления timezone offset

2. **Системные локали**:
   - Язык системы
   - Региональные настройки

---

## Куда и как отправляется

### Место отправки
**Файл**: `Telegram/SourceFiles/mtproto/session_private.cpp`
**Функция**: `SessionPrivate::tryToSend()`
**Строки**: 693-705

### Момент отправки
- При каждой инициализации сессии с сервером
- При каждом переподключении
- При смене Data Center
- **НЕ отправляется**: Для CDN-серверов (вместо реальных данных отправляется "n/a")

### Метод MTProto
**Название**: `initConnection`

**Структура вызова**:
```cpp
initWrapper = MTPInitConnection<SerializedRequest>(
    MTP_flags(Flag::f_params | (mtprotoProxy ? Flag::f_proxy : Flag(0))),
    MTP_int(ApiId),                        // 2040
    MTP_string(deviceModel),               // "HP EliteBook 840"
    MTP_string(systemVersion),             // "Windows 11 x64"
    MTP_string(appVersion),                // "6.2.4 x64"
    MTP_string(systemLangCode),            // "en-US"
    MTP_string(langPackName),              // "tdesktop"
    MTP_string(cloudLangCode),             // "en"
    clientProxyFields,                     // опционально: proxy host + port
    MTP_jsonObject(prepareInitParams()),   // {"tz_offset": 10800}
    SerializedRequest());
```

### Схема данных (MTProto)
**TL-схема** (приблизительно):
```tl
initConnection#c1cd5ea9 {X:Type}
    flags:#
    api_id:int
    device_model:string
    system_version:string
    app_version:string
    system_lang_code:string
    lang_pack:string
    lang_code:string
    proxy:flags.0?InputClientProxy
    params:flags.1?JSONValue
    query:!X = X;
```

### Инициализация в приложении
**Файл**: `Telegram/SourceFiles/main/main_account.cpp:415-424`

```cpp
void Account::startMtp(std::unique_ptr<MTP::Config> config) {
    auto fields = base::take(_mtpFields);
    fields.config = std::move(config);
    fields.deviceModel = Platform::DeviceModelPretty();     // Получение device model
    fields.systemVersion = Platform::SystemVersionPretty(); // Получение system version
    _mtp = std::make_unique<MTP::Instance>(
        MTP::Instance::Mode::Normal,
        std::move(fields));
}
```

### Особые случаи

#### CDN серверы
**Код**: `Telegram/SourceFiles/mtproto/session_private.cpp:678-683`

```cpp
const auto deviceModel = (_currentDcType == DcType::Cdn)
    ? "n/a"
    : _instance->deviceModel();
const auto systemVersion = (_currentDcType == DcType::Cdn)
    ? "n/a"
    : _instance->systemVersion();
```

Для CDN (Content Delivery Network) серверов:
- `device_model` = `"n/a"`
- `system_version` = `"n/a"`

**Причина**: CDN-серверы используются только для загрузки медиа-файлов, им не нужна детальная информация о системе.

---

## Структура данных

### Таблица отправляемых параметров

| Параметр | Тип | Обязательный | Пример | Максимальная длина |
|----------|-----|--------------|--------|-------------------|
| api_id | int | ✅ Да | 2040 | - |
| device_model | string | ✅ Да | "HP EliteBook 840" | 15-32 символа |
| system_version | string | ✅ Да | "Windows 11 x64" | - |
| app_version | string | ✅ Да | "6.2.4 x64" | - |
| system_lang_code | string | ✅ Да | "ru-RU" | 5-10 символов |
| lang_pack | string | ✅ Да | "tdesktop" | - |
| lang_code | string | ✅ Да | "ru" | 2-5 символов |
| proxy | InputClientProxy | ❌ Нет | {host, port} | - |
| params | JSONValue | ✅ Да | {"tz_offset": 10800} | - |

### Пример полного запроса

**JSON представление** (концептуально):
```json
{
  "method": "initConnection",
  "api_id": 2040,
  "device_model": "HP EliteBook 840 G7",
  "system_version": "Windows 11 x64",
  "app_version": "6.2.4 x64",
  "system_lang_code": "en-US",
  "lang_pack": "tdesktop",
  "lang_code": "en",
  "params": {
    "tz_offset": 10800
  }
}
```

**С прокси**:
```json
{
  "method": "initConnection",
  "api_id": 2040,
  "device_model": "ThinkPad X1 Carbon",
  "system_version": "Linux 6.5.0 KDE Wayland glibc 2.38",
  "app_version": "6.2.4 x86_64 Flatpak",
  "system_lang_code": "ru-RU",
  "lang_pack": "tdesktop",
  "lang_code": "ru",
  "proxy": {
    "host": "proxy.example.com",
    "port": 443
  },
  "params": {
    "tz_offset": 10800
  }
}
```

---

## Код и файлы

### Структура кодовой базы

#### 1. Платформо-специфичный код (Device Model и System Version)

**Windows**:
- Файл: `Telegram/lib_base/base/platform/win/base_info_win.cpp`
- Функции:
  - `DeviceModelPretty()` (строки 154-186)
  - `SystemVersionPretty()` (строки 188-235)

**Linux**:
- Файл: `Telegram/lib_base/base/platform/linux/base_info_linux.cpp`
- Функции:
  - `DeviceModelPretty()` (строки 87-136)
  - `SystemVersionPretty()` (строки 138-183)

**macOS**:
- Файл: `Telegram/lib_base/base/platform/mac/base_info_mac.mm`
- Функции:
  - `DeviceFromSystemProfiler()` (строки 121-141)
  - `DeviceModelPretty()` (строки 145-166)
  - `SystemVersionPretty()` (строки 168-179)

**Общая логика**:
- Файл: `Telegram/lib_base/base/platform/base_platform_info.cpp`
- Функции:
  - `SimplifyDeviceModel()` - упрощение строки модели
  - `SimplifyGoodDeviceModel()` - обработка хороших моделей
  - `ProductNameToDeviceModel()` - преобразование названия продукта
  - `FinalizeDeviceModel()` - финализация с fallback
  - `IsDeviceModelOk()` - проверка корректности модели

#### 2. Инициализация MTP (сбор данных)

**Файл**: `Telegram/SourceFiles/main/main_account.cpp`
- Функция: `Account::startMtp()` (строки 415-424)
- Что делает:
  - Вызывает `Platform::DeviceModelPretty()`
  - Вызывает `Platform::SystemVersionPretty()`
  - Сохраняет в `fields.deviceModel` и `fields.systemVersion`
  - Создает MTP::Instance с этими полями

#### 3. MTProto сессия (отправка данных)

**Файл**: `Telegram/SourceFiles/mtproto/session_private.cpp`

**Функции**:
- `ComputeAppVersion()` (строки 79-99) - вычисление версии приложения
- `prepareInitParams()` (строки 549-570) - подготовка JSON параметров (timezone)
- `tryToSend()` (строки 671-708) - формирование и отправка initConnection

**Ключевой код отправки** (строки 693-705):
```cpp
initWrapper = MTPInitConnection<SerializedRequest>(
    MTP_flags(Flag::f_params | (mtprotoProxy ? Flag::f_proxy : Flag(0))),
    MTP_int(ApiId),
    MTP_string(deviceModel),
    MTP_string(systemVersion),
    MTP_string(appVersion),
    MTP_string(systemLangCode),
    MTP_string(langPackName),
    MTP_string(cloudLangCode),
    clientProxyFields,
    MTP_jsonObject(prepareInitParams()),
    SerializedRequest());
```

#### 4. MTP Instance (хранение данных)

**Файл**: `Telegram/SourceFiles/mtproto/mtp_instance.cpp`
- Класс: `Instance::Private`
- Методы:
  - `deviceModel()` const (строка 87) - возвращает сохраненную модель
  - `systemVersion()` const (строка 88) - возвращает сохраненную версию

#### 5. Настройки приложения

**Файл**: `Telegram/SourceFiles/core/core_settings.cpp`
- Функция: `Settings::deviceModel()` (строки 1194-1197)
- Логика:
  - Проверяет наличие пользовательской модели устройства
  - Если есть - возвращает её
  - Если нет - вызывает `Platform::DeviceModelPretty()`

**Файл**: `Telegram/SourceFiles/core/core_settings.h`
- Поле: `rpl::variable<QString> _customDeviceModel` (строка 1064)
- Хранит пользовательскую модель устройства

#### 6. Версия приложения

**Файл**: `Telegram/SourceFiles/core/version.h`
```cpp
constexpr auto AppVersion = 6002004;
constexpr auto AppVersionStr = "6.2.4";
constexpr auto AppBetaVersion = false;
```

#### 7. API ID

**Файл**: `Telegram/SourceFiles/config.h`
```cpp
constexpr auto ApiId = 2040;  // Стандартный Desktop
```

**Файл**: `Telegram/SourceFiles/api/api_authorizations.cpp`
```cpp
constexpr auto TestApiId = 17349;
constexpr auto SnapApiId = 611335;
constexpr auto DesktopApiId = 2040;
```

### Дерево вызовов

```
main_account.cpp: Account::startMtp()
    ↓
    Platform::DeviceModelPretty()
    Platform::SystemVersionPretty()
    ↓
    MTP::Instance создается с fields
    ↓
session_private.cpp: SessionPrivate::tryToSend()
    ↓
    _instance->deviceModel()
    _instance->systemVersion()
    ComputeAppVersion()
    prepareInitParams()
    ↓
    MTPInitConnection<SerializedRequest>(...)
    ↓
    Отправка на сервер Telegram
```

### Список всех файлов

1. `Telegram/lib_base/base/platform/base_platform_info.h` - заголовочный файл с объявлениями
2. `Telegram/lib_base/base/platform/base_platform_info.cpp` - общая логика обработки
3. `Telegram/lib_base/base/platform/win/base_info_win.cpp` - Windows специфичный код
4. `Telegram/lib_base/base/platform/linux/base_info_linux.cpp` - Linux специфичный код
5. `Telegram/lib_base/base/platform/mac/base_info_mac.mm` - macOS специфичный код
6. `Telegram/SourceFiles/main/main_account.cpp` - инициализация аккаунта и MTP
7. `Telegram/SourceFiles/mtproto/session_private.cpp` - MTProto сессия и отправка
8. `Telegram/SourceFiles/mtproto/session_private.h` - заголовочный файл сессии
9. `Telegram/SourceFiles/mtproto/mtp_instance.cpp` - экземпляр MTP
10. `Telegram/SourceFiles/mtproto/mtp_instance.h` - заголовочный файл MTP
11. `Telegram/SourceFiles/core/core_settings.cpp` - настройки приложения
12. `Telegram/SourceFiles/core/core_settings.h` - заголовочный файл настроек
13. `Telegram/SourceFiles/core/version.h` - версия приложения
14. `Telegram/SourceFiles/config.h` - конфигурация (API ID)
15. `Telegram/SourceFiles/api/api_authorizations.cpp` - обработка авторизаций

---

## Можно ли изменить

### ✅ Что МОЖНО изменить

#### 1. Device Model (Модель устройства)
**Как**: Через настройки приложения (требуется UI или изменение конфига)

**Код**: `Core::Settings::_customDeviceModel`

**Где хранится**: В настройках приложения

**Применение**: Значение берется из `Settings::deviceModel()`:
```cpp
QString Settings::deviceModel() const {
    const auto custom = customDeviceModel();
    return custom.isEmpty() ? Platform::DeviceModelPretty() : custom;
}
```

**Способы изменения**:
1. Через UI настроек (если реализовано)
2. Через прямое редактирование конфига/базы данных
3. Модификация кода для принудительной установки значения

#### 2. Language Code (Язык в приложении)
**Как**: Через настройки языка в Telegram Desktop

**Где**: Settings → Language

**Применение**: Сразу же при следующем подключении

#### 3. API ID
**Как**: Изменение `config.h` перед компиляцией

**Файл**: `Telegram/SourceFiles/config.h`

**Риски**:
- Требуется перекомпиляция
- Может привести к блокировке аккаунта
- Нарушает условия использования API

#### 4. App Version
**Как**: Изменение `version.h` перед компиляцией

**Файл**: `Telegram/SourceFiles/core/version.h`

**Риски**:
- Требуется перекомпиляция
- Может вызвать проблемы совместимости
- Серверы могут отклонять устаревшие версии

### ⚠️ Что можно изменить только через модификацию кода

#### 1. Device Model (постоянно)
**Способ**: Изменить функцию `DeviceModelPretty()` для возврата нужного значения

**Пример модификации** (`base_info_win.cpp`):
```cpp
QString DeviceModelPretty() {
    return QString("Custom Device Name");  // Ваше значение
}
```

#### 2. System Version
**Способ**: Изменить функцию `SystemVersionPretty()`

**Пример модификации** (`base_info_win.cpp`):
```cpp
QString SystemVersionPretty() {
    return QString("Custom OS 1.0");  // Ваше значение
}
```

#### 3. Timezone Offset
**Способ**: Изменить функцию `prepareInitParams()`

**Пример модификации** (`session_private.cpp`):
```cpp
MTPVector<MTPJSONObjectValue> SessionPrivate::prepareInitParams() {
    return MTP_vector<MTPJSONObjectValue>(
        1,
        MTP_jsonObjectValue(
            MTP_string("tz_offset"),
            MTP_jsonNumber(MTP_double(0))));  // Всегда UTC
}
```

### ❌ Что НЕЛЬЗЯ изменить/отключить

#### 1. Отправку initConnection
**Почему**: Это обязательная часть MTProto протокола

**Последствия отключения**: Сервер не примет подключение

#### 2. Отправку без модификации исходного кода
**Почему**: Значения жёстко зашиты в код и собираются автоматически

**Что требуется**: Модификация исходного кода и перекомпиляция

#### 3. Системный язык (system_lang_code)
**Почему**: Берётся напрямую из ОС через Qt

**Альтернатива**: Изменить язык в системных настройках ОС

### Методы изменения данных

#### Метод 1: Через настройки (только Device Model)
**Плюсы**:
- Не требует перекомпиляции
- Официальный способ
- Безопасно

**Минусы**:
- Ограниченные возможности
- Не все параметры доступны

#### Метод 2: Модификация исходного кода
**Плюсы**:
- Полный контроль над всеми параметрами
- Можно изменить любые значения

**Минусы**:
- Требуется перекомпиляция
- Нарушение условий использования
- Риск блокировки аккаунта
- Сложность для непрограммистов

**Пример патча** (для всех трёх платформ):

**1. Windows** (`base_info_win.cpp`):
```cpp
QString DeviceModelPretty() {
    return QString("Desktop");  // Вместо реального определения
}

QString SystemVersionPretty() {
    return QString("Windows 10 x64");  // Фиксированное значение
}
```

**2. Linux** (`base_info_linux.cpp`):
```cpp
QString DeviceModelPretty() {
    return QString("Desktop");
}

QString SystemVersionPretty() {
    return QString("Linux");
}
```

**3. macOS** (`base_info_mac.mm`):
```cpp
QString DeviceModelPretty() {
    return QString("Mac");
}

QString SystemVersionPretty() {
    return QString("macOS 14.0");
}
```

**4. Timezone** (`session_private.cpp`):
```cpp
MTPVector<MTPJSONObjectValue> SessionPrivate::prepareInitParams() {
    return MTP_vector<MTPJSONObjectValue>(
        1,
        MTP_jsonObjectValue(
            MTP_string("tz_offset"),
            MTP_jsonNumber(MTP_double(0))));  // Всегда UTC
}
```

**5. App Version** (`session_private.cpp`):
```cpp
QString ComputeAppVersion() {
    return QString("6.2.4");  // Без архитектуры и источника
}
```

#### Метод 3: Runtime патчинг (продвинутый)
**Описание**: Изменение памяти процесса во время выполнения

**Инструменты**:
- Binary patching
- Memory editors
- Debuggers (GDB, LLDB, WinDbg)
- Frida, x64dbg

**Плюсы**:
- Не требует перекомпиляции
- Можно изменять "на лету"

**Минусы**:
- Очень сложно
- Требует глубоких знаний
- Нестабильно
- Может нарушать защиту

### Рекомендации

1. **Для обычных пользователей**: Не рекомендуется изменять эти данные, так как это может привести к проблемам с аккаунтом.

2. **Для исследователей**: Изменения через модификацию кода безопасны для изучения протокола, но не используйте их на основных аккаунтах.

3. **Для разработчиков**: Если нужно скрыть реальную модель устройства, используйте официальный способ через `Core::Settings::_customDeviceModel`.

4. **Для приватности**: Помните, что даже изменив device_model, другие параметры (system_version, timezone, языки) всё равно отправляются и могут идентифицировать вас.

---

## Приложение: Примеры реальных данных

### Пример 1: Windows Desktop (HP Laptop)
```
api_id: 2040
device_model: "HP EliteBook 840 G7"
system_version: "Windows 11 x64"
app_version: "6.2.4 x64"
system_lang_code: "en-US"
lang_pack: "tdesktop"
lang_code: "en"
params: {"tz_offset": -18000}
```

### Пример 2: Linux Desktop (Flatpak)
```
api_id: 2040
device_model: "ThinkPad X1 Carbon Gen 9"
system_version: "Linux 6.5.0 KDE Wayland glibc 2.38"
app_version: "6.2.4 x86_64 Flatpak"
system_lang_code: "en-US"
lang_pack: "tdesktop"
lang_code: "en"
params: {"tz_offset": -18000}
```

### Пример 3: macOS (App Store)
```
api_id: 2040
device_model: "MacBook Air M1"
system_version: "macOS 14.0"
app_version: "6.2.4 Mac App Store"
system_lang_code: "en-US"
lang_pack: "tdesktop"
lang_code: "en"
params: {"tz_offset": -18000}
```

### Пример 4: Windows (Microsoft Store)
```
api_id: 2040
device_model: "Surface Laptop 4"
system_version: "Windows 11 arm64"
app_version: "6.2.4 arm64 Microsoft Store"
system_lang_code: "en-US"
lang_pack: "tdesktop"
lang_code: "en"
params: {"tz_offset": -18000}
```

### Пример 5: Linux (Snap, с прокси)
```
api_id: 611335
device_model: "Desktop"
system_version: "Linux 5.15.0 GNOME X11 glibc 2.35"
app_version: "6.2.4 x86_64 Snap"
system_lang_code: "ru-RU"
lang_pack: "tdesktop"
lang_code: "ru"
proxy: {
  "host": "proxy.example.com",
  "port": 443
}
params: {"tz_offset": 10800}
```

### Пример 6: Generic Linux (без DMI)
```
api_id: 2040
device_model: "Laptop"
system_version: "Linux 6.2.0 XFCE X11 glibc 2.37"
app_version: "6.2.4 aarch64"
system_lang_code: "de-DE"
lang_pack: "tdesktop"
lang_code: "de"
params: {"tz_offset": 3600}
```

---

## Заключение

Telegram Desktop собирает и отправляет **детальную информацию о вашей системе** при каждом подключении:

### Что точно отправляется:
1. ✅ Модель устройства (из BIOS/DMI/system_profiler)
2. ✅ Версия ОС и архитектура
3. ✅ Версия приложения и источник установки
4. ✅ Языки системы и приложения
5. ✅ Часовой пояс (с точностью до 15 минут)
6. ✅ Информация о прокси (если используется)

### Почему это делается:
- Идентификация клиента на сервере
- Отображение активных сессий в настройках
- Статистика использования
- Возможное определение ботов/автоматизации
- Безопасность (обнаружение подозрительных подключений)

### Можно ли это изменить:
- ⚠️ Частично - только модель устройства через настройки
- 🔧 Полностью - только через модификацию исходного кода и перекомпиляцию
- ❌ Отключить - нельзя, это часть протокола MTProto

### Риски приватности:
- Уникальная комбинация параметров может идентифицировать пользователя
- Даже с изменённой моделью устройства, другие параметры остаются
- Часовой пояс может указать на ваше местоположение

---

*Документ создан на основе анализа исходного кода Telegram Desktop версии 6.2.4*

*Дата анализа: 2025-10-24*

*Платформы: Windows, Linux, macOS*
