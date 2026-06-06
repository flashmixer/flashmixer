# Flash Mixer — руководство по MCP-интеграции

Подключите любого MCP-совместимого AI-агента к **Flash Mixer**, чтобы создавать и
управлять заказами микширования Bitcoin программно.

- **Endpoint:** `https://flashmixer.io/mcp`
- **Транспорт:** HTTP Streamable
- **Протокол:** JSON-RPC 2.0
- **Аутентификация:** `Authorization: Bearer <YOUR_MCP_TOKEN>`

> *For educational and privacy purposes only.*

---

## 1. Обзор

MCP-сервер — это тонкий, **stateless** мост, позволяющий AI-агентам использовать публичные
возможности Flash Mixer через единый endpoint (`/mcp`). Он говорит на стандартном MCP,
поэтому работает с PyCharm AI Assistant, Claude Desktop и любым клиентом, поддерживающим
MCP поверх HTTP Streamable.

Он предоставляет **5 инструментов**:

| Инструмент | Назначение |
|---|---|
| [`get_pool_configuration`](#инструмент-get_pool_configuration) | Прочитать текущие лимиты пулов, диапазоны комиссий, задержки, курс BTC/USD |
| [`calculate_exchange_fees`](#инструмент-calculate_exchange_fees) | Оценка суммы к отправке + разбивка комиссий |
| [`create_exchange_order`](#инструмент-create_exchange_order) | Создать заказ микширования (1–2 отложенные выплаты) |
| [`get_order_status`](#инструмент-get_order_status) | Проверить статус, подтверждения, выплаты |
| [`trigger_payment_check`](#инструмент-trigger_payment_check) | Принудительно запустить немедленную повторную проверку платежа |

> 💡 Рекомендация: сначала вызывайте `get_pool_configuration`. Он возвращает
> **авторитетные** допустимые диапазоны (мин/макс суммы, диапазоны комиссий, варианты
> задержек) и текущий курс BTC/USD, чтобы ваш агент никогда не зашивал значения, которые
> могут измениться.

---

## 2. Аутентификация

Запросы аутентифицируются **Bearer-токеном** в заголовке `Authorization`:

```
Authorization: Bearer <YOUR_MCP_TOKEN>
```

- Токен **статический**, выдаётся под интеграцию и проверяется на **каждом** запросе.
- Отсутствующий/неверный токен → **401 Unauthorized**.
- Токен — **секрет**: никогда не коммитьте его, не раскрывайте на стороне клиента в
  публичных приложениях и ротируйте при утечке.

### Получение токена

Запросите MCP-токен доступа у поддержки: **flashmixer@proton.me** (укажите предполагаемое
использование). Вы получите токен, который нужно поместить в конфиг клиента, как показано
ниже.

> Примечание: внутри MCP-слой хранит собственные учётные данные для доступа к ядру
> сервиса; это **не** то, чем занимается интегратор. Вам нужен только ваш **Bearer-токен**.

---

## 3. Конфигурация клиента

Формат конфига — стандартная карта MCP `mcpServers`.

### PyCharm AI Assistant

**GUI:** Settings → Tools → AI Assistant → Model Context Protocol → добавить сервер:
- **URL:** `https://flashmixer.io/mcp` *(завершающий `/mcp` обязателен)*
- **Headers:** `{ "Authorization": "Bearer <YOUR_MCP_TOKEN>" }`

**Файл уровня проекта** (например, `.junie/mcp/mcp.json` в вашем репозитории):

```json
{
  "mcpServers": {
    "flash-mixer": {
      "url": "https://flashmixer.io/mcp",
      "headers": { "Authorization": "Bearer <YOUR_MCP_TOKEN>" }
    }
  }
}
```

### Claude Desktop

Добавьте в конфиг MCP Claude Desktop:

```json
{
  "mcpServers": {
    "flash-mixer": {
      "url": "https://flashmixer.io/mcp",
      "headers": { "Authorization": "Bearer <YOUR_MCP_TOKEN>" }
    }
  }
}
```

### Любой MCP HTTP-клиент

Направьте его на `https://flashmixer.io/mcp`, транспорт **HTTP Streamable**, с заголовком
`Authorization: Bearer <YOUR_MCP_TOKEN>`.

---

## 4. Список инструментов

```bash
curl -X POST https://flashmixer.io/mcp \
  -H "Authorization: Bearer <YOUR_MCP_TOKEN>" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{ "jsonrpc": "2.0", "id": 1, "method": "tools/list", "params": {} }'
```

Возвращает описания и входные схемы всех 5 инструментов.

---

## 5. Инструменты

Все вызовы используют `method: "tools/call"` с `params.name` и `params.arguments`.

### Инструмент: `get_pool_configuration`

Получить текущие лимиты пулов, диапазоны комиссий, задержки и курс BTC/USD. Используйте,
чтобы узнать, какие параметры допустимы перед созданием заказа.

- **Параметры:** нет.
- **Возвращает:** текущий курс BTC/USD + таймстамп; для каждого пула — мин/макс сумму,
  диапазон процентной комиссии, фиксированную комиссию (USD), диапазон и варианты задержек;
  общие настройки (макс. число адресов выплат, мин. сумму выплаты, TTL заказа, требуемые
  подтверждения).

```json
{
  "jsonrpc": "2.0", "id": 2, "method": "tools/call",
  "params": { "name": "get_pool_configuration", "arguments": {} }
}
```

### Инструмент: `calculate_exchange_fees`

Оценить суммарные комиссии и сумму к отправке перед созданием заказа. Берёт текущий курс
BTC/USD и считает локально.

| Параметр | Тип | Описание |
|---|---|---|
| `pool` | string | `"standard"` или `"premium"` |
| `net_amount_btc` | string | Сумма, которую вы хотите **получить**, напр. `"0.5"` |
| `fee_percent` | number | Ваша % комиссия (должна быть в диапазоне пула — см. `get_pool_configuration`) |

- **Возвращает:** желаемую сумму получения; разбивку комиссий (процентная комиссия в BTC,
  фиксированная USD→BTC, суммарная комиссия); **сумму к отправке**; текущий курс + таймстамп;
  сводку (Send X → Receive Y after Z fee).

```json
{
  "jsonrpc": "2.0", "id": 3, "method": "tools/call",
  "params": {
    "name": "calculate_exchange_fees",
    "arguments": { "pool": "standard", "net_amount_btc": "0.5", "fee_percent": 3.0 }
  }
}
```

### Инструмент: `create_exchange_order`

Создать новый заказ микширования Bitcoin с отложенными выплатами.

| Параметр | Тип | Описание |
|---|---|---|
| `pool` | string | `"standard"` или `"premium"` |
| `fee_percent` | number | Ваша % комиссия в пределах диапазона пула |
| `payouts` | array | **1–2** объекта выплат (см. ниже) |

Каждый объект `payouts[]`:

| Поле | Тип | Описание |
|---|---|---|
| `payout_address` | string | Адрес назначения BTC (`bc1…`, `1…` или `3…`) |
| `net_amount_btc` | string | Сумма, которую этот адрес должен **получить**, напр. `"0.5"` |
| `delay_hours` | integer | Задержка перед этой выплатой (0–72; ≥ 2 для Premium) |
| `payout_order` | integer | `1` или `2` |

- **Возвращает:** детали заказа, включая **order UUID**, уникальный **депозитный (платёжный)
  адрес**, время истечения и расписание выплат.

```json
{
  "jsonrpc": "2.0", "id": 4, "method": "tools/call",
  "params": {
    "name": "create_exchange_order",
    "arguments": {
      "pool": "standard",
      "fee_percent": 3.0,
      "payouts": [
        { "payout_address": "bc1qexample1...", "net_amount_btc": "0.3", "delay_hours": 2, "payout_order": 1 },
        { "payout_address": "bc1qexample2...", "net_amount_btc": "0.2", "delay_hours": 12, "payout_order": 2 }
      ]
    }
  }
}
```

> Сохраните возвращённый **order UUID** — это то, по чему вы (или ваш агент) проверяете
> статус позже. Аккаунта нет; UUID — это доступ к заказу.

### Инструмент: `get_order_status`

Проверить существующий заказ: статус платежа, подтверждения и детали выплат.

| Параметр | Тип | Описание |
|---|---|---|
| `order_uuid` | string | UUID, возвращённый `create_exchange_order` |

- **Возвращает:** статус (`new`/`confirming`/`confirmed`/`notified`/`processing`/`completed`/
  `expired`), число подтверждений (X/3) и статус по каждой выплате.

```json
{
  "jsonrpc": "2.0", "id": 5, "method": "tools/call",
  "params": { "name": "get_order_status", "arguments": { "order_uuid": "<order-uuid>" } }
}
```

### Инструмент: `trigger_payment_check`

Вручную запустить проверку платежа по заказу — полезно сразу после отправки, вместо
ожидания автоматической проверки.

| Параметр | Тип | Описание |
|---|---|---|
| `order_uuid` | string | UUID для повторной проверки |

- **Возвращает:** обнаружен ли платёж, со статусом, числом подтверждений и хешем транзакции,
  если найден.

```json
{
  "jsonrpc": "2.0", "id": 6, "method": "tools/call",
  "params": { "name": "trigger_payment_check", "arguments": { "order_uuid": "<order-uuid>" } }
}
```

---

## 6. Типичный поток агента

1. `get_pool_configuration` → узнать допустимые диапазоны + текущий курс.
2. `calculate_exchange_fees` → подтвердить сумму к отправке для желаемой суммы получения пользователя.
3. `create_exchange_order` → получить депозитный адрес + order UUID.
4. *(пользователь отправляет BTC)*
5. `get_order_status` (или `trigger_payment_check` сразу после оплаты) → отслеживать до завершения.

---

## 7. Устранение неполадок

| Симптом | Вероятная причина |
|---|---|
| `401 Unauthorized` | Отсутствует/неверен Bearer-токен, либо заголовок `Authorization` срезан прокси |
| Endpoint не найден | В URL отсутствует завершающий `/mcp` |
| Инструмент отклоняет `fee_percent` / сумму | Значение вне допустимого диапазона пула — перечитайте `get_pool_configuration` |
| Инструменты не перечислены | Клиент не использует транспорт HTTP Streamable, либо неверный endpoint |

---

## 8. Заметки и лимиты

- Endpoint **stateless** — без сессий; каждый вызов самостоятелен.
- Программный доступ **ограничен по частоте**; проектируйте агентов на backoff, а не на «долбёжку».
- Держите **токен в секрете**; ротируйте при утечке.
- Публичные возможности без ключа (config, create, status, расчёт комиссий) покрывают
  обычное использование; некоторые служебные операции ограничены — вашей интеграции они не понадобятся.

> Вопросы или запрос токена: **flashmixer@proton.me** · *For educational and privacy purposes only.*
