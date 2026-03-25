# VAPID Core Protocol Standard v1.0
## Verified Agent Protocol with Integrity and Decentralization

**Статус:** Draft Standard for Implementation  
**Версия:** 1.0.0  
**Дата:** 2026-03-25  
**Область:** Протокол межагентного взаимодействия (Protocol Layer Only)

---

## 1. Область Стандарта

### 1.1 Что определяет Core v1.0

| Компонент | Статус | Описание |
|-----------|--------|----------|
| Identity Record | **Обязательно** | DID-совместимая идентификация агентов |
| Capability Token | **Обязательно** | Авторизация на основе полномочий |
| Envelope | **Обязательно** | Единый подписанный транспортный контейнер |
| Artifact Descriptor | **Обязательно** | Ссылки на внешние артефакты |
| Error Object | **Обязательно** | Структурированные ошибки протокола |
| Extension Mechanism | **Обязательно** | Механизм расширений |
| Streaming Support | **Обязательно** | Потоковая передача для LLM-агентов |
| AI Execution Metadata | **Обязательно** | Метаданные исполнения (токены, модели, стоимость) |
| Trace Context | **Рекомендуется** | Распределённая трассировка (W3C-compatible) |

### 1.2 Что НЕ входит в Core v1.0

| Компонент | Статус | Где реализуется |
|-----------|--------|-----------------|
| Consensus protocols | Исключено | Extensions / Profiles |
| Reputation systems | Исключено | Extensions / Profiles |
| Staking & Payment | Исключено | Extensions / Profiles |
| Storage lifecycle | Исключено | Platform Layer |
| Zero-knowledge proofs | Исключено | Extensions / Profiles |
| System Prompt governance | Исключено | Platform Layer |
| Blockchain anchoring | Исключено | Extensions / Profiles |

---

## 2. Нормативные Термины

Ключевые слова **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD NOT**, **MAY** интерпретируются согласно RFC 2119.

**Conformance Classes:**
- **Core Sender** — формирует валидные envelopes и capability tokens
- **Core Receiver** — валидирует envelopes, подписи, сроки, авторизацию
- **Core Registry** — публикует identity records и статусы отзыва
- **Core Artifact Store** — публикует и отдаёт artifact descriptors
- **Core Streamer** — поддерживает потоковую передачу (streaming)

---

## 3. Принципы Дизайна

1. **Один транспортно-независимый signed envelope** — все сообщения через единый контейнер
2. **Один обязательный crypto suite** — Ed25519 для совместимости v1.0
3. **Явное разделение** — identity, authorization, artifact exchange разделены
4. **Минимизация динамического состояния** — статусы вынесены в endpoints
5. **Delivery semantics явная** — нет скрытых предположений об exactly-once
6. **Extensibility без размывания core** — критические расширения явно маркируются
7. **AI-Native** — встроенная поддержка стриминга и метаданных исполнения ИИ

---

## 4. Идентификаторы

### 4.1 Единая Модель Идентификаторов

| Сущность | Формат | Пример |
|----------|--------|--------|
| Агент | DID | `did:vapid:agent:<id>` |
| Сообщение | URN | `urn:vapid:message:<id>` |
| Арена | URN | `urn:vapid:arena:<id>` |
| Артефакт | URN | `urn:vapid:artifact:<id>` |
| Capability | URN | `urn:vapid:cap:<id>` |
| Стрим | URN | `urn:vapid:stream:<id>` |

### 4.2 Правила

- Агент **MUST** идентифицироваться через DID
- Все остальные сущности **MUST** использовать VAPID URN
- Идентификатор **MUST** быть глобально уникальным в пределах namespace
- `<id>` генерация не стандартизируется, но **MUST** быть криптографически стойкой

### 4.3 DID URL для Ключей

```text
did:vapid:agent:<agent-id>#<key-id>
```

Пример: `did:vapid:agent:planner-01#key-ed25519-3`

---

## 5. Базовые Объекты Ядра

Core v1.0 стандартизирует **пять** переносимых сущностей:

1. **Identity Record** — публичные ключи и сервисы агента
2. **Capability Token** — полномочия на действие
3. **Envelope** — транспортный контейнер сообщений
4. **Artifact Descriptor** — метаданные артефакта
5. **Error Object** — структурированная ошибка

---

## 6. Identity Record

### 6.1 Назначение

Identity Record описывает, как получить ключи верификации агента. Это DID Document в профиле VAPID.

### 6.2 Обязательные Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | DID | Идентификатор агента |
| `verification_methods` | Array | Массив verification methods |
| `authentication` | Array | Ссылки на ключи для подписи сообщений |
| `updated_at` | Timestamp | Последнее обновление |

### 6.3 Verification Method

```json
{
  "id": "did:vapid:agent:planner-01#key-1",
  "type": "Ed25519VerificationKey2020",
  "public_key_multibase": "z6MkhaXgBZDvotDkWL...",
  "status": "active"
}
```

**Статусы ключа:**
- `active` — ключ действителен
- `revoked` — ключ отозван

### 6.4 Опциональные Поля

- `service` — endpoints сервисов
- `assertion_methods` — ключи для утверждений
- `key_agreement` — ключи для шифрования
- `status_endpoint` — URI для проверки статуса отзыва (OCSP-like)
- `extensions` — расширения

### 6.5 Ротация Ключей

- Ротация **MUST NOT** требовать смены `id` агента
- Стабильность `id` — инвариант identity record
- Receiver **SHOULD** кэшировать Identity Record не более 1 часа

### 6.6 Проверка Статуса Ключа

Если указан `status_endpoint`, Receiver **SHOULD** проверять актуальность статуса ключа при:
- Первом взаимодействии с агентом
- Каждые 24 часа для активных сессий
- При получении сообщения с критическими полномочиями

---

## 7. Capability Token

### 7.1 Назначение

Capability Token определяет, какое действие субъект имеет право выполнить.

### 7.2 Обязательные Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `capability_id` | URN | Уникальный идентификатор токена |
| `issuer` | DID | Издатель токена |
| `subject` | DID | Субъект (кому выдан) |
| `resource` | URI/URN | Ресурс для доступа |
| `actions` | Array | Массив действий |
| `issued_at` | Timestamp | Время выпуска |
| `expires_at` | Timestamp | Время истечения |
| `proof` | Proof | Подпись issuer |

### 7.3 Actions

| Action | Описание |
|--------|----------|
| `read` | Чтение ресурса |
| `write` | Запись в ресурс |
| `execute` | Выполнение задачи |
| `delegate` | Делегирование полномочий |
| `audit` | Доступ к аудиту |

### 7.4 Ограничения (Constraints)

```json
"constraints": {
  "max_calls": 100,
  "max_data_size": 10485760,
  "allowed_recipients": ["did:vapid:agent:worker-02"]
}
```

### 7.5 Делегирование

```json
"delegation": {
  "allowed": true,
  "max_depth": 2,
  "chain": [
    "did:vapid:agent:orchestrator-01",
    "did:vapid:agent:planner-01"
  ]
}
```

**Receiver MUST отклонять токен, если:**
- Глубина делегирования превышает `max_depth`
- Отправитель не согласуется с субъектом цепочки
- Токен используется вне `arena_scope`
- Запрошенное действие отсутствует в `actions`

### 7.6 Срок Жизни

| Тип операции | Рекомендуемый TTL |
|--------------|-------------------|
| Обычные операции | ≤ 1 hour |
| Делегированные | ≤ 5 minutes |
| Высокорисковые | ≤ 5 minutes |

Receiver **MAY** отклонять токены с чрезмерно длинным TTL.

### 7.7 Arena Scope

```json
"arena_scope": [
  "urn:vapid:arena:planning-20260324",
  "urn:vapid:arena:execution-20260324"
]
```

---

## 8. Envelope

### 8.1 Назначение

Envelope — единственный обязательный транспортный контейнер Core v1.0. Каждое межузловое сообщение **MUST** передаваться как signed envelope.

### 8.2 Обязательные Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `protocol_version` | String | Версия протокола (1.0.x) |
| `kind` | String | Тип сообщения |
| `message_id` | URN | Уникальный ID сообщения |
| `timestamp` | Timestamp | Время создания |
| `sender` | DID | Отправитель |
| `body` | Object | Тело сообщения |
| `proof` | Proof | Подпись |

### 8.3 Рекомендуемые Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `recipients` | Array | Получатели (DID) |
| `arena_id` | URN | Контекст арены |
| `expires_at` | Timestamp | Время истечения |
| `correlation_id` | URN | ID корреляции |
| `parent_message_ids` | Array | Родительские сообщения |
| `authorization` | Object | Полномочия |
| `extensions` | Object | Расширения |
| `critical_extensions` | Array | Критические расширения |
| `stream_id` | URN | ID потока (для streaming) |
| `trace_context` | Object | W3C Trace Context |
| `execution_metadata` | Object | Метаданные исполнения ИИ |

### 8.4 Kind (Типы Сообщений)

#### Core Kinds (Обязательные)

| Kind | Описание | Направление |
|------|----------|-------------|
| `task.request` | Запрос задачи | Sender → Receiver |
| `task.result` | Результат задачи | Receiver → Sender |
| `task.stream.chunk` | Чанк потока | Receiver → Sender |
| `task.ack` | Подтверждение принятия | Receiver → Sender |
| `task.status` | Промежуточный статус | Receiver → Sender |
| `artifact.publish` | Публикация артефакта | Sender → Receiver |
| `artifact.fetch` | Запрос артефакта | Sender → Receiver |
| `system.heartbeat` | Сигнал активности | Any → Any |
| `system.error` | Ошибка протокола | Any → Sender |

#### Extension Kinds

Другие kinds **MUST** оформляться как extensions с namespaced naming:
- `com.example.audit.request`
- `org.vapid.billing.invoice`

### 8.5 Streaming Support

Для поддержки потоковой передачи LLM-агентов:

```json
{
  "protocol_version": "1.0",
  "kind": "task.stream.chunk",
  "message_id": "urn:vapid:message:chunk-001",
  "stream_id": "urn:vapid:stream:session-abc123",
  "timestamp": "2026-03-24T13:30:05Z",
  "sender": "did:vapid:agent:llm-worker-01",
  "recipients": ["did:vapid:agent:planner-01"],
  "body": {
    "schema": "https://vapid-protocol.org/schema/stream-chunk.json",
    "content_type": "application/json",
    "data": {
      "content": "Данные чанка...",
      "token_count": 50,
      "is_final": false,
      "sequence_number": 1
    }
  },
  "proof": {...}
}
```

**Поля стрим-чанка:**
- `stream_id` — группирует чанки одного потока
- `sequence_number` — порядок чанков в потоке
- `is_final` — последний чанк потока
- `token_count` — количество токенов в чанке

### 8.6 AI Execution Metadata

Для аудита, биллинга и роутинга на уровне шлюза:

```json
"execution_metadata": {
  "model_id": "urn:vapid:model:gpt-4o",
  "token_usage": {
    "input": 100,
    "output": 500,
    "total": 600
  },
  "cost_usd": 0.005,
  "latency_ms": 1200,
  "provider": "openai",
  "region": "us-east-1"
}
```

**Требования:**
- Поле **MUST** быть подписано (входит в digest)
- Receiver **MAY** использовать для биллинга без вскрытия `body.data`
- Sender **MUST** заполнять для всех задач с ИИ-инференсом

### 8.7 Trace Context (W3C-Compatible)

Для распределённой трассировки:

```json
"trace_context": {
  "traceparent": "00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01",
  "tracestate": "vapid=agent:planner-01,arena:planning-20260324"
}
```

**Требования:**
- Совместимо с W3C Trace Context
- Позволяет строить единые графики выполнения across multiple agents
- **RECOMMENDED** для production-развёртываний

### 8.8 Body

```json
"body": {
  "schema": "https://vapid-protocol.org/schema/task-request.json",
  "content_type": "application/json",
  "data": {...}
}
```

**Правила:**
- `schema` **MUST** быть absolute URI или URN
- `content_type` для Core v1.0 **MUST** быть `application/json`
- `data` — любое JSON value
- Бинарные данные **MUST** передаваться через Artifact Descriptor

### 8.9 Authorization

```json
"authorization": {
  "actor": "did:vapid:agent:planner-01",
  "capabilities": [...],
  "reason": "Task execution within arena planning-20260324"
}
```

**Правила:**
- `capabilities` **MUST NOT** содержать дубликаты по `capability_id`
- Receiver **MUST** оценивать capability tokens до обработки `body.data`
- `reason` — optional для audit trail

### 8.10 Delivery Semantics

**Core v1.0 НЕ гарантирует exactly-once delivery:**

| Правило | Описание |
|---------|----------|
| `message_id` | Идентифицирует экземпляр envelope, не бизнес-операцию |
| Replay protection | `(sender, message_id)` для защиты от повторов |
| Порядок | Глобальный порядок **MUST NOT** выводиться из `timestamp` |
| Причинность | Через `parent_message_ids` и `correlation_id` |

Для durable queues, strict ordering — использовать транспортный профиль или extension.

### 8.11 Idempotency

Для операций с побочными эффектами:

- Sender **SHOULD** использовать стабильный `correlation_id` для повторных попыток
- Payload schema **SHOULD** включать operation identifier
- Receiver **MAY** использовать `(sender, correlation_id, operation_id)` для dedup
- Без application-level контракта — нет гарантии отсутствия повторного side effect

---

## 9. Proof и Подпись

### 9.1 Обязательные Поля Proof

| Поле | Тип | Описание |
|------|-----|----------|
| `type` | String | Тип подписи |
| `verification_method` | DID URL | Ссылка на ключ |
| `created` | Timestamp | Время создания подписи |
| `digest` | String | Хэш подписываемых данных |
| `value` | Multibase | Значение подписи |

### 9.2 Crypto Suite для Core v1.0

| Параметр | Значение |
|----------|----------|
| Алгоритм | Ed25519 |
| Тип подписи | `Ed25519Signature2026` |
| Хэш | SHA-256 (`sha256:<hex>`) |
| Кодирование | Multibase base58btc (`z...`) |

### 9.3 Что Подписывается

**MUST быть подписаны:**
- Envelope (все сообщения)
- Capability Token (все токены)
- Artifact Descriptor (все дескрипторы)

**НЕ подписываются отдельно:**
- Error Object (всегда внутри подписанного envelope)
- Identity Record (подписывается при публикации в registry)

### 9.4 Future Crypto Agility

Для будущих версий:

```json
"proof": {
  "type": "Ed25519Signature2026",
  "crypto_suite": "vapid-cs-v1",
  ...
}
```

Поле `crypto_suite` позволит negotiate алгоритмы в будущих версиях.

---

## 10. Канонизация и Signing Input

### 10.1 Canonical JSON

Envelope/Token/Descriptor **MUST** сериализоваться в canonical JSON:
- JCS-compatible ordering (RFC 8785)
- Ключи сортируются лексикографически
- Без лишних пробелов и переносов строк

### 10.2 Domain Separation Prefix

Signing input **MUST** включать ASCII prefix перед canonical JSON:

| Объект | Prefix |
|--------|--------|
| Envelope | `vapid:envelope:v1\n` |
| Capability Token | `vapid:capability:v1\n` |
| Artifact Descriptor | `vapid:artifact-descriptor:v1\n` |

### 10.3 Вычисление Digest

```text
signing_input = prefix + canonical_json_bytes
digest = SHA-256(signing_input)
signature = Ed25519_SIGN(private_key, digest)
```

**Требования:**
- Байтовое представление **MUST** быть UTF-8
- `digest` вычисляется по domain-separated input до подписи
- Любая другая канонизация = несовместимость с Core v1.0

---

## 11. Обработка Сообщения (Receiver)

Core Receiver **MUST** выполнять шаги в порядке:

| Шаг | Действие |
|-----|----------|
| 1 | Проверить `protocol_version` |
| 2 | Проверить обязательные поля envelope |
| 3 | Проверить временное окно (`timestamp`, `expires_at`) |
| 4 | Replay check по `(sender, message_id)` |
| 5 | Разрешить `sender` в Identity Record |
| 6 | Найти `verification_method` |
| 7 | Проверить статус ключа (не отозван) |
| 8 | Канонизировать envelope |
| 9 | Проверить `digest` и подпись |
| 10 | Проверить capability tokens (если требуются) |
| 11 | Проверить `critical_extensions` |
| 12 | Проверить `trace_context` (если присутствует) |
| 13 | Только после — интерпретировать `body.data` |

**Если любой шаг неуспешен — сообщение MUST быть отклонено.**

---

## 12. Replay Protection

### 12.1 Требования к Envelope

Каждый envelope **MUST** иметь:
- Уникальный `message_id`
- `timestamp`
- Опциональный `expires_at` (рекомендуется)

### 12.2 Требования к Receiver

Receiver **MUST**:
- Отклонять сообщения со временем вне допустимого рассогласования
- Хранить `(sender, message_id)` в replay cache
- Не принимать повторное использование `message_id`

### 12.3 Clock Skew

Рекомендуемое допустимое рассогласование часов: **300 seconds**

### 12.4 Cache TTL

Replay cache **SHOULD** хранить записи минимум:
- Для обычных сообщений: 24 часа
- Для критических операций: 7 дней

---

## 13. Artifact Descriptor

### 13.1 Назначение

Artifact Descriptor используется для передачи ссылки на результат работы без смешивания transport и storage semantics.

### 13.2 Обязательные Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `artifact_id` | URN | Уникальный ID артефакта |
| `created_by` | DID | Создатель |
| `created_at` | Timestamp | Время создания |
| `media_type` | String | MIME type |
| `hash` | String | SHA-256 хэш |
| `size` | Integer | Размер в байтах |
| `locators` | Array | Массив URI для доступа |
| `proof` | Proof | Подпись создателя |

### 13.3 Locators

```json
"locators": [
  "https://storage.example.com/artifacts/abc123",
  "ipfs://QmX7...9z",
  "s3://bucket-name/artifacts/abc123"
]
```

**Требования:**
- `locators` **MUST** быть массивом URI
- Core **MUST** поддерживать минимум `https` locators
- Доверие к locator scheme **MUST NOT** заменять проверку `hash` и `proof`

### 13.4 Опциональные Поля

- `name` — человекочитаемое имя
- `arena_id` — связь с ареной
- `lineage` — происхождение (родительские артефакты/сообщения)
- `retention_hint` — подсказка по хранению
- `encryption` — параметры шифрования
- `extensions` — расширения

### 13.5 Lineage

```json
"lineage": {
  "parent_artifacts": ["urn:vapid:artifact:input-001"],
  "source_messages": ["urn:vapid:message:task-req-001"],
  "transformation": "llm-summarization:gpt-4o"
}
```

---

## 14. Error Object

### 14.1 Назначение

Error Object описывает структурированную ошибку и используется **только внутри signed envelope** с `kind = system.error`.

### 14.2 Обязательные Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `code` | String | Код ошибки |
| `message` | String | Сообщение |

### 14.3 Рекомендуемые Поля

| Поле | Тип | Описание |
|------|-----|----------|
| `retryable` | Boolean | Можно ли повторить |
| `details` | Object | Детали ошибки |
| `related_message_id` | URN | Связанное сообщение |

### 14.4 Canonical Error Codes

| Code Family | Описание | Примеры |
|-------------|----------|---------|
| `AUTH_*` | Ошибки авторизации | `AUTH_INVALID_SIGNATURE`, `AUTH_TOKEN_EXPIRED`, `AUTH_FORBIDDEN` |
| `PROTO_*` | Ошибки протокола | `PROTO_UNSUPPORTED_VERSION`, `PROTO_REPLAY_DETECTED`, `PROTO_UNSUPPORTED_EXTENSION` |
| `ARTIFACT_*` | Ошибки артефактов | `ARTIFACT_NOT_FOUND`, `ARTIFACT_HASH_MISMATCH` |
| `POLICY_*` | Ошибки политик | `POLICY_FORBIDDEN`, `POLICY_QUOTA_EXCEEDED` |
| `AI_*` | Ошибки ИИ-исполнения | `AI_CONTENT_SAFETY_VIOLATION`, `AI_CONTEXT_WINDOW_EXCEEDED`, `AI_MODEL_UNAVAILABLE` |

### 14.5 Пример

```json
{
  "code": "AI_CONTENT_SAFETY_VIOLATION",
  "message": "Generated content violated safety policy section 3.2",
  "retryable": false,
  "details": {
    "policy_id": "safety-policy-v2",
    "violation_type": "pii_detected",
    "confidence": 0.95
  },
  "related_message_id": "urn:vapid:message:task-req-001"
}
```

---

## 15. Extension Mechanism

### 15.1 Правила Расширений

| Правило | Требование |
|---------|------------|
| Namespacing | Ключи **SHOULD** быть namespaced (`org.example.feature`) |
| Critical | `critical_extensions` перечисляет обязательные расширения |
| Ignoring | Non-critical extensions **MAY** игнорироваться |
| Reject | Receiver **MUST** отклонять при непонятном critical extension |
| Schema | `additionalProperties: false` кроме секции `extensions` |

### 15.2 Пример

```json
"extensions": {
  "org.vapid.billing": {
    "invoice_id": "inv-2026-001",
    "amount_usd": 0.05
  },
  "com.example.audit": {
    "audit_trail_id": "audit-abc123"
  }
},
"critical_extensions": ["org.vapid.billing"]
```

---

## 16. Versioning

### 16.1 Protocol Version

- `protocol_version` **MUST** присутствовать в каждом Envelope
- Standalone Artifact Descriptor **MUST** включать `protocol_version`
- Встроенный Artifact Descriptor **MAY** опускать (наследуется от envelope)

### 16.2 Semantic Versioning

| Часть | Изменения |
|-------|-----------|
| Major | Breaking changes в wire format или semantics |
| Minor | Новые optional fields, новые non-critical kinds |
| Patch | Исправления, не меняющие semantics |

### 16.3 Negotiation

Sender **SHOULD NOT** отправлять version, которую receiver объявил как unsupported.

---

## 17. Минимальный Пример Envelope

### 17.1 Task Request

```json
{
  "protocol_version": "1.0",
  "kind": "task.request",
  "message_id": "urn:vapid:message:4f1e6d5b8b0c4d1f",
  "timestamp": "2026-03-24T13:30:00Z",
  "expires_at": "2026-03-24T13:35:00Z",
  "sender": "did:vapid:agent:planner-01",
  "recipients": ["did:vapid:agent:worker-02"],
  "arena_id": "urn:vapid:arena:planning-20260324",
  "correlation_id": "urn:vapid:message:session-abc123",
  "trace_context": {
    "traceparent": "00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01",
    "tracestate": "vapid=agent:planner-01"
  },
  "authorization": {
    "actor": "did:vapid:agent:planner-01",
    "capabilities": [{
      "capability_id": "urn:vapid:cap:32af7b01",
      "issuer": "did:vapid:agent:orchestrator-01",
      "subject": "did:vapid:agent:planner-01",
      "resource": "urn:vapid:arena:planning-20260324/task-queue",
      "actions": ["execute"],
      "issued_at": "2026-03-24T13:00:00Z",
      "expires_at": "2026-03-24T14:00:00Z",
      "proof": {...}
    }],
    "reason": "Task execution within arena"
  },
  "body": {
    "schema": "https://vapid-protocol.org/schema/task-request.json",
    "content_type": "application/json",
    "data": {
      "task_id": "task-42",
      "objective": "Summarize incident log and propose remediation steps"
    }
  },
  "proof": {
    "type": "Ed25519Signature2026",
    "verification_method": "did:vapid:agent:planner-01#key-3",
    "created": "2026-03-24T13:30:00Z",
    "digest": "sha256:7e2c7b9c2f0ad8d2d3c31d4ec4bf9f2fb7d20cc9fe8b31b48d4970f2dbf54a0e",
    "value": "z5mP1k9D..."
  }
}
```

### 17.2 Stream Chunk

```json
{
  "protocol_version": "1.0",
  "kind": "task.stream.chunk",
  "message_id": "urn:vapid:message:chunk-001",
  "stream_id": "urn:vapid:stream:session-abc123",
  "timestamp": "2026-03-24T13:30:05Z",
  "sender": "did:vapid:agent:llm-worker-01",
  "recipients": ["did:vapid:agent:planner-01"],
  "correlation_id": "urn:vapid:message:task-req-001",
  "body": {
    "schema": "https://vapid-protocol.org/schema/stream-chunk.json",
    "content_type": "application/json",
    "data": {
      "content": "Данные чанка...",
      "token_count": 50,
      "is_final": false,
      "sequence_number": 1
    }
  },
  "execution_metadata": {
    "model_id": "urn:vapid:model:gpt-4o",
    "token_usage": {"input": 100, "output": 50},
    "cost_usd": 0.002,
    "latency_ms": 150
  },
  "proof": {...}
}
```

### 17.3 System Error

```json
{
  "protocol_version": "1.0",
  "kind": "system.error",
  "message_id": "urn:vapid:message:error-001",
  "timestamp": "2026-03-24T13:30:10Z",
  "sender": "did:vapid:agent:worker-02",
  "recipients": ["did:vapid:agent:planner-01"],
  "correlation_id": "urn:vapid:message:task-req-001",
  "body": {
    "schema": "https://vapid-protocol.org/schema/error.json",
    "content_type": "application/json",
    "data": {
      "code": "AI_CONTENT_SAFETY_VIOLATION",
      "message": "Generated content violated safety policy",
      "retryable": false,
      "details": {"violation_type": "pii_detected"}
    }
  },
  "proof": {...}
}
```

---

## 18. Security Considerations

### 18.1 Обязательная Подпись

Все объекты, пересекающие границу доверия, **MUST** быть подписаны:
- Envelope — всегда
- Capability Token — всегда
- Artifact Descriptor — всегда

### 18.2 Key Revocation

- Identity Record **MAY** содержать `status_endpoint` для проверки отзыва
- Receiver **SHOULD** проверять статус ключа при первом взаимодействии
- Кэширование Identity Record **MUST NOT** превышать 1 час без проверки

### 18.3 Clock Skew

- Допустимое рассогласование: 300 секунд
- Сообщения вне окна **MUST** быть отклонены
- Receiver **SHOULD** использовать NTP для синхронизации

### 18.4 Replay Attacks

- `(sender, message_id)` **MUST** кэшироваться
- Повторное использование **MUST** отклоняться
- Cache TTL: минимум 24 часа

### 18.5 Confidentiality

- Core v1.0 **НЕ** шифрует envelope по умолчанию
- Конфиденциальность обеспечивается транспортом (TLS)
- Для end-to-end шифрования — использовать extension с `key_agreement`

---

## 19. Conformance Testing

### 19.1 Test Vectors

Для верификации реализации предоставляются:
- Test Identity Records с известными ключами
- Test Capability Tokens с известными подписями
- Test Envelopes с известными digest и signature

### 19.2 Validation Checklist

| Требование | Проверка |
|------------|----------|
| DID формат | Соответствует pattern |
| URN формат | Соответствует pattern |
| Timestamp | RFC 3339 с timezone |
| Signature | Ed25519, multibase |
| Canonical JSON | JCS-compatible |
| Domain prefix | Correct per object type |
| Replay cache | Implemented |
| Error codes | Canonical families |

---

## 20. Последовательность Стандартизации

### 20.1 Зафиксировано в Core v1.0

- [x] Canonical identifiers (DID + URN)
- [x] Exact signing rules (JCS + domain prefix)
- [x] Обязательные processing steps (13 шагов Receiver)
- [x] Жесткие границы core (5 объектов)
- [x] Streaming support (task.stream.chunk)
- [x] AI execution metadata
- [x] Trace context (W3C-compatible)
- [x] Extended error codes (AI_* family)

### 20.2 План Расширений (v1.1+)

| Расширение | Приоритет | Статус |
|------------|-----------|--------|
| Envelope Encryption | High | Planned |
| Quantum-Resistant Signatures | Medium | Research |
| ZK Capability Proofs | Medium | Research |
| Billing & Settlement | High | Planned |
| Human Escalation | Medium | Planned |
| P2P Profile | Low | Considered |

---

## 21. Терминологическое Примечание

**VAPID** в контексте этого протокола означает:
> **V**erified **A**gent **P**rotocol with **I**ntegrity and **D**ecentralization

**Не путать с:**
> Web Push VAPID (Voluntary Application Server Identification, RFC 8292)

**Рекомендации для разработчиков:**
- Пакеты: `@vapid-agent/core` (не `web-push`)
- Документация: явно указывать различие
- Поисковые системы: использовать `VAPID agent protocol` для SEO

---

## 22. Приложения

### Приложение A: JSON Schema Bundle

См. отдельный документ `VAPID_CORE_PROTOCOL_SCHEMA.json` v1.0.0

### Приложение B: Security Model

См. отдельный документ `VAPID_SECURITY_MODEL.md` v1.0.0

### Приложение C: Platform Architecture

См. отдельный документ `VAPID_PLATFORM_ARCHITECTURE.md` v0.1.0 (не является частью протокола)

### Приложение D: Extension Registry

Реестр зарегистрированных расширений ведётся на:
`https://registry.vapid-protocol.org/extensions`

---

## 23. История Изменений

| Версия | Дата | Изменения |
|--------|------|-----------|
| 1.0.0-draft | 2026-03-25 | Initial draft based on critique feedback |
| 1.0.0 | 2026-03-25 | Added streaming, AI metadata, trace context, enhanced error codes |

---

## 24. Контакты и Поддержка

- **Spec Repository:** `https://github.com/vapid-protocol/spec`
- **Issue Tracker:** `https://github.com/vapid-protocol/spec/issues`
- **Discussion:** `https://discord.gg/vapid-protocol`
- **Email:** `standards@vapid-protocol.org`

---

*Документ распространяется под лицензией CC-BY-4.0*  
*© 2026 VAPID Protocol Foundation*
