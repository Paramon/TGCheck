# Полный механизм работы черновиков (Drafts) в Telegram Desktop

## 📋 Оглавление

1. [Краткий обзор](#краткий-обзор)
2. [Архитектура черновиков](#архитектура-черновиков)
3. [Процесс редактирования текста](#процесс-редактирования-текста)
4. [Typing Indicator (Индикатор "печатает")](#typing-indicator)
5. [Локальное сохранение черновиков](#локальное-сохранение-черновиков)
6. [Синхронизация с облаком](#синхронизация-с-облаком)
7. [Получение черновиков с сервера](#получение-черновиков-с-сервера)
8. [Streamed Drafts (Совместное редактирование)](#streamed-drafts)
9. [Временные интервалы и таймауты](#временные-интервалы)
10. [Диаграммы и потоки данных](#диаграммы)

---

## Краткий обзор

### Что такое черновик?

Черновик (Draft) — это несохраненное сообщение в поле ввода, которое:
- ✅ Сохраняется локально при изменении текста
- ✅ Синхронизируется с облаком через ~14 секунд после последнего изменения
- ✅ Синхронизируется между устройствами
- ✅ Может содержать текст, форматирование, ответ на сообщение, веб-превью
- ❌ **НЕ отправляет содержимое при typing indicator**

### Ключевые файлы

```
Структуры данных:
├── data/data_drafts.h                      - Классы Draft, DraftKey, WebPageDraft
└── data/data_drafts.cpp                    - Конструкторы и утилиты

Typing indicator:
├── api/api_send_progress.h                 - Enum SendProgressType
└── api/api_send_progress.cpp               - Логика отправки typing

UI и редактирование:
├── history/view/controls/
│   └── history_view_compose_controls.cpp   - Обработка ввода текста

История и хранение:
├── history/history.h                       - Управление черновиками в истории
├── history/history.cpp                     - Функции setDraft, applyCloudDraft
└── storage/storage_account.cpp             - Сохранение в локальную БД

API и синхронизация:
├── apiwrap.h                               - saveDraftToCloudDelayed()
├── apiwrap.cpp                             - saveDraftsToCloud()
└── api/api_updates.cpp                     - Обработка updateDraftMessage

Streamed Drafts:
├── history/history_streamed_drafts.h       - Интерфейс
└── history/history_streamed_drafts.cpp     - Реализация
```

---

## Архитектура черновиков

### Типы черновиков (DraftKey)

**Файл:** `data/data_drafts.h:75-230`

```cpp
class DraftKey {
public:
    // Основные типы
    static constexpr DraftKey Local(MsgId topicRootId, PeerId monoforumPeerId);
    static constexpr DraftKey LocalEdit(MsgId topicRootId, PeerId monoforumPeerId);
    static constexpr DraftKey Cloud(MsgId topicRootId, PeerId monoforumPeerId);
    static constexpr DraftKey Scheduled();
    static constexpr DraftKey ScheduledEdit();
    static constexpr DraftKey Shortcut(BusinessShortcutId shortcutId);
    static constexpr DraftKey ShortcutEdit(BusinessShortcutId shortcutId);
};
```

#### Типы черновиков:

| Тип | Описание | Когда используется |
|-----|----------|-------------------|
| **Local** | Локальный черновик нового сообщения | При печати нового сообщения |
| **LocalEdit** | Локальный черновик редактирования | При редактировании существующего сообщения |
| **Cloud** | Облачный черновик | Синхронизирован с сервером |
| **Scheduled** | Черновик запланированного сообщения | В разделе запланированных |
| **ScheduledEdit** | Редактирование запланированного | При редактировании запланированного |
| **Shortcut** | Черновик быстрого ответа | В бизнес-аккаунтах |

### Структура Draft

**Файл:** `data/data_drafts.h:50-73`

```cpp
struct Draft {
    TimeId date = 0;                    // Время создания/обновления
    TextWithTags textWithTags;          // Текст с форматированием
    FullReplyTo reply;                  // Ответ на сообщение (или ID редактируемого)
    SuggestPostOptions suggest;         // Опции предлагаемого поста
    MessageCursor cursor;               // Позиция курсора
    WebPageDraft webpage;               // Веб-превью
    mtpRequestId saveRequestId = 0;     // ID запроса сохранения на сервер
};
```

#### Поля Draft:

- **date**: Unix timestamp создания или последнего обновления
- **textWithTags**: Текст с тегами форматирования (жирный, курсив, упоминания и т.д.)
- **reply.messageId**:
  - Для обычного черновика: ID сообщения, на которое отвечаем
  - **Для edit-черновика: ID редактируемого сообщения**
- **cursor**: Позиция курсора и выделение текста
- **webpage**: Информация о веб-превью (URL, forced media size)
- **saveRequestId**: ID активного запроса на сохранение в облако

### WebPageDraft

**Файл:** `data/data_drafts.h:35-48`

```cpp
struct WebPageDraft {
    WebPageId id = 0;                   // ID веб-страницы
    QString url;                        // URL
    bool forceLargeMedia : 1 = false;   // Принудительно большой медиа
    bool forceSmallMedia : 1 = false;   // Принудительно маленький медиа
    bool invert : 1 = false;            // Инвертировать позицию
    bool manual : 1 = false;            // Вручную добавленный
    bool removed : 1 = false;           // Превью удалено
};
```

### Хранение черновиков в History

**Файл:** `history/history.h:297-299`

```cpp
Data::Draft *draft(Data::DraftKey key) const;
void setDraft(Data::DraftKey key, std::unique_ptr<Data::Draft> &&draft);
void clearDraft(Data::DraftKey key);
```

Черновики хранятся в `History` в виде `flat_map`:

```cpp
using HistoryDrafts = base::flat_map<DraftKey, std::unique_ptr<Draft>>;
```

---

## Процесс редактирования текста

### 1. Пользователь вводит текст

**Файл:** `history/view/controls/history_view_compose_controls.cpp:1865-1890`

```cpp
void ComposeControls::fieldChanged() {
    // Проверяем, нужно ли отправлять typing indicator
    const auto typing = (!_inlineBot
        && !_header->isEditingMessage()
        && (_textUpdateEvents & TextUpdateEvent::SendTyping));

    updateSendButtonType();
    _hasSendText = HasSendText(_field);

    if (updateBotCommandShown() || updateLikeShown()) {
        updateControlsVisibility();
        updateControlsGeometry(_wrap->size());
    }

    // Отложенная отправка typing и обновление inline-бота
    InvokeQueued(_field.get(), [=] {
        updateInlineBotQuery();
        if ((!_autocomplete || !_autocomplete->stickersEmoji()) && typing) {
            _sendActionUpdates.fire({ Api::SendProgressType::Typing });  // ← TYPING!
        }
    });

    checkCharsLimitation();

    _saveCloudDraftTimer.cancel();
    if (!(_textUpdateEvents & TextUpdateEvent::SaveDraft)) {
        return;
    }
    _saveDraftText = true;
    saveDraft(true);  // ← Локальное сохранение
}
```

#### Условия для typing indicator:

1. ✅ **НЕ** используется inline-бот (например, `@gif`)
2. ✅ **НЕ** редактируется существующее сообщение
3. ✅ Флаг `TextUpdateEvent::SendTyping` активен
4. ✅ **НЕ** открыта панель выбора стикеров/эмодзи через autocomplete

### 2. Обработка изменений

```
Пользователь печатает
         ↓
fieldChanged() вызывается
         ↓
    ┌────┴────┐
    ↓         ↓
 Typing    Локальное
отправка   сохранение
```

---

## Typing Indicator

> Подробный анализ: `FuckTG/typing_indicator_analysis.md`

### Что отправляется?

**Файл:** `api/api_send_progress.cpp:111-152`

```cpp
void SendProgressManager::send(const Key &key, int progress) {
    if (skipRequest(key)) {
        return;
    }

    // Создаем MTP action БЕЗ текста
    const auto action = [&]() -> MTPsendMessageAction {
        switch (key.type) {
        case Type::Typing: return MTP_sendMessageTypingAction();
        case Type::RecordVideo: return MTP_sendMessageRecordVideoAction();
        // ... другие типы
        }
    }();

    // Отправляем messages.setTyping
    const auto requestId = _session->api().request(MTPmessages_SetTyping(
        MTP_flags(key.topMsgId
            ? MTPmessages_SetTyping::Flag::f_top_msg_id
            : MTPmessages_SetTyping::Flag(0)),
        key.history->peer->input,      // ← Кому
        MTP_int(key.topMsgId),          // ← ID темы/треда
        action                          // ← Тип действия (typing)
    )).done([=](const MTPBool &result, mtpRequestId requestId) {
        done(requestId);
    }).send();
}
```

### MTP метод: messages.setTyping

**Схема:** `mtproto/scheme/api.tl:2252`

```
messages.setTyping#58943ee2
    flags:#
    peer:InputPeer               ← Собеседник/чат
    top_msg_id:flags.0?int      ← ID темы (для форумов)
    action:SendMessageAction     ← Тип действия
= Bool;
```

**Пример запроса:**

```
messages.setTyping
    flags: 0
    peer: inputPeerUser {
        user_id: 123456789
        access_hash: 0x...
    }
    top_msg_id: 0
    action: sendMessageTypingAction
```

### Интервалы отправки

**Файл:** `api/api_send_progress.cpp:21-24`

```cpp
constexpr auto kCancelTypingActionTimeout = crl::time(5000);   // 5 сек
constexpr auto kSendMySpeakingInterval = 3 * crl::time(1000);  // 3 сек (для звонков)
constexpr auto kSendMyTypingInterval = 5 * crl::time(1000);    // 5 сек (для typing)
constexpr auto kSendTypingsToOfflineFor = TimeId(30);          // 30 сек
```

#### Логика интервалов:

**Файл:** `api/api_send_progress.cpp:85-108`

```cpp
bool SendProgressManager::updated(const Key &key, bool doing) {
    const auto now = crl::now();
    const auto i = _updated.find(key);
    if (doing) {
        const auto sendEach = (key.type == SendProgressType::Speaking)
            ? kSendMySpeakingInterval  // 3 сек для Speaking
            : kSendMyTypingInterval;    // 5 сек для Typing
        if (i == end(_updated)) {
            _updated.emplace(key, now + 2 * sendEach);  // Первая отправка сразу
        } else if (i->second > now + sendEach) {
            return false;  // Еще рано, пропускаем
        } else {
            i->second = now + 2 * sendEach;  // Обновляем время
        }
    } else {
        // Отмена typing
        if (i == end(_updated)) {
            return false;
        } else if (i->second <= now) {
            return false;
        } else {
            _updated.erase(i);
        }
    }
    return true;
}
```

### Когда typing НЕ отправляется

**Файл:** `api/api_send_progress.cpp:154-171`

```cpp
bool SendProgressManager::skipRequest(const Key &key) const {
    const auto user = key.history->peer->asUser();
    if (!user) {
        return false;  // Для групп/каналов отправляем
    } else if (user->isSelf()) {
        return true;   // Себе не отправляем
    } else if (user->isBot() && !user->isSupport()) {
        return true;   // Ботам не отправляем (кроме support)
    }

    // Проверяем, когда пользователь был онлайн
    const auto recently = base::unixtime::now() - kSendTypingsToOfflineFor;
    const auto lastseen = user->lastseen();
    if (lastseen.isRecently()) {
        return false;  // Недавно был онлайн - отправляем
    } else if (const auto value = lastseen.onlineTill()) {
        return (value < recently);  // Был онлайн > 30 сек назад - не отправляем
    }
    return true;  // Не знаем статус - не отправляем
}
```

#### Условия пропуска typing:

1. ❌ Отправка себе (`Saved Messages`)
2. ❌ Отправка ботам (кроме `@support`)
3. ❌ Пользователь был оффлайн > 30 секунд
4. ❌ Пользователь в каналах (не мегагруппах)

### Типы SendProgressType

**Файл:** `api/api_send_progress.h:21-36`

```cpp
enum class SendProgressType {
    Typing,              // Печатает текст
    RecordVideo,         // Записывает видео
    UploadVideo,         // Загружает видео (с прогрессом)
    RecordVoice,         // Записывает голосовое
    UploadVoice,         // Загружает голосовое (с прогрессом)
    RecordRound,         // Записывает видеосообщение
    UploadRound,         // Загружает видеосообщение (с прогрессом)
    UploadPhoto,         // Загружает фото (с прогрессом)
    UploadFile,          // Загружает файл (с прогрессом)
    ChooseLocation,      // Выбирает местоположение
    ChooseContact,       // Выбирает контакт
    ChooseSticker,       // Выбирает стикер
    PlayGame,            // Играет в игру
    Speaking,            // Говорит в групповом звонке
};
```

---

## Локальное сохранение черновиков

### Таймауты сохранения

**Файл:** `history/view/controls/history_view_compose_controls.cpp:99-101`

```cpp
constexpr auto kSaveDraftTimeout = crl::time(1000);           // 1 сек
constexpr auto kSaveDraftAnywayTimeout = 5 * crl::time(1000); // 5 сек
constexpr auto kSaveCloudDraftIdleTimeout = 14 * crl::time(1000); // 14 сек
```

### Логика сохранения

**Файл:** `history/view/controls/history_view_compose_controls.cpp:1932-1943`

```cpp
void ComposeControls::saveDraft(bool delayed) {
    if (delayed) {
        const auto now = crl::now();
        if (!_saveDraftStart) {
            _saveDraftStart = now;
            return _saveDraftTimer.callOnce(kSaveDraftTimeout);  // Ждем 1 сек
        } else if (now - _saveDraftStart < kSaveDraftAnywayTimeout) {
            return _saveDraftTimer.callOnce(kSaveDraftTimeout);  // Ждем еще 1 сек
        }
        // Прошло > 5 сек - сохраняем принудительно
    }
    writeDrafts();  // Сохраняем немедленно
}
```

#### Алгоритм:

```
Пользователь печатает
         ↓
   fieldChanged()
         ↓
   saveDraft(delayed=true)
         ↓
    _saveDraftStart = now
         ↓
   Таймер 1 сек
         ↓
   ┌─────┴─────┐
   ↓           ↓
Печатает?   Нет
   ↓           ↓
Сброс      Сохранить
таймера    локально
   ↓
Прошло > 5 сек?
   ↓
  Да → Сохранить принудительно
```

### writeDrafts()

**Файл:** `history/view/controls/history_view_compose_controls.cpp:2000-2017`

```cpp
void ComposeControls::writeDrafts() {
    const auto save = (_history != nullptr)
        && (_saveDraftStart > 0)
        && (draftKeyCurrent() != Data::DraftKey::None());
    _saveDraftStart = 0;
    _saveDraftTimer.cancel();

    if (save) {
        if (_saveDraftText) {
            writeDraftTexts();  // Сохраняем текст
        }
        writeDraftCursors();    // Сохраняем позицию курсора
    }
    _saveDraftText = false;

    // Запускаем таймер облачного сохранения
    if (!isEditingMessage() && !_inlineBot) {
        _saveCloudDraftTimer.callOnce(kSaveCloudDraftIdleTimeout);  // 14 сек
    }
}
```

### Сохранение в локальную БД

**Файл:** `storage/storage_account.cpp:1262-1307`

```cpp
void Account::writeDrafts(not_null<History*> history) {
    const auto peerId = history->peer->id;
    const auto &map = history->draftsMap();
    const auto supportMode = history->session().supportMode();
    const auto sourcesIt = _draftSources.find(history);
    const auto &sources = (sourcesIt != _draftSources.end())
        ? sourcesIt->second
        : EmptyMessageDraftSources();

    auto count = 0;
    EnumerateDrafts(
        map,
        sources,
        supportMode,
        [&](Data::DraftKey key, const Storage::MessageDraft &draft) {
            // Сериализуем и сохраняем черновик
            writeDraft(peerId, key, draft);
            ++count;
        });

    if (!count) {
        clearDrafts(peerId);  // Удаляем все черновики, если их нет
    }
}
```

### Структура MessageDraft для хранения

**Файл:** `storage/storage_account.h`

```cpp
struct MessageDraft {
    FullReplyTo reply;              // Ответ/редактирование
    SuggestPostOptions suggest;     // Опции поста
    TextWithTags textWithTags;      // Текст с тегами
    Data::WebPageDraft webpage;     // Веб-превью
};

struct MessageDraftSource {
    Fn<MessageDraft()> draft;       // Функция для получения черновика
    Fn<MessageCursor()> cursor;     // Функция для получения курсора
};
```

---

## Синхронизация с облаком

### Запуск синхронизации

**Файл:** `apiwrap.cpp:1938-1943`

```cpp
void ApiWrap::saveDraftToCloudDelayed(not_null<Data::Thread*> thread) {
    _draftsSaveRequestIds.emplace(base::make_weak(thread), 0);
    if (!_draftsSaveTimer.isActive()) {
        _draftsSaveTimer.callOnce(kSaveCloudDraftTimeout);  // 1 сек
    }
}
```

**Таймаут:** `kSaveCloudDraftTimeout = 1000` мс (1 секунда)

### Отправка на сервер

**Файл:** `apiwrap.cpp:2138-2251`

```cpp
void ApiWrap::saveDraftsToCloud() {
    for (auto i = begin(_draftsSaveRequestIds); i != end(_draftsSaveRequestIds);) {
        const auto weak = i->first;
        const auto thread = weak.get();
        if (!thread) {
            i = _draftsSaveRequestIds.erase(i);
            continue;
        } else if (i->second) {
            ++i;
            continue; // sent already
        }

        const auto history = thread->owningHistory();
        const auto topicRootId = thread->topicRootId();
        const auto monoforumPeerId = thread->monoforumPeerId();

        // Получаем локальный черновик
        auto localDraft = history->localDraft(topicRootId, monoforumPeerId);

        // Создаем или обновляем облачный черновик
        auto cloudDraft = history->createCloudDraft(
            topicRootId,
            monoforumPeerId,
            localDraft);

        // Формируем флаги
        auto flags = MTPmessages_SaveDraft::Flags(0);
        auto &textWithTags = cloudDraft->textWithTags;

        if (cloudDraft->webpage.removed) {
            flags |= MTPmessages_SaveDraft::Flag::f_no_webpage;
        } else if (!cloudDraft->webpage.url.isEmpty()) {
            flags |= MTPmessages_SaveDraft::Flag::f_media;
        }
        if (cloudDraft->reply.messageId
            || cloudDraft->reply.topicRootId
            || cloudDraft->reply.monoforumPeerId) {
            flags |= MTPmessages_SaveDraft::Flag::f_reply_to;
        }
        if (!textWithTags.tags.isEmpty()) {
            flags |= MTPmessages_SaveDraft::Flag::f_entities;
        }

        // Конвертируем теги в entities
        auto entities = Api::EntitiesToMTP(
            _session,
            TextUtilities::ConvertTextTagsToEntities(textWithTags.tags),
            Api::ConvertOption::SkipLocal);

        history->startSavingCloudDraft(topicRootId, monoforumPeerId);

        // ОТПРАВЛЯЕМ НА СЕРВЕР
        cloudDraft->saveRequestId = request(MTPmessages_SaveDraft(
            MTP_flags(flags),
            ReplyToForMTP(history, cloudDraft->reply),  // reply_to
            history->peer->input,                       // peer
            MTP_string(textWithTags.text),              // message (текст)
            entities,                                    // entities (форматирование)
            Data::WebPageForMTP(                        // media (веб-превью)
                cloudDraft->webpage,
                textWithTags.text.isEmpty()),
            MTP_long(0),                                // effect
            Api::SuggestToMTP(cloudDraft->suggest)      // suggested_post
        )).done([=](const MTPBool &result, const MTP::Response &response) {
            // Успешно сохранено
            history->finishSavingCloudDraft(
                topicRootId,
                monoforumPeerId,
                Api::UnixtimeFromMsgId(response.outerMsgId));
            // ...
        }).fail([=](const MTP::Error &error, const MTP::Response &response) {
            // Ошибка сохранения
            history->clearCloudDraft(topicRootId, monoforumPeerId);
            // ...
        }).send();

        i->second = cloudDraft->saveRequestId;
        ++i;
    }
}
```

### MTP метод: messages.saveDraft

**Схема:** `mtproto/scheme/api.tl:2305`

```
messages.saveDraft#54ae308e
    flags:#
    no_webpage:flags.1?true          ← Отключить веб-превью
    invert_media:flags.6?true        ← Инвертировать медиа
    reply_to:flags.4?InputReplyTo    ← Ответ на сообщение
    peer:InputPeer                   ← Кому
    message:string                   ← Текст черновика
    entities:flags.3?Vector<MessageEntity>  ← Форматирование
    media:flags.5?InputMedia         ← Медиа (веб-превью)
    effect:flags.7?long              ← Эффект
    suggested_post:flags.8?SuggestedPost    ← Предложенный пост
= Bool;
```

**Пример запроса:**

```
messages.saveDraft
    flags: 8 (entities present)
    peer: inputPeerUser {
        user_id: 123456789
        access_hash: 0x...
    }
    message: "Привет, как дела?"
    entities: [
        messageEntityBold {
            offset: 0
            length: 6
        }
    ]
```

### Что отправляется в облако?

✅ **Отправляется:**
- Текст сообщения
- Форматирование (жирный, курсив, ссылки, упоминания)
- ID сообщения, на которое отвечаем
- ID темы/топика (для форумов)
- Веб-превью (URL и настройки отображения)
- Опции предлагаемого поста (для каналов)

❌ **НЕ отправляется:**
- Позиция курсора (только локально)
- История редактирования
- Черновики редактирования (edit drafts)

### Предотвращение конфликтов

**Файл:** `history/history.cpp:376-406`

```cpp
bool History::skipCloudDraftUpdate(
        MsgId topicRootId,
        PeerId monoforumPeerId,
        TimeId date) const {
    const auto key = Data::DraftKey::Local(topicRootId, monoforumPeerId);
    const auto i = _acceptCloudDraftsAfter.find(key);
    return _savingCloudDraftRequests.contains(key)  // Сейчас сохраняем
        || (i != _acceptCloudDraftsAfter.end() && date < i->second);  // Старый
}

void History::startSavingCloudDraft(
        MsgId topicRootId,
        PeerId monoforumPeerId) {
    const auto key = Data::DraftKey::Local(topicRootId, monoforumPeerId);
    ++_savingCloudDraftRequests[key];  // Увеличиваем счетчик запросов
}

void History::finishSavingCloudDraft(
        MsgId topicRootId,
        PeerId monoforumPeerId,
        TimeId savedAt) {
    const auto key = Data::DraftKey::Local(topicRootId, monoforumPeerId);
    const auto i = _savingCloudDraftRequests.find(key);
    if (i != _savingCloudDraftRequests.end()) {
        if (--i->second <= 0) {
            _savingCloudDraftRequests.erase(i);
        }
    }
    // Не принимаем черновики старше savedAt + 2 сек
    auto &after = _acceptCloudDraftsAfter[key];
    after = std::max(after, savedAt + kSkipCloudDraftsFor);  // +2 сек
}
```

**Константа:** `kSkipCloudDraftsFor = TimeId(2)` (2 секунды)

#### Логика:

1. Перед отправкой на сервер вызывается `startSavingCloudDraft()`
2. Во время отправки `skipCloudDraftUpdate()` вернет `true`
3. После получения ответа вызывается `finishSavingCloudDraft()`
4. Черновики с `date < savedAt + 2 сек` игнорируются

---

## Получение черновиков с сервера

### Update от сервера

**Схема:** `mtproto/scheme/api.tl:344`

```
updateDraftMessage#edfc111e
    flags:#
    peer:Peer                        ← От кого
    top_msg_id:flags.0?int          ← ID темы/топика
    saved_peer_id:flags.1?Peer      ← ID сохраненного чата
    draft:DraftMessage               ← Черновик
= Update;
```

### Обработка update

**Файл:** `api/api_updates.cpp:2710-2726`

```cpp
case mtpc_updateDraftMessage: {
    const auto &data = update.c_updateDraftMessage();
    const auto peerId = peerFromMTP(data.vpeer());
    const auto topicRootId = data.vtop_msg_id().value_or_empty();
    const auto monoforumPeerId = data.vsaved_peer_id()
        ? peerFromMTP(*data.vsaved_peer_id())
        : PeerId();

    data.vdraft().match([&](const MTPDdraftMessage &data) {
        Data::ApplyPeerCloudDraft(&session(), peerId, topicRootId, monoforumPeerId, data);
    }, [&](const MTPDdraftMessageEmpty &data) {
        Data::ClearPeerCloudDraft(&session(), peerId, topicRootId, monoforumPeerId, data.vdate().value_or_empty());
    });
} break;
```

### ApplyPeerCloudDraft

**Файл:** `data/data_drafts.cpp:73-135`

```cpp
void ApplyPeerCloudDraft(
        not_null<Main::Session*> session,
        PeerId peerId,
        MsgId topicRootId,
        PeerId monoforumPeerId,
        const MTPDdraftMessage &draft) {
    const auto history = session->data().history(peerId);
    const auto date = draft.vdate().v;

    // Проверяем, не игнорируем ли мы этот update
    if (history->skipCloudDraftUpdate(topicRootId, monoforumPeerId, date)) {
        return;
    }

    // Парсим текст с entities
    const auto textWithTags = TextWithTags{
        qs(draft.vmessage()),
        TextUtilities::ConvertEntitiesToTextTags(
            Api::EntitiesFromMTP(
                session,
                draft.ventities().value_or_empty()))
    };

    // Парсим reply
    auto replyTo = draft.vreply_to()
        ? ReplyToFromMTP(history, *draft.vreply_to())
        : FullReplyTo();
    replyTo.topicRootId = topicRootId;
    replyTo.monoforumPeerId = monoforumPeerId;

    // Парсим webpage
    auto webpage = WebPageDraft{
        .invert = draft.is_invert_media(),
        .removed = draft.is_no_webpage(),
    };
    if (const auto media = draft.vmedia()) {
        media->match([&](const MTPDmessageMediaWebPage &data) {
            const auto parsed = session->data().processWebpage(data.vwebpage());
            if (!parsed->failed) {
                webpage.forceLargeMedia = data.is_force_large_media();
                webpage.forceSmallMedia = data.is_force_small_media();
                webpage.manual = data.is_manual();
                webpage.url = parsed->url;
                webpage.id = parsed->id;
            }
        }, [](const auto &) {});
    }

    // Создаем облачный черновик
    auto cloudDraft = std::make_unique<Draft>(
        textWithTags,
        replyTo,
        suggest,
        MessageCursor(Ui::kQFixedMax, Ui::kQFixedMax, Ui::kQFixedMax),
        std::move(webpage));
    cloudDraft->date = date;

    history->setCloudDraft(std::move(cloudDraft));
    history->applyCloudDraft(topicRootId, monoforumPeerId);
}
```

### applyCloudDraft

**Файл:** `history/history.cpp:408-431`

```cpp
void History::applyCloudDraft(MsgId topicRootId, PeerId monoforumPeerId) {
    if (!topicRootId && session().supportMode()) {
        updateChatListEntry();
        session().supportHelper().cloudDraftChanged(this);
    } else {
        createLocalDraftFromCloud(topicRootId, monoforumPeerId);  // Копируем в local
        if (const auto thread = threadFor(topicRootId, monoforumPeerId)) {
            thread->updateChatListSortPosition();
            if (topicRootId) {
                session().changes().topicUpdated(
                    thread->asTopic(),
                    Data::TopicUpdate::Flag::CloudDraft);
            } else if (monoforumPeerId) {
                session().changes().sublistUpdated(
                    thread->asSublist(),
                    Data::SublistUpdate::Flag::CloudDraft);
            } else {
                session().changes().historyUpdated(
                    this,
                    UpdateFlag::CloudDraft);
            }
        }
    }
}
```

### createLocalDraftFromCloud

**Файл:** `history/history.cpp:232-269`

```cpp
void History::createLocalDraftFromCloud(
        MsgId topicRootId,
        PeerId monoforumPeerId) {
    const auto draft = cloudDraft(topicRootId, monoforumPeerId);
    if (!draft) {
        clearLocalDraft(topicRootId, monoforumPeerId);
        return;
    } else if (Data::DraftIsNull(draft) || !draft->date) {
        return;
    }

    draft->reply.topicRootId = topicRootId;
    draft->reply.monoforumPeerId = monoforumPeerId;

    auto existing = localDraft(topicRootId, monoforumPeerId);

    // Применяем облачный черновик только если он новее локального
    if (Data::DraftIsNull(existing)
        || !existing->date
        || draft->date >= existing->date) {
        if (!existing) {
            setLocalDraft(std::make_unique<Data::Draft>(
                draft->textWithTags,
                draft->reply,
                draft->suggest,
                draft->cursor,
                draft->webpage));
            existing = localDraft(topicRootId, monoforumPeerId);
        } else if (existing != draft) {
            existing->textWithTags = draft->textWithTags;
            existing->reply = draft->reply;
            existing->suggest = draft->suggest;
            existing->cursor = draft->cursor;
            existing->webpage = draft->webpage;
        }
        existing->date = draft->date;
    }
}
```

#### Логика применения:

```
Получен updateDraftMessage
         ↓
   ApplyPeerCloudDraft()
         ↓
   Проверка skipCloudDraftUpdate()
         ↓
    ┌────┴────┐
    ↓         ↓
Пропускаем  Применяем
(сохраняем  (парсим draft)
сейчас)        ↓
          setCloudDraft()
               ↓
          applyCloudDraft()
               ↓
     createLocalDraftFromCloud()
               ↓
          Сравниваем даты
               ↓
        ┌──────┴──────┐
        ↓             ↓
   Cloud новее   Local новее
        ↓             ↓
  Обновляем     Игнорируем
   Local
```

### Запрос всех черновиков

**MTP метод:** `messages.getAllDrafts#6a3f8d65 = Updates;`

Вызывается при старте приложения для загрузки всех черновиков.

---

## Streamed Drafts

> **Новая фича:** Показывает, что печатает другой пользователь в реальном времени

### Что это?

**Streamed Drafts** — это механизм отображения черновика (draft) другого пользователя в чате в реальном времени. Используется для совместного редактирования или просмотра того, что печатает собеседник.

### MTP Action: sendMessageTextDraftAction

**Схема:** `mtproto/scheme/api.tl:541`

```
sendMessageTextDraftAction#376d975c
    random_id:long          ← ID черновика для обновления
    text:TextWithEntities   ← ТЕКСТ ЧЕРНОВИКА (!)
= SendMessageAction;
```

**Важно:** В отличие от обычного `typing`, здесь **отправляется сам текст черновика**!

### Обработка входящего streamed draft

**Файл:** `api/api_updates.cpp:1115-1118`

```cpp
} else if (action.type() == mtpc_sendMessageTextDraftAction) {
    const auto &data = action.c_sendMessageTextDraftAction();
    history->streamedDrafts().apply(rootId, fromId, when, data);
    return;
}
```

### HistoryStreamedDrafts::apply

**Файл:** `history/history_streamed_drafts.cpp:33-70`

```cpp
void HistoryStreamedDrafts::apply(
        MsgId rootId,
        PeerId fromId,
        TimeId when,
        const MTPDsendMessageTextDraftAction &data) {
    if (!rootId) {
        rootId = Data::ForumTopic::kGeneralId;
    }
    if (!when) {
        clear(rootId);  // Отмена черновика
        return;
    }

    // Парсим текст с форматированием
    const auto text = Api::ParseTextWithEntities(
        &_history->session(),
        data.vtext());
    const auto randomId = data.vrandom_id().v;

    // Пытаемся обновить существующий черновик
    if (update(rootId, randomId, text)) {
        return;
    }

    // Создаем новое локальное сообщение для отображения черновика
    clear(rootId);
    _drafts.emplace(rootId, Draft{
        .message = _history->addNewLocalMessage({
            .id = _history->owner().nextLocalMessageId(),
            .flags = MessageFlag::Local | MessageFlag::HasReplyInfo,
            .from = fromId,
            .replyTo = {
                .messageId = { _history->peer->id, rootId },
                .topicRootId = rootId,
            },
            .date = when,
        }, text, MTP_messageMediaEmpty()),
        .randomId = randomId,
        .updated = crl::now(),
    });

    // Запускаем таймер очистки через 30 сек
    if (!_checkTimer.isActive()) {
        _checkTimer.callOnce(kClearTimeout);  // 30 сек
    }
}
```

### Обновление существующего streamed draft

**Файл:** `history/history_streamed_drafts.cpp:72-83`

```cpp
bool HistoryStreamedDrafts::update(
        MsgId rootId,
        uint64 randomId,
        const TextWithEntities &text) {
    const auto i = _drafts.find(rootId);
    if (i == end(_drafts) || i->second.randomId != randomId) {
        return false;  // Другой черновик или нет черновика
    }
    i->second.message->setText(text);  // Обновляем текст
    i->second.updated = crl::now();     // Обновляем время
    return true;
}
```

### Очистка старых streamed drafts

**Файл:** `history/history_streamed_drafts.cpp:105-124`

```cpp
void HistoryStreamedDrafts::check() {
    auto closest = crl::time();
    const auto now = crl::now();
    for (auto i = begin(_drafts); i != end(_drafts);) {
        if (now - i->second.updated >= kClearTimeout) {  // 30 сек
            i->second.message->destroy();
            i = _drafts.erase(i);
        } else {
            if (!closest || closest > i->second.updated) {
                closest = i->second.updated;
            }
            ++i;
        }
    }
    if (closest) {
        _checkTimer.callOnce(kClearTimeout - (now - closest));
    } else {
        scheduleDestroy();  // Все черновики удалены
    }
}
```

**Таймаут:** `kClearTimeout = 30 * crl::time(1000)` (30 секунд)

### Отображение в UI

Streamed draft отображается как обычное локальное сообщение в истории чата. Оно:
- Имеет флаг `MessageFlag::Local`
- Не сохраняется в базу данных
- Автоматически удаляется через 30 секунд или при отправке реального сообщения

### Удаление при отправке сообщения

**Файл:** `history/history_streamed_drafts.cpp:96-103`

```cpp
void HistoryStreamedDrafts::applyItemAdded(not_null<HistoryItem*> item) {
    const auto rootId = item->topicRootId();
    const auto i = _drafts.find(rootId);
    if (i == end(_drafts) || i->second.message->from() != item->from()) {
        return;
    }
    clear(rootId);  // Удаляем streamed draft
}
```

---

## Временные интервалы

### Сводная таблица таймаутов

| Действие | Интервал | Файл | Константа |
|----------|----------|------|-----------|
| **Отправка typing** | Каждые **5 сек** | `api/api_send_progress.cpp:23` | `kSendMyTypingInterval` |
| **Отмена typing** | Через **5 сек** | `api/api_send_progress.cpp:21` | `kCancelTypingActionTimeout` |
| **Локальное сохранение (отложенное)** | **1 сек** | `history/view/controls/history_view_compose_controls.cpp:99` | `kSaveDraftTimeout` |
| **Локальное сохранение (принудительное)** | **5 сек** | `history/view/controls/history_view_compose_controls.cpp:100` | `kSaveDraftAnywayTimeout` |
| **Запуск облачной синхронизации** | **14 сек** после последнего изменения | `history/view/controls/history_view_compose_controls.cpp:101` | `kSaveCloudDraftIdleTimeout` |
| **Отправка на сервер (задержка)** | **1 сек** | `apiwrap.cpp:95` | `kSaveCloudDraftTimeout` |
| **Игнорирование старых cloud drafts** | **2 сек** | `history/history.cpp:80` | `kSkipCloudDraftsFor` |
| **Typing для оффлайн пользователей** | Не отправлять если > **30 сек** | `api/api_send_progress.cpp:24` | `kSendTypingsToOfflineFor` |
| **Очистка streamed drafts** | **30 сек** | `history/history_streamed_drafts.cpp:18` | `kClearTimeout` |

### Визуализация временных интервалов

```
Время →
0s    1s    2s    3s    4s    5s    6s    7s    8s    9s    10s   11s   12s   13s   14s   15s

Печать: ┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼
        ↓     ↓     ↓     ↓     ↓                                                       ↓
Typing: ●─────●─────●─────●─────●  (каждые 5 сек)                                      ⊗ отмена

Local:  ●───→●     ●───→●     ●───→●     (отложенное 1 сек, принудительное 5 сек)

Cloud:  ●────────────────────────────────────────────────────────────────────────────→●
        (ждем 14 сек после последнего изменения)
```

---

## Диаграммы

### 1. Полный жизненный цикл черновика

```
┌─────────────────┐
│ Пользователь    │
│ печатает текст  │
└────────┬────────┘
         ↓
┌────────────────────────────────────────────────┐
│ ComposeControls::fieldChanged()                │
│ - Обновляет UI                                 │
│ - Отправляет typing (если условия выполнены)  │
│ - Запускает saveDraft(delayed=true)           │
└────────┬───────────────────────────────────────┘
         ↓
    ┌────┴────┐
    ↓         ↓
┌──────────┐  ┌──────────────────────────────────┐
│ Typing   │  │ saveDraft(delayed=true)          │
│ каждые   │  │ - Ждет 1 сек                     │
│ 5 сек    │  │ - Или 5 сек принудительно        │
└──────────┘  └─────────┬────────────────────────┘
                        ↓
              ┌─────────────────────┐
              │ writeDrafts()       │
              │ - Сохранить текст   │
              │ - Сохранить курсор  │
              └─────────┬───────────┘
                        ↓
              ┌─────────────────────────────────┐
              │ storage::Account::writeDrafts() │
              │ - Записать в локальную БД       │
              └─────────┬───────────────────────┘
                        ↓
              ┌───────────────────────────────────┐
              │ _saveCloudDraftTimer.callOnce(14s)│
              │ - Запланировать облачное          │
              └─────────┬─────────────────────────┘
                        ↓ (через 14 сек)
              ┌─────────────────────────────┐
              │ ApiWrap::saveCurrentDraft   │
              │ ToCloud()                   │
              │ - Найти активный чат        │
              └─────────┬───────────────────┘
                        ↓
              ┌─────────────────────────────┐
              │ ApiWrap::saveDraftToCloud   │
              │ Delayed()                   │
              │ - Добавить в очередь        │
              └─────────┬───────────────────┘
                        ↓ (через 1 сек)
              ┌─────────────────────────────┐
              │ ApiWrap::saveDraftsToCloud()│
              │ - Создать cloudDraft        │
              │ - Отправить MTP запрос      │
              └─────────┬───────────────────┘
                        ↓
              ┌─────────────────────────────┐
              │ messages.saveDraft          │
              │   peer: ...                 │
              │   message: "текст"          │
              │   entities: [...]           │
              └─────────┬───────────────────┘
                        ↓
              ┌─────────────────────────────┐
              │ Сервер Telegram             │
              │ - Сохраняет черновик        │
              │ - Отправляет другим         │
              │   устройствам               │
              └─────────────────────────────┘
```

### 2. Получение черновика с сервера

```
┌─────────────────────┐
│ Сервер Telegram     │
│ отправляет update   │
└──────────┬──────────┘
           ↓
┌──────────────────────────────────┐
│ updateDraftMessage               │
│   peer: ...                      │
│   draft:                         │
│     message: "текст"             │
│     entities: [...]              │
│     date: 1234567890             │
└──────────┬───────────────────────┘
           ↓
┌──────────────────────────────────┐
│ Api::Updates::applyUpdates()     │
│ - Обрабатывает update            │
└──────────┬───────────────────────┘
           ↓
┌──────────────────────────────────┐
│ Data::ApplyPeerCloudDraft()      │
│ - Парсит текст, entities         │
│ - Парсит reply, webpage          │
│ - Создает cloudDraft             │
└──────────┬───────────────────────┘
           ↓
┌──────────────────────────────────┐
│ history->skipCloudDraftUpdate()? │
└──────────┬───────────────────────┘
           ↓
    ┌──────┴──────┐
    ↓             ↓
┌─────────┐  ┌────────────────────────┐
│ Да      │  │ Нет                    │
│ (игнор) │  └───────────┬────────────┘
└─────────┘              ↓
              ┌──────────────────────────┐
              │ history->setCloudDraft() │
              │ - Сохранить облачный     │
              └───────────┬──────────────┘
                          ↓
              ┌──────────────────────────┐
              │ history->applyCloudDraft()│
              │ - Применить к UI         │
              └───────────┬──────────────┘
                          ↓
              ┌────────────────────────────┐
              │ createLocalDraftFromCloud()│
              │ - Копировать в localDraft  │
              │ - Сравнить даты            │
              └───────────┬────────────────┘
                          ↓
              ┌────────────────────────────┐
              │ UI обновлен                │
              │ Черновик применен          │
              └────────────────────────────┘
```

### 3. Streamed Draft (Совместное редактирование)

```
┌─────────────────────┐
│ Пользователь A      │
│ печатает "Привет"   │
└──────────┬──────────┘
           ↓
┌──────────────────────────────────┐
│ messages.setTyping               │
│   action: sendMessageTextDraft   │
│     Action {                     │
│       random_id: 12345           │
│       text: "Привет"             │
│     }                            │
└──────────┬───────────────────────┘
           ↓
┌──────────────────────────────────┐
│ Сервер Telegram                  │
│ - Пересылает другим участникам   │
└──────────┬───────────────────────┘
           ↓
┌──────────────────────────────────┐
│ Пользователь B получает update   │
│ updateUserTyping:                │
│   action: sendMessageTextDraft   │
│     Action                       │
└──────────┬───────────────────────┘
           ↓
┌──────────────────────────────────┐
│ Api::Updates::handleSendAction   │
│ Update()                         │
│ - Проверяет тип action           │
└──────────┬───────────────────────┘
           ↓
┌──────────────────────────────────┐
│ history->streamedDrafts().apply()│
│ - random_id: 12345               │
│ - text: "Привет"                 │
└──────────┬───────────────────────┘
           ↓
    ┌──────┴──────┐
    ↓             ↓
┌─────────────┐  ┌─────────────────────────┐
│ Существует? │  │ Нет                     │
│ Да          │  │ - Создать локальное     │
│ - Обновить  │  │   сообщение             │
│   текст     │  │ - Отобразить в чате     │
└─────────────┘  └─────────┬───────────────┘
                           ↓
              ┌────────────────────────────┐
              │ Таймер 30 сек              │
              │ - Автоудаление черновика   │
              └────────────────────────────┘
```

### 4. Процесс отправки typing

```
┌─────────────────────────────────┐
│ _sendActionUpdates.fire(        │
│   { Api::SendProgressType::     │
│     Typing })                   │
└────────────┬────────────────────┘
             ↓
┌─────────────────────────────────┐
│ SendProgressManager::update()   │
│ - Проверка интервала (5 сек)    │
└────────────┬────────────────────┘
             ↓
        ┌────┴────┐
        ↓         ↓
┌─────────────┐  ┌──────────────────────┐
│ Слишком     │  │ Прошло >= 5 сек      │
│ рано        │  │                      │
│ (skip)      │  └───────────┬──────────┘
└─────────────┘              ↓
              ┌──────────────────────────┐
              │ SendProgressManager::    │
              │ send()                   │
              └───────────┬──────────────┘
                          ↓
              ┌──────────────────────────┐
              │ skipRequest()?           │
              │ - isSelf?                │
              │ - isBot?                 │
              │ - isOffline > 30s?       │
              └───────────┬──────────────┘
                          ↓
                  ┌───────┴───────┐
                  ↓               ↓
          ┌─────────────┐  ┌──────────────────┐
          │ Да (skip)   │  │ Нет              │
          └─────────────┘  └───────┬──────────┘
                                   ↓
                      ┌────────────────────────┐
                      │ MTPmessages_SetTyping( │
                      │   peer: ...,           │
                      │   action: typing       │
                      │ )                      │
                      └────────────┬───────────┘
                                   ↓
                      ┌────────────────────────┐
                      │ Сервер получает        │
                      │ Показывает другим      │
                      │ "User is typing..."    │
                      └────────────────────────┘
```

---

## Ключевые выводы

### Что отправляется при редактировании:

| Механизм | Содержание | Когда | Интервал |
|----------|-----------|-------|----------|
| **Typing indicator** | ❌ Только тип действия (typing) | При каждом изменении текста | Каждые 5 сек |
| **Local draft** | ✅ Весь текст + форматирование | При каждом изменении | 1-5 сек (локально) |
| **Cloud draft** | ✅ Весь текст + форматирование + reply + webpage | После прекращения печати | 14 сек после последнего изменения |
| **Streamed draft** | ✅ Весь текст + форматирование | При использовании collaborative editing | При каждом изменении |

### Безопасность:

1. **Typing indicator НЕ содержит текст** — отправляется только факт печати
2. **Cloud drafts отправляются редко** — только через 14 секунд после последнего изменения
3. **Streamed drafts** — опциональная фича, требует явного включения

### Производительность:

1. **Локальное сохранение** — быстрое (1-5 сек), не блокирует UI
2. **Облачная синхронизация** — отложенная (14 сек), экономит трафик
3. **Typing indicator** — оптимизирован (5 сек интервал, пропуск для оффлайн)

### Синхронизация между устройствами:

1. Черновики синхронизируются через облако
2. Используется система timestamp для разрешения конфликтов
3. Локальные изменения имеют приоритет перед облачными (если новее)

---

## Связанные документы

- [Анализ Typing Indicator](./typing_indicator_analysis.md) — подробный анализ индикатора "печатает"
- [Обработка Ctrl+V](./ctrl_v_handling.md) — как работает вставка из буфера
- [Обработка Ctrl+C](./ctrl_c_handling.md) — как работает копирование в буфер

---

