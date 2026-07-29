# Архитектура и настройка

Workflow в n8n Cloud: **AI-консультант · Ночной WhatsApp**, ID `DjhDC9WLC1j05cYT`.

---

## Ноды

### 1. Wazzup Inbound
Тип: `n8n-nodes-base.webhook`

Принимает `POST` на `/wazzup-inbound`. Режим ответа — `onReceived`, то есть `200` уходит мгновенно, до генерации. Это принципиально: у Wazzup таймаут 30 секунд, а полный цикл занимает 1,5–3,5 секунды плюс ретраи.

### 2. Parse & Gate
Тип: `n8n-nodes-base.code`

Основной фильтр. По порядку:

- Отсекает тестовый пинг `{test: true}` при подписке
- Игнорирует вебхуки `statuses` — там нет объекта `messages`
- Считает расписание через `Intl.DateTimeFormat` с таймзоной `Asia/Almaty`
- Отбирает входящие: `isEcho === false` или `status === 'inbound'`. Это защита от петли — иначе бот отвечал бы на собственные сообщения
- Пропускает отредактированные и удалённые
- Обрабатывает команду `/reset` и флаг `RESET_ALL_CHATS`
- Отсекает дубли по `messageId`
- Разбирает вложения: на этапе `awaiting_payment` изображение считается квитанцией, в остальных случаях формируется пометка о типе файла
- Читает и пополняет историю диалога

Номер клиента: для WhatsApp берётся из `chatId`, поле `contact.phone` заполняется только у Telegram и MAX.

### 3. Find amoCRM Contact
Тип: `n8n-nodes-base.httpRequest`

`GET /api/v4/contacts?query={номер}&with=leads`. Ошибки не роняют цепочку — `onError: continueRegularOutput`.

Авторизация: Bearer Auth с долгосрочным токеном amoCRM.

### 4. Build Context
Тип: `n8n-nodes-base.code`

Собирает данные для промпта, вытаскивает `leadId` первой связанной сделки. Умеет разбирать ответ, пришедший строкой — amoCRM отдаёт `application/hal+json`, который n8n не всегда распознаёт автоматически.

### 5. Generate Reply + Gemini
Типы: `@n8n/n8n-nodes-langchain.chainLlm` и `lmChatGoogleGemini`

Модель `models/gemini-3.1-flash-lite`, температура 0.5, до 1024 токенов. Три попытки с паузой 2 секунды — Google периодически отвечает 503 при высокой нагрузке.

### 6. Send Gate
Тип: `n8n-nodes-base.code`

Разбирает JSON от модели, обновляет состояние диалога, проверяет белый список номеров. Возвращает один или два элемента: ответ клиенту и, при завершённой оплате, сводку менеджеру. Нода отправки выполняется для каждого элемента.

### 7. Send WhatsApp
Тип: `n8n-nodes-base.httpRequest`

`POST https://api.wazzup24.com/v3/message`, тело: `channelId`, `chatId`, `chatType`, `text`. Отдельные креды Bearer Auth с ключом Wazzup — **не те же, что у amoCRM**.

### 8. CRM Note Gate + Add amoCRM Note
Параллельная ветка от `Generate Reply`. При `stage=paid` создаёт примечание в сделке: `POST /api/v4/leads/{id}/notes`. Дубли исключены флагом в состоянии диалога.

---

## Хранение состояния

Используется `$getWorkflowStaticData('global')`. Структура:

```js
store.chats[chatId] = {
  greeted: true,        // было ли приветствие
  lang: 'ru',           // зафиксированный язык
  lastMsgId: '...',     // для отсечки дублей
  stage: 'collecting',  // этап диалога
  history: [            // до 26 реплик
    { role: 'client', text: '...' },
    { role: 'bot', text: '...' }
  ],
  notified: false,      // сводка менеджеру отправлена
  crmNoted: false       // примечание в CRM создано
}
```

При превышении 300 чатов старые записи удаляются.

---

## Учётные записи в n8n

| Название | Тип | Где используется |
|---|---|---|
| `AMOCrm API` | Bearer Auth | Find amoCRM Contact, Add amoCRM Note |
| `wazzup api` | Bearer Auth | Send WhatsApp |
| `Google Gemini(PaLM) Api account` | Google PaLM | Gemini |

Токены хранятся только в n8n. В экспорте workflow передаются идентификаторы, не значения.

---

## Развёртывание с нуля

1. Импортировать workflow в n8n
2. Создать три учётные записи из таблицы выше и привязать их к соответствующим нодам
3. Опубликовать workflow, скопировать production-адрес вебхука
4. Подписать вебхук в Wazzup:

```http
PATCH https://api.wazzup24.com/v3/webhooks
Authorization: Bearer {apiKey}
Content-Type: application/json

{
  "webhooksUri": "https://ваш-инстанс.app.n8n.cloud/webhook/wazzup-inbound",
  "subscriptions": {
    "messagesAndStatuses": true,
    "contactsAndDealsCreation": false
  }
}
```

Ответ `OK` со статусом 200 означает успех. Ответ `testPostNotPassed` — вебхук не ответил на проверочный запрос, проверьте что workflow опубликован.

**Важно:** адрес вебхука в Wazzup один на весь аккаунт, не на номер. Перед подпиской проверьте, что там сейчас: `GET https://api.wazzup24.com/v3/webhooks`. Если адрес занят другой интеграцией, она перестанет получать сообщения.

5. Проверить канал: `GET https://api.wazzup24.com/v3/channels`. Нужен `state: active`
6. Прогнать тест на своём номере при `LIVE_FOR_EVERYONE = false`
7. Переключить на всех клиентов

---

## Диагностика

**Клиент не получает ответ.** Откройте Executions. Если выполнений нет вообще — запрос не дошёл, проверьте адрес вебхука в Wazzup и что workflow опубликован.

**«Failed to fetch» у вызывающей стороны.** Workflow упал, n8n вернул ошибку без CORS-заголовков. Смотрите последнее выполнение со статусом error.

**Бот молчит, хотя выполнение прошло.** Ищите в логах ноды `Parse & Gate`: `OUTSIDE WINDOW`, `DUPLICATE`, `NON-TEXT`, `NO INBOUND MESSAGE`. Либо в `Send Gate`: `NOMER NE V SPISKE`.

**Gemini отвечает 503.** Временная нагрузка на стороне Google, лечится ретраями. Если стабильно — смените модель.

**Примечание в CRM не создалось.** В логах `NET SDELKI V CRM` — у номера нет связанной сделки. Сводка при этом всё равно уходит менеджеру в WhatsApp.
