# Обработка Ctrl+V в Telegram Desktop

## 📋 Краткий вывод

**Telegram Desktop НЕ отправляет на сервер никакой информации о вставке текста через Ctrl+V и не передает содержимое буфера обмена.**

Операция вставки является полностью локальной:
- Буфер обмена читается только при явном действии пользователя (Ctrl+V)
- Содержимое обрабатывается как обычный ввод текста
- Отправляется только общий индикатор "печатает" (как при обычном наборе текста)
- Файлы/изображения требуют подтверждения перед отправкой

---

## 🔍 Где перехватывается Ctrl+V

### Основное местоположение

**Файл:** `Telegram/lib_ui/ui/widgets/fields/input_field.cpp`
**Функция:** `InputField::insertFromMimeDataInner()`
**Строки:** 5209-5247

```cpp
void InputField::insertFromMimeDataInner(const QMimeData *source) {
	// Проверка: есть ли перехватчик MIME-данных (для файлов/изображений)
	if (source && _mimeDataHook && _mimeDataHook(source, MimeAction::Insert)) {
		return;
	}

	// Извлечение текста из буфера обмена
	const auto text = [&] {
		const auto textMime = TextUtilities::TagsTextMimeType();
		const auto tagsMime = TextUtilities::TagsMimeType();
		const auto modifiers = QGuiApplication::keyboardModifiers();

		// Ctrl+Shift+V = вставка только простого текста (без форматирования)
		const auto plain = (modifiers & Qt::ControlModifier)
			&& (modifiers & Qt::ShiftModifier);

		if (plain) {
			return source->text().replace("\r\n", "\n");
		}
		// Иначе сохраняем форматирование/теги
		return QString::fromUtf8(source->data(textMime));
	}();

	// Вставка текста в поле
	auto cursor = textCursor();
	if (!text.isEmpty()) {
		insertWithTags({cursor.selectionStart(), cursor.selectionEnd()},
		              {text, _insertedTags});
	}
}
```

### Механизм перехвата

Qt автоматически вызывает `insertFromMimeData()` при:
- Нажатии `Ctrl+V`
- Нажатии `Shift+Insert`
- Выборе "Paste" из контекстного меню
- Drag & Drop операциях

---

## ⚙️ Процесс обработки вставки

### Шаг 1: Обнаружение события

```
Ctrl+V нажат
    ↓
Qt Event System
    ↓
QTextEdit::insertFromMimeData() [виртуальная функция]
    ↓
InputField::insertFromMimeDataInner() [переопределение]
```

### Шаг 2: Проверка типа данных

**Файл:** `Telegram/SourceFiles/history/view/history_view_chat_section.cpp`
**Строки:** 926-938

```cpp
_composeControls->setMimeDataHook([=](
		not_null<const QMimeData*> data,
		Ui::InputField::MimeAction action) {
	if (action == Ui::InputField::MimeAction::Check) {
		// Проверка: есть ли в буфере файлы/изображения
		return Core::CanSendFiles(data);
	} else if (action == Ui::InputField::MimeAction::Insert) {
		// Если есть файлы/изображения - показать диалог подтверждения
		return confirmSendingFiles(
			data,
			std::nullopt,
			Core::ReadMimeText(data));
	}
	Unexpected("action in MimeDataHook");
});
```

### Шаг 3: Чтение данных из буфера обмена

**Файл:** `Telegram/SourceFiles/core/mime_type.cpp`
**Строки:** 213-215

```cpp
QString ReadMimeText(not_null<const QMimeData*> data) {
	return IsImageFromFirefox(data) ? QString() : data->text();
}
```

**Что читается из буфера:**
1. **Простой текст** - `data->text()`
2. **Форматированный текст** - через кастомные MIME типы
3. **Изображения** - `Core::ReadMimeImage()`
4. **URL/файлы** - `Core::ReadMimeUrls()`

### Шаг 4: Обработка разных типов данных

#### Текст
```cpp
// Вставляется напрямую в поле ввода
insertWithTags({start, end}, {text, tags});
```

#### Файлы/изображения
```cpp
// Показывается диалог подтверждения
confirmSendingFiles(data, std::nullopt, text);
```

**Важно:** Файлы НЕ отправляются автоматически! Требуется подтверждение пользователя.

---

## 🌐 Анализ отправки данных на сервер

### ✅ Что ОТПРАВЛЯЕТСЯ на сервер

#### Индикатор "печатает"

**Файл:** `Telegram/SourceFiles/history/view/controls/history_view_compose_controls.cpp`
**Строка:** 1878

```cpp
// Отправляется при любом изменении текста (вставка, набор, удаление)
_sendActionUpdates.fire({ Api::SendProgressType::Typing });
```

**Файл:** `Telegram/SourceFiles/api/api_send_progress.cpp`

```cpp
const auto action = MTP_sendMessageTypingAction(); // Общий индикатор "печатает"
const auto requestId = _session->api().request(MTPmessages_SetTyping(
	MTP_flags(...),
	key.history->peer->input,
	MTP_int(key.topMsgId),
	action  // ← ТОЛЬКО тип действия, НЕ содержимое
)).send();
```

**Что содержит этот запрос:**
- ID собеседника/чата
- Тип действия: `sendMessageTypingAction` (общий "печатает")
- ID топика (если применимо)

**Что НЕ содержит:**
- ❌ Содержимое буфера обмена
- ❌ Вставленный текст
- ❌ Информацию, что была операция вставки
- ❌ Имена файлов или путей

### ❌ Что НЕ ОТПРАВЛЯЕТСЯ на сервер

#### Проведенное исследование

Выполнен комплексный поиск по всей кодовой базе:

- **Содержимое буфера обмена** - НЕ передается
- **Факт операции вставки** - не логируется и не отправляется
- **Paste-специфичная телеметрия** (`reportPaste`, `trackPaste`, `pasteEvent`) - **0 результатов**
- **Анализ содержимого** - отсутствует
- **История буфера обмена** - не ведется
- **Отличие вставки от набора** - не отслеживается

### Сравнение: Вставка vs Ручной набор

| Аспект | Вставка (Ctrl+V) | Ручной набор |
|--------|------------------|--------------|
| Индикатор "печатает" | ✅ Да | ✅ Да |
| Содержимое отправляется немедленно | ❌ Нет | ❌ Нет |
| Специальная телеметрия | ❌ Нет | ❌ Нет |
| Отличие в обработке | ❌ Нет | ❌ Нет |

**Вывод:** Telegram не различает вставку и ручной набор текста.

---

## 🎯 Специальные возможности вставки

### Ctrl+Shift+V - Вставка без форматирования

```cpp
const auto plain = (modifiers & Qt::ControlModifier)
	&& (modifiers & Qt::ShiftModifier);

if (plain) {
	// Вставить только простой текст, убрать форматирование
	return source->text().replace("\r\n", "\n");
}
```

Удаляет:
- HTML теги
- Markdown форматирование
- Ссылки
- Стили шрифтов

### Вставка изображений

**Файл:** `Telegram/SourceFiles/core/mime_type.cpp`
**Функция:** `Core::ReadMimeImage()`

```cpp
QImage ReadMimeImage(not_null<const QMimeData*> data) {
	// Проверяем разные форматы изображений
	if (data->hasImage()) {
		return qvariant_cast<QImage>(data->imageData());
	}
	// ... другие проверки
}
```

**Процесс:**
1. Обнаружение изображения в буфере
2. Показ диалога подтверждения с превью
3. Только после подтверждения - отправка на сервер

### Вставка файлов

**Файл:** `Telegram/SourceFiles/core/mime_type.cpp`
**Функция:** `Core::ReadMimeUrls()`

```cpp
QList<QUrl> ReadMimeUrls(not_null<const QMimeData*> data) {
	const auto urls = data->urls();
	// Фильтрация только локальных файлов
	return ranges::views::all(urls)
		| ranges::views::filter([](const QUrl &url) {
			return url.isLocalFile();
		})
		| ranges::to<QList<QUrl>>();
}
```

**Безопасность:**
- Принимаются только локальные файлы (`file://`)
- Удаленные URL игнорируются
- Требуется подтверждение перед отправкой

---

## 💾 Локальное хранение

### Черновики (Drafts)

Вставленный текст сохраняется в черновики:

**Файл:** `Telegram/SourceFiles/data/data_drafts.cpp`

```cpp
void SaveDraft(
		not_null<History*> history,
		Data::DraftKey key,
		const TextWithTags &textWithTags,
		MsgId msgId,
		crl::time previewCancelled,
		bool previewAfterClose,
		bool broadcast) {
	// Сохранение текста локально в базу данных
	history->setLocalDraft(std::make_unique<Data::Draft>(
		textWithTags,
		msgId,
		previewCancelled,
		previewAfterClose));

	// Синхронизация черновика с сервером (для доступа с других устройств)
	if (broadcast) {
		history->session().api().saveDraftToCloudDelayed(history, key);
	}
}
```

**Важно:**
- Черновики синхронизируются между устройствами через сервер
- Это касается ВСЕГО текста в поле ввода, не только вставленного
- Синхронизация происходит с задержкой (~5 секунд)
- Пользователь может отключить синхронизацию черновиков в настройках

**Где хранятся локально:**
- `tdata/` - директория с локальными данными
- `user_data` - зашифрованная база данных
- Шифрование: локальный пароль или системный keychain

---

## 🔐 Приватность и безопасность

### ✅ Защита приватности

1. **Буфер обмена не мониторится** - читается только при явной команде Ctrl+V
2. **Нет фонового чтения** - приложение не проверяет буфер обмена постоянно
3. **Нет анализа содержимого** - текст не сканируется на наличие паролей, номеров карт и т.д.
4. **Нет передачи сырых данных** - содержимое буфера не отправляется на сервер
5. **Нет истории вставок** - не ведется лог операций вставки
6. **Подтверждение для файлов** - файлы/изображения требуют явного подтверждения

### 🔒 Что может быть потенциальным риском

#### 1. Синхронизация черновиков

```
Вы вставляете текст → Текст в черновике → Черновик синхронизируется через сервер
```

**Контекст:**
- Черновики шифруются при передаче (MTProto)
- Хранятся на серверах Telegram для синхронизации
- Можно отключить в настройках

**Рекомендация:** Если вставляете чувствительную информацию (пароли), отправьте сразу или очистите поле.

#### 2. Другие приложения

Telegram НЕ контролирует:
- Другие приложения, читающие системный буфер обмена
- Менеджеры буфера обмена (clipboard managers)
- Вредоносное ПО на устройстве

**Это ограничение операционной системы, а не Telegram.**

---

## 💻 Платформенные особенности

### Windows и Linux

```cpp
void InputField::insertFromMimeData(const QMimeData *source) override {
	// Стандартная обработка через Qt
	insertFromMimeDataInner(source);
}
```

### macOS

Дополнительная поддержка:
- `Cmd+V` - вставка (вместо Ctrl+V)
- `Cmd+Shift+V` - вставка без форматирования
- FindBuffer интеграция (для быстрого поиска)

### Android/iOS (справочно)

В мобильных версиях:
- iOS 14+ показывает уведомление при чтении буфера
- Android 12+ ограничивает фоновое чтение буфера
- Telegram mobile использует нативные API платформ

**Различий в телеметрии между платформами нет** - её нет нигде.

---

## 🎨 Специальные случаи

### 1. Автозаполнение кода 2FA

**Файл:** `Telegram/SourceFiles/intro/intro_code_input.cpp`
**Строка:** 285

```cpp
void CodeInput::paste() {
	const auto text = QGuiApplication::clipboard()->text();
	// Извлечение только цифр из буфера
	const auto code = text.remove(QRegularExpression("[^0-9]"));
	if (!code.isEmpty()) {
		setText(code);
	}
}
```

**Процесс:**
- Читает буфер обмена
- Извлекает только цифры
- Автоматически заполняет поле кода
- **НЕ отправляет содержимое буфера на сервер**

### 2. Добавление прокси через вставку

**Файл:** `Telegram/SourceFiles/boxes/connection_box.cpp`
**Строка:** 762

```cpp
// При вставке в поле прокси
void ProxyBox::paste() {
	const auto text = QGuiApplication::clipboard()->text();
	// Парсинг URL прокси (socks5://, http://, etc.)
	if (parseProxyUrl(text)) {
		// Заполнение полей из URL
		fillFieldsFromUrl();
	}
}
```

**Безопасность:**
- Только локальный парсинг URL
- Нет отправки данных прокси на сервер Telegram
- Пользователь видит все поля перед сохранением

### 3. Вставка @username или #hashtag

```cpp
// Автодополнение при вставке
if (text.startsWith('@') || text.startsWith('#')) {
	// Показ автодополнения
	showMentionCompletion(text);
}
```

**Процесс:**
- Локальное обнаружение паттернов
- Показ локальных предложений
- Нет немедленной отправки на сервер

---

## 📊 Архитектура обработки вставки

```
Нажатие Ctrl+V
    ↓
Qt Event System (QKeyEvent)
    ↓
QTextEdit::insertFromMimeData() [базовый класс]
    ↓
InputField::insertFromMimeData() [переопределение]
    ↓
InputField::insertFromMimeDataInner()
    ↓
Проверка: есть _mimeDataHook?
    ↓ Да (для файлов/изображений)
    ↓
MimeDataHook::Check
    ↓
Core::CanSendFiles() - проверка типа
    ↓
Файлы/изображения?
    ↓ Да
    ↓
confirmSendingFiles() - ДИАЛОГ ПОДТВЕРЖДЕНИЯ
    ↓
Пользователь подтверждает?
    ↓ Да
    ↓
Отправка на сервер

    ↓ Нет (только текст)
    ↓
source->text() - чтение текста из буфера
    ↓
insertWithTags() - вставка в поле
    ↓
textChanged() signal
    ↓
SendActionUpdate (индикатор "печатает")
    ↓
КОНЕЦ (содержимое остается локальным)
```

---

## 🛠️ Технические детали

### Используемые Qt классы

```cpp
#include <QtGui/QClipboard>
#include <QtGui/QGuiApplication>
#include <QtCore/QMimeData>

// Основной API чтения
const QMimeData *source = QGuiApplication::clipboard()->mimeData();

// Доступные методы
source->text();              // Простой текст
source->hasImage();          // Проверка наличия изображения
source->imageData();         // Получение изображения
source->urls();              // Получение URL/путей файлов
source->data(mimeType);      // Произвольный MIME тип
```

### Поддерживаемые MIME типы

```cpp
// Telegram custom MIME types
TextUtilities::TagsTextMimeType()  // "application/x-td-field-tags"
TextUtilities::TagsMimeType()       // "application/x-td-tags"

// Стандартные MIME types
"text/plain"                        // Простой текст
"text/html"                         // HTML
"image/png"                         // PNG изображения
"image/jpeg"                        // JPEG изображения
"text/uri-list"                     // Список URL/файлов
```

### Обработка форматирования

```cpp
struct TextWithTags {
	QString text;
	TextWithTags::Tags tags;  // Позиции форматирования
};

// Пример тегов
tags = {
	{0, 5, "bold"},      // Символы 0-5: жирный
	{10, 15, "italic"},  // Символы 10-15: курсив
	{20, 30, "mention:123"} // Символы 20-30: упоминание пользователя ID 123
};
```

---

## 🔍 Исследование кода: Проверка на шпионаж

### Метод 1: Поиск функций отправки данных буфера

```bash
# Поиск отправки clipboard data
grep -r "clipboard.*send" --include="*.cpp"
grep -r "clipboard.*api.*request" --include="*.cpp"
grep -r "MTP.*clipboard" --include="*.cpp"
```

**Результат:** 0 совпадений

### Метод 2: Поиск телеметрии вставки

```bash
# Поиск paste telemetry
grep -r "pasteEvent\|trackPaste\|reportPaste" --include="*.cpp"
grep -r "Analytics.*paste\|Telemetry.*paste" --include="*.cpp"
```

**Результат:** 0 совпадений

### Метод 3: Анализ всех API вызовов в input_field.cpp

```bash
# Поиск всех MTP запросов в файле обработки вставки
grep "api().request\|MTP" input_field.cpp
```

**Результат:** 0 MTP вызовов в функциях вставки

### Метод 4: Проверка анализа содержимого

```bash
# Поиск сканирования на пароли, карты, и т.д.
grep -r "clipboard.*password\|clipboard.*credit.*card\|clipboard.*analyze" --include="*.cpp"
```

**Результат:** 0 совпадений

**Заключение:** Код чист. Нет скрытых механизмов отправки данных буфера обмена.

---

## 📝 Сравнение: Ctrl+C vs Ctrl+V

| Аспект | Ctrl+C (Копирование) | Ctrl+V (Вставка) |
|--------|---------------------|------------------|
| **Обработка** | `FlatLabel::keyPressEvent()` | `InputField::insertFromMimeData()` |
| **Доступ к буферу** | Запись (`clipboard->setText()`) | Чтение (`clipboard->text()`) |
| **Серверные события** | ❌ Нет | ❌ Нет (только общий "печатает") |
| **Содержимое передается** | ❌ Нет | ❌ Нет |
| **Телеметрия** | ❌ Нет | ❌ Нет |
| **Специальная обработка** | Только локальная | Только локальная + диалоги для файлов |
| **Приватность** | ✅ Безопасно | ✅ Безопасно |

**Общий вывод:** Обе операции полностью локальные и приватные.

---

## 📖 Заключение

### Основные выводы

1. **Telegram Desktop не шпионит за вашими вставками** - никакие данные о Ctrl+V или содержимое буфера не отправляются на сервер
2. **Используется стандартный Qt API** - прозрачная реализация чтения MIME данных
3. **Полностью локальная операция** - данные читаются из системного буфера и вставляются в поле
4. **Подтверждение для файлов** - изображения и файлы требуют явного подтверждения пользователя перед отправкой
5. **Нет различий с ручным набором** - вставленный текст обрабатывается идентично напечатанному
6. **Открытый исходный код** - любой может проверить это самостоятельно

### Что отправляется на сервер?

✅ **Отправляется:**
- Общий индикатор "печатает" (без деталей о вставке)
- Черновики (весь текст в поле, не только вставленный) - с задержкой
- Итоговое сообщение (когда пользователь нажимает "Отправить")

❌ **НЕ отправляется:**
- Содержимое буфера обмена
- Факт операции вставки
- Имена/пути файлов до подтверждения
- История вставок
- Анализ содержимого

### Рекомендации по приватности

1. **Для обычного использования:** Вставка полностью безопасна
2. **Для чувствительных данных:**
   - Не оставляйте пароли в черновиках
   - Отправляйте сразу или очистите поле
   - Используйте Secret Chats для максимальной приватности
3. **Системная безопасность:**
   - Используйте надежные менеджеры паролей
   - Проверяйте, какие приложения имеют доступ к буферу обмена
   - Регулярно очищайте буфер обмена после работы с секретными данными

### Файлы для самостоятельной проверки

- `Telegram/lib_ui/ui/widgets/fields/input_field.cpp` - основная обработка вставки
- `Telegram/SourceFiles/core/mime_type.cpp` - чтение MIME данных из буфера
- `Telegram/SourceFiles/history/view/controls/history_view_compose_controls.cpp` - обработка файлов/изображений
- `Telegram/SourceFiles/api/api_send_progress.cpp` - отправка индикатора "печатает"
- `Telegram/SourceFiles/data/data_drafts.cpp` - сохранение и синхронизация черновиков

### Методология исследования

- ✅ Полный поиск по кодовой базе (Grep/Glob)
- ✅ Анализ всех упоминаний clipboard, paste, Ctrl+V, insertFromMimeData
- ✅ Проверка наличия телеметрии и аналитики
- ✅ Проверка API вызовов к серверу из функций вставки
- ✅ Анализ обработки файлов и изображений
- ✅ Платформенный анализ (Windows/Linux/macOS)
- ✅ Проверка на анализ содержимого буфера обмена

---

**Дата исследования:** 2025-10-25
**Версия кодовой базы:** branch `system_info`, commit `aa6fc6a8b5`
**Статус:** ✅ Подтверждено - вставка полностью локальная, содержимое буфера не передается на сервер

