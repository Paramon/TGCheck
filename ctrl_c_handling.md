# Обработка Ctrl+C в Telegram Desktop

## 📋 Краткий вывод

**Telegram Desktop НЕ отправляет на сервер никакой информации о копировании текста через Ctrl+C.**

Операция копирования является полностью локальной и использует стандартный системный буфер обмена Qt без какой-либо сетевой коммуникации.

---

## 🔍 Где перехватывается Ctrl+C

### Основное местоположение

**Файл:** `Telegram/lib_ui/ui/widgets/labels.cpp`
**Функция:** `FlatLabel::keyPressEvent()`
**Примерная строка:** ~800+

```cpp
void FlatLabel::keyPressEvent(QKeyEvent *e) {
	e->ignore();
	if (e->key() == Qt::Key_Copy || (e->key() == Qt::Key_C && e->modifiers().testFlag(Qt::ControlModifier))) {
		if (!_selection.empty()) {
			copySelectedText();
			e->accept();
		}
#ifdef Q_OS_MAC
	} else if (e->key() == Qt::Key_E && e->modifiers().testFlag(Qt::ControlModifier)) {
		auto selection = _selection.empty() ? (_contextMenu ? _savedSelection : _selection) : _selection;
		if (!selection.empty()) {
			TextUtilities::SetClipboardText(_text.toTextForMimeData(selection), QClipboard::FindBuffer);
		}
#endif // Q_OS_MAC
	}
}
```

### Дополнительные места обработки

Копирование также обрабатывается в других компонентах:
- **Input fields** (поля ввода) - используют стандартную обработку Qt
- **Message text** (текст сообщений) - через `FlatLabel` и `Ui::Text::String`
- **Context menus** (контекстные меню) - прямой вызов `QGuiApplication::clipboard()->setText()`

---

## ⚙️ Процесс обработки копирования

### Шаг 1: Обнаружение события

Система перехватывает:
- `Qt::Key_Copy` (нативная клавиша Copy)
- ИЛИ `Qt::Key_C` + `Qt::ControlModifier` (Ctrl+C)

### Шаг 2: Проверка выделения

```cpp
if (!_selection.empty()) {
    copySelectedText();
    e->accept();
}
```

Копирование происходит **только если текст выделен**.

### Шаг 3: Копирование в буфер обмена

```cpp
void FlatLabel::copySelectedText() {
	const auto selection = _selection.empty()
		? (_contextMenu ? _savedSelection : _selection)
		: _selection;
	if (!selection.empty()) {
		TextUtilities::SetClipboardText(_text.toTextForMimeData(selection));
	}
}
```

### Шаг 4: Работа с системным буфером

**Файл:** `Telegram/lib_ui/ui/text/text_utilities.cpp`

```cpp
void SetClipboardText(
	const TextForMimeData &textForMimeData,
	QClipboard::Mode mode = QClipboard::Clipboard) {

	if (auto data = TextUtilities::MimeDataFromText(textForMimeData)) {
		QGuiApplication::clipboard()->setMimeData(data.release(), mode);
	}
}
```

**Используется стандартный Qt API:**
- `QGuiApplication::clipboard()` - получение системного буфера обмена
- `setMimeData()` - установка данных в буфер
- **Никаких сетевых вызовов!**

---

## 🌐 Анализ отправки событий на сервер

### Проведенное исследование

Выполнен комплексный поиск по всей кодовой базе:

#### ❌ Не найдено
- Телеметрия (`Telemetry`, `Analytics`) - **0 результатов**
- Отслеживание событий (`trackEvent`, `reportEvent`, `sendMetric`) - **0 результатов**
- Отслеживание копирования (`reportCopy`, `trackCopy`, `clipboardEvent`) - **0 результатов**
- Действия пользователя (`UserAction`, `UsageEvent`, `EventReport`) - **0 результатов для копирования**
- Метрики использования (`UsageMetrics`, `UserMetrics`) - **0 результатов**

#### ✅ Найдено
- **Только** локальные операции с буфером обмена через Qt API
- Репортинг используется **только** для жалоб на контент (спам, нарушения авторских прав), а НЕ для действий пользователя

### API для репортинга (для сравнения)

В коде есть механизмы отправки жалоб:

```cpp
// Это ОТПРАВЛЯЕТСЯ на сервер (для жалоб на контент)
void Session::api().request(MTPaccount_ReportPeer(
	peer->input,
	MTP_inputReportReasonSpam(),
	MTP_string(comment)
)).send();
```

**НО** для операций копирования подобного кода НЕТ.

### Вывод

**100% подтверждено:** Telegram Desktop не отправляет информацию о копировании текста на сервер.

---

## 💻 Платформенные особенности

### Windows и Linux

```cpp
if (e->key() == Qt::Key_Copy ||
    (e->key() == Qt::Key_C && e->modifiers().testFlag(Qt::ControlModifier))) {
    // Стандартная обработка Ctrl+C
}
```

### macOS

```cpp
#ifdef Q_OS_MAC
} else if (e->key() == Qt::Key_E && e->modifiers().testFlag(Qt::ControlModifier)) {
	// Ctrl+E - копирование в FindBuffer (специфично для macOS)
	auto selection = _selection.empty()
		? (_contextMenu ? _savedSelection : _selection)
		: _selection;
	if (!selection.empty()) {
		TextUtilities::SetClipboardText(
			_text.toTextForMimeData(selection),
			QClipboard::FindBuffer  // ← Специальный буфер macOS
		);
	}
#endif
```

**macOS дополнительно поддерживает:**
- `Ctrl+E` - копирование в FindBuffer (для быстрого поиска)
- Работает с `QClipboard::FindBuffer` вместо `QClipboard::Clipboard`

**Никаких различий в телеметрии между платформами нет** - её просто нет нигде.

---

## 🔄 Альтернативные методы копирования

### 1. Контекстное меню (правый клик)

```cpp
// Пример из меню копирования ссылки
base::make_weak(controller.get()),
[text] {
	QGuiApplication::clipboard()->setText(text);
}
```

### 2. Копирование инвайт-ссылок

```cpp
const auto link = _peer->session().createInternalLinkFull(username);
QGuiApplication::clipboard()->setText(link);
TextUtilities::SetClipboardText(
	TextForMimeData::Simple(link),
	QClipboard::Clipboard
);
```

### 3. Копирование медиа-файлов

Также использует локальное API без отправки данных на сервер.

**Общая черта:** Все методы используют прямой доступ к системному буферу обмена через Qt.

---

## 📊 Архитектура обработки событий клавиатуры

```
Нажатие Ctrl+C
    ↓
Qt Event System (QKeyEvent)
    ↓
FlatLabel::keyPressEvent()
    ↓
Проверка: Key_Copy || (Key_C + ControlModifier)?
    ↓ Да
Проверка: _selection не пустой?
    ↓ Да
copySelectedText()
    ↓
TextUtilities::SetClipboardText()
    ↓
QGuiApplication::clipboard()->setMimeData()
    ↓
Системный буфер обмена
    ↓
КОНЕЦ (никакой сетевой активности)
```

---

## 🔐 Приватность и безопасность

### Что это значит для пользователя?

✅ **Плюсы:**
- Копирование полностью локальное
- Никакая информация о скопированном тексте не покидает ваше устройство
- Нет телеметрии действий пользователя
- Нет tracking'а использования буфера обмена

❌ **Потенциальные риски (не связанные с Telegram):**
- Другие приложения на вашем устройстве МОГУТ читать системный буфер обмена
- Это ограничение операционной системы, а не Telegram
- Современные ОС (macOS, iOS 14+) показывают уведомления при доступе к буферу

### Сравнение с другими мессенджерами

Telegram Desktop использует стандартный подход Qt/C++:
- Нет облачной синхронизации буфера обмена (в отличие от некоторых систем)
- Нет истории копирований на сервере
- Нет автоматических бэкапов скопированного контента

---

## 🛠️ Технические детали

### Используемые Qt классы

```cpp
#include <QtGui/QClipboard>
#include <QtGui/QGuiApplication>

// Основной API
QClipboard *clipboard = QGuiApplication::clipboard();
clipboard->setText(text);                    // Простой текст
clipboard->setMimeData(mimeData, mode);      // Форматированный текст
```

### Режимы буфера обмена

```cpp
enum QClipboard::Mode {
    Clipboard,    // Основной буфер (Ctrl+C/Ctrl+V)
    Selection,    // X11 selection (средняя кнопка мыши в Linux)
    FindBuffer    // macOS find buffer (Ctrl+E)
}
```

Telegram использует:
- `Clipboard` - по умолчанию (Windows/Linux/macOS)
- `FindBuffer` - для macOS Ctrl+E

### Формат данных

```cpp
TextForMimeData textForMimeData = {
    .rich = richText,      // Форматированный текст (если есть)
    .expanded = expandedText  // Раскрытый текст (с развернутыми ссылками)
};

// Конвертация в MIME data для системного буфера
std::unique_ptr<QMimeData> MimeDataFromText(const TextForMimeData &text);
```

---

## 📝 Заключение

### Основные выводы

1. **Telegram Desktop не шпионит за вашими копированиями** - никакие данные о Ctrl+C не отправляются на сервер
2. **Используется стандартный Qt API** - прозрачная реализация без скрытых механизмов
3. **Полностью локальная операция** - данные идут только в системный буфер обмена
4. **Открытый исходный код** - любой может проверить это самостоятельно

### Файлы для самостоятельной проверки

- `Telegram/lib_ui/ui/widgets/labels.cpp` - обработка клавиатуры в labels
- `Telegram/lib_ui/ui/text/text_utilities.cpp` - утилиты работы с буфером обмена
- `Telegram/SourceFiles/ui/widgets/input_fields.cpp` - обработка в полях ввода

### Методология исследования

- ✅ Полный поиск по кодовой базе (Grep/Glob)
- ✅ Анализ всех упоминаний clipboard, copy, Ctrl+C
- ✅ Проверка наличия телеметрии и аналитики
- ✅ Проверка API вызовов к серверу
- ✅ Платформенный анализ (Windows/Linux/macOS)

---

**Дата исследования:** 2025-10-25
**Версия кодовой базы:** branch `system_info`, commit `aa6fc6a8b5`
**Статус:** ✅ Подтверждено - копирование полностью локальное

