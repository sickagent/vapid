# VAPID Core Protocol Standard

## Статус

- Название: `VAPID Core Protocol`
- Полная форма акронима: `Verified Agent Protocol with Integrity and Decentralization`
- Версия: `1.0.0-draft`
- Дата: `2026-03-25`
- Статус: `Draft Standard for Implementation`

Этот документ определяет минимальное межоперабельное ядро протокола для общения ИИ-агентов. Он намеренно ограничивает область стандартизации и оставляет сложные подсистемы за пределами `Core v1`.

## 1. Область стандарта

`VAPID Core Protocol` определяет:

- идентификаторы участников и объектов;
- формат signed envelope;
- базовую модель capability-based authorization;
- формат artifact descriptor;
- модель ошибок;
- version negotiation;
- extension mechanism.

`Core v1` не определяет:

- consensus protocols;
- reputation systems;
- staking;
- payment rails;
- storage lifecycle tiers;
- zero-knowledge proofs;
- trusted execution attestation;
- system prompt governance;
- blockchain anchoring.

Если реализация требует этих механизмов, они должны оформляться как отдельные `profiles` или `extensions`.

## 2. Нормативные термины

Ключевые слова `MUST`, `MUST NOT`, `REQUIRED`, `SHOULD`, `SHOULD NOT`, `MAY` интерпретируются как нормативные требования.

## 3. Принципы дизайна

`Core v1` строится на следующих принципах:

- один транспортно-независимый signed envelope;
- один обязательный crypto suite для совместимости;
- явное разделение identity, authorization и artifact exchange;
- минимизация динамического состояния внутри переносимых объектов;
- явная фиксация delivery semantics вместо скрытых предположений об exactly-once;
- строгая extensibility-модель без размывания core semantics.

## 4. Conformance Classes

Реализация может заявлять соответствие одной или нескольким ролям:

- `Core Sender`: умеет формировать валидные VAPID envelopes и capability tokens;
- `Core Receiver`: умеет валидировать envelopes, подписи, сроки действия и authorization;
- `Core Registry`: умеет публиковать identity records и статусы отзыва;
- `Core Artifact Store`: умеет публиковать и отдавать artifact descriptors.

Совместимость двух реализаций гарантируется только в пределах пересечения поддерживаемых conformance classes.

## 5. Идентификаторы

Для `Core v1` вводятся следующие правила:

- агент `MUST` идентифицироваться через DID;
- сообщения, арены, capability tokens и артефакты `MUST` использовать VAPID URN.

Рекомендуемые форматы:

- `agent_id`: `did:vapid:agent:<id>` или другой DID-compatible identifier;
- `message_id`: `urn:vapid:message:<id>`;
- `arena_id`: `urn:vapid:arena:<id>`;
- `artifact_id`: `urn:vapid:artifact:<id>`;
- `capability_id`: `urn:vapid:cap:<id>`.

`Core v1` не требует конкретного способа генерации `<id>`, но идентификатор `MUST` быть глобально уникальным в пределах своего namespace.

## 6. Базовые объекты ядра

`Core v1` стандартизирует только пять переносимых сущностей:

- `Identity Record`
- `Capability Token`
- `Envelope`
- `Artifact Descriptor`
- `Error Object`

Все остальные сущности должны передаваться как payload или extensions поверх этих объектов.

## 7. Identity Record

Identity Record описывает, как получить ключи верификации агента.

Обязательные поля:

- `id`: DID агента;
- `verification_methods`: массив поддерживаемых verification methods;
- `authentication`: ссылки на ключи, которыми агент имеет право подписывать сообщения;
- `updated_at`: timestamp последнего обновления.

Рекомендуемая структура verification method:

- `id`: стабильный идентификатор ключа;
- `type`: тип ключа;
- `public_key_multibase`: открытый ключ;
- `status`: `active` или `revoked`.

Для `Core v1` реализация `MUST` поддерживать как минимум один тип ключа: `Ed25519`.

Ротация ключей `MUST NOT` требовать смены идентификатора агента. Если deployment profile не определяет иное, стабильность `id` считается инвариантом identity record.

Identity Record `MAY` содержать:

- `service`;
- `assertion_methods`;
- `key_agreement`;
- `extensions`.

Динамические trust-поля, включая `reputation`, `stake` и `runtime status`, `MUST NOT` рассматриваться как часть базовой модели идентичности.

## 8. Capability Token

Capability Token определяет, какое действие субъект имеет право выполнить.

Обязательные поля:

- `capability_id`;
- `issuer`;
- `subject`;
- `resource`;
- `actions`;
- `issued_at`;
- `expires_at`;
- `proof`.

Рекомендуемые дополнительные поля:

- `arena_scope`;
- `audience`;
- `constraints`;
- `delegation`;
- `extensions`.

### 8.1 Обязательные правила

- `issuer` `MUST` быть идентификатором доверенной стороны или валидного делегата;
- `subject` `MUST` совпадать с отправителем сообщения или с субъектом делегирования;
- `resource` `MUST` быть comparison-stable identifier и `MUST NOT` кодировать действие внутри строки ресурса;
- receiver `MUST` сравнивать `resource` по точному строковому значению, если deployment profile явно не определяет normalization rules;
- для межреализационной совместимости senders `SHOULD` использовать absolute `URI` или `URN` resource identifiers;
- `actions` `MUST` быть отдельным массивом значений, например `read`, `write`, `execute`, `delegate`, `audit`;
- `expires_at` `MUST` быть позже `issued_at`;
- `proof` `MUST` содержать валидную подпись issuer.

### 8.2 Ограничения делегирования

Если токен делегируемый, он `MUST` явно содержать:

- `delegation.allowed`: boolean;
- `delegation.max_depth`: integer;
- `delegation.chain`: массив DID делегатов, если делегирование уже произошло.

Receiver `MUST` отклонять токен, если:

- глубина делегирования превышает `max_depth`;
- отправитель не согласуется с текущим субъектом делегированной цепочки;
- токен используется вне `arena_scope`;
- запрошенное действие отсутствует в `actions`.

### 8.3 Срок жизни токена

Для `Core v1` capability token `SHOULD` быть короткоживущим. Рекомендуемое время жизни:

- не более `1 hour` для обычных операций;
- не более `5 minutes` для делегированных или высокорисковых действий.

Receiver `MAY` отклонять токены с чрезмерно длинным TTL согласно локальной политике.

## 9. Envelope

Envelope является единственным обязательным транспортным контейнером `Core v1`.

Каждое межузловое сообщение `MUST` передаваться как signed envelope.

### 9.1 Обязательные поля Envelope

- `protocol_version`
- `kind`
- `message_id`
- `timestamp`
- `sender`
- `body`
- `proof`

### 9.2 Рекомендуемые поля Envelope

- `recipients`
- `arena_id`
- `expires_at`
- `correlation_id`
- `parent_message_ids`
- `authorization`
- `extensions`
- `critical_extensions`

### 9.3 Kind

`Core v1` резервирует следующий минимальный набор `kind`:

- `task.request`
- `task.result`
- `artifact.publish`
- `artifact.fetch`
- `system.heartbeat`
- `system.error`

Другие kinds `MUST` оформляться как extensions и `SHOULD` использовать namespaced naming, например `com.example.audit.request`.

### 9.4 Body

`Core v1` использует JSON body и не стандартизирует произвольный binary payload внутри envelope.

Body `MUST` иметь:

- `schema`: absolute `URI`, идентифицирующий schema или payload profile;
- `content_type`: для `Core v1` значение `MUST` быть `application/json`;
- `data`: JSON value.

Бинарные данные `MUST` передаваться через `Artifact Descriptor`, а не inline внутри core envelope.

Поле `schema` `MUST` трактоваться как identifier, а не как инструкция на обязательное сетевое разрешение. `URN` допустим, поскольку является URI scheme. Relative references и unnamespaced aliases не входят в `Core v1`.

### 9.5 Authorization

Если операция требует прав, envelope `MUST` включать `authorization`.

Минимальная структура:

- `capabilities`: массив embedded capability tokens;
- `actor`: DID фактического исполнителя;
- `reason`: optional string для audit trail.

Поле `authorization.capabilities` `MUST NOT` содержать один и тот же `capability_id` более одного раза. Receiver `MAY` выполнить deduplication по `capability_id` до policy evaluation, но дубликаты `MUST NOT` повышать эффективные полномочия.

Receiver `MUST` оценивать capability tokens до обработки `body.data`.

### 9.6 Error Handling

Ошибки уровня протокола `MUST` возвращаться в envelope с `kind = system.error`.

Отдельный неподписанный error channel не является частью `Core v1`.

### 9.7 Delivery Semantics

`Core v1` стандартизирует формат и проверку сообщения, но не стандартизирует обязательную сетевую модель доставки.

Из этого следуют обязательные границы интерпретации:

- `Core v1` `MUST NOT` трактоваться как протокол с гарантией `exactly-once delivery`;
- `message_id` идентифицирует конкретный экземпляр `envelope`, а не семантическую бизнес-операцию;
- повторная отправка `MAY` использовать новый `message_id`, если формируется новый экземпляр envelope;
- получатель `MUST` трактовать `(sender, message_id)` как идентификатор попытки доставки для целей replay protection;
- глобальный порядок сообщений `MUST NOT` выводиться только из `timestamp`;
- причинная связь `SHOULD` выражаться через `parent_message_ids` и `correlation_id`.

Если deployment требует durable queues, strict ordering или delivery acknowledgements, эти свойства должны определяться транспортным профилем либо отдельным расширением.

### 9.8 Idempotency и Side Effects

Для операций с внешними побочными эффектами одной replay-защиты недостаточно.

Поэтому:

- sender `SHOULD` использовать стабильный `correlation_id` для повторных попыток одной и той же семантической операции;
- application payload schema `SHOULD` включать собственный operation identifier, если операция не является естественно идемпотентной;
- receiver `MAY` использовать комбинацию из `sender`, `correlation_id` и application-level operation id для suppress-dedup логики;
- отсутствие такого application-level контракта означает, что `Core v1` не может гарантировать отсутствие повторного side effect.

`Core v1` не стандартизирует rollback, compensation и saga-coordination.

### 9.9 Cancellation и Time Bounds

`Core v1` поддерживает только базовые временные границы через `timestamp` и `expires_at`.

Следовательно:

- протокол `MUST NOT` предполагать универсальную встроенную семантику `cancel`, `pause`, `resume` или `interrupt`;
- реализация `MAY` вводить такие сообщения как extensions;
- human escalation, pause/resume и operator intervention не входят в обязательную область `Core v1`.

## 10. Proof и подпись

Envelope, Capability Token и Artifact Descriptor `MUST` быть подписаны.

Минимальная структура `proof`:

- `type`
- `verification_method`
- `created`
- `digest`
- `value`

Для `Core v1` обязательны:

- `type = Ed25519Signature2026`
- `digest = sha256:<hex>`

Подробные правила верификации определены в `VAPID_SECURITY_MODEL.md`.

## 11. Канонизация и signing input

Чтобы две реализации проверяли одну и ту же подпись одинаково, `Core v1` вводит обязательное правило:

- envelope или token `MUST` сериализоваться в canonical JSON;
- canonicalization `MUST` использовать JCS-compatible ordering;
- signing input `MUST` быть получен из объекта с сохранением всех полей, кроме `proof.value`;
- signing input `MUST` включать ASCII domain separation prefix перед canonical JSON bytes;
- prefix `MUST` быть `vapid:envelope:v1\n` для `Envelope`, `vapid:capability:v1\n` для `Capability Token` и `vapid:artifact-descriptor:v1\n` для `Artifact Descriptor`;
- байтовое представление signing input `MUST` быть UTF-8;
- `digest` `MUST` вычисляться по domain-separated signing input до подписи.

Любая реализация, использующая другую канонизацию, не является совместимой с `Core v1`.

## 12. Обработка сообщения

Receiver, претендующий на соответствие `Core Receiver`, `MUST` выполнять следующие шаги в порядке:

1. Проверить `protocol_version`.
2. Проверить обязательные поля envelope.
3. Проверить временное окно `timestamp` и `expires_at`.
4. Выполнить replay check по `message_id` и `sender`.
5. Разрешить `sender` в identity record.
6. Найти `verification_method`.
7. Канонизировать envelope.
8. Проверить `digest` и подпись.
9. Проверить capability tokens, если они обязательны для данного `kind`.
10. Проверить `critical_extensions`.
11. Только после этого интерпретировать `body.data`.

Если любой шаг завершается неуспешно, реализация `MUST` отклонить сообщение.

## 13. Replay Protection

Каждый envelope `MUST` иметь:

- уникальный `message_id`;
- `timestamp`;
- опциональный, но рекомендуемый `expires_at`.

Receiver `MUST`:

- отклонять сообщения со временем вне допустимого рассогласования часов;
- хранить `(sender, message_id)` в replay cache;
- не принимать повторное использование `message_id`.

Рекомендуемое допустимое рассогласование часов для `Core v1`: `300 seconds`.

## 14. Artifact Descriptor

Artifact Descriptor используется для передачи ссылки на результат работы без смешивания transport и storage semantics.

Обязательные поля:

- `artifact_id`
- `created_by`
- `created_at`
- `media_type`
- `hash`
- `size`
- `locators`
- `proof`

`locators` `MUST` быть массивом `URI`. `Core Artifact Store` и `Core Receiver` `MUST` поддерживать как минимум `https` locators. Дополнительные URI schemes `MAY` поддерживаться профилями или локальной политикой. Доверие к locator scheme `MUST NOT` заменять проверку descriptor `hash` и `proof`.

Artifact Descriptor `MAY` содержать:

- `name`
- `lineage`
- `retention_hint`
- `encryption`
- `extensions`

Если артефакт связан с arena, descriptor `SHOULD` содержать `arena_id`.

## 15. Error Object

Error Object описывает структурированную ошибку и используется только внутри signed envelope.

Обязательные поля:

- `code`
- `message`

Рекомендуемые поля:

- `retryable`
- `details`
- `related_message_id`

`Core v1` резервирует canonical error code families:

- `AUTH_*`
- `PROTO_*`
- `ARTIFACT_*`
- `POLICY_*`

Если ошибка напрямую соответствует каноническому случаю, sender `SHOULD` использовать соответствующий canonical code, например:

- `AUTH_INVALID_SIGNATURE`
- `AUTH_TOKEN_EXPIRED`
- `AUTH_FORBIDDEN`
- `PROTO_UNSUPPORTED_VERSION`
- `PROTO_REPLAY_DETECTED`
- `PROTO_UNSUPPORTED_EXTENSION`
- `ARTIFACT_NOT_FOUND`
- `POLICY_FORBIDDEN`

## 16. Extension Mechanism

Чтобы расширения не ломали совместимость, `Core v1` вводит следующие правила:

- любой core object `MAY` содержать поле `extensions`;
- ключи внутри `extensions` `SHOULD` быть namespaced, например `org.example.audit`;
- поле `critical_extensions` `MAY` перечислять обязательные для понимания расширения;
- каждый элемент `critical_extensions` `MUST` присутствовать как ключ в `extensions`;
- receiver `MUST` отклонять envelope, если не понимает critical extension;
- неcritical extensions `MAY` игнорироваться.

Machine-readable schemas для core objects `MUST` использовать `additionalProperties: false`, кроме секции `extensions`.

## 17. Versioning

`protocol_version` `MUST` присутствовать в каждом `Envelope`.

Standalone `Artifact Descriptor` `MUST` включать `protocol_version`.

`Artifact Descriptor`, встроенный в `Envelope`, `MAY` опускать `protocol_version`; если поле присутствует, оно `MUST` совпадать с `protocol_version` внешнего envelope.

Правила совместимости:

- major version меняется при breaking changes;
- minor version `MAY` добавлять новые optional fields и новые noncritical kinds;
- patch version `MUST NOT` менять semantics wire format.

Sender `SHOULD NOT` отправлять version, которую receiver заранее объявил как unsupported.

## 18. Минимальный пример Envelope

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
  "authorization": {
    "actor": "did:vapid:agent:planner-01",
    "capabilities": [
      {
        "capability_id": "urn:vapid:cap:32af7b01",
        "issuer": "did:vapid:agent:orchestrator-01",
        "subject": "did:vapid:agent:planner-01",
        "resource": "urn:vapid:arena:planning-20260324/task-queue",
        "actions": ["execute"],
        "issued_at": "2026-03-24T13:00:00Z",
        "expires_at": "2026-03-24T14:00:00Z",
        "proof": {
          "type": "Ed25519Signature2026",
          "verification_method": "did:vapid:agent:orchestrator-01#key-1",
          "created": "2026-03-24T13:00:00Z",
          "digest": "sha256:7cb4d3a5d9318c3e5f47d6cf0d5e6f40a4cbe0a72f7b2f9de27a4c9b9f6f3c12",
          "value": "z3v8wQfR2K4..."
        }
      }
    ]
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

## 19. Последовательность дальнейшей стандартизации

Любая машиночитаемая схема для `VAPID v1` должна выводиться из настоящего `Core Protocol Standard`, а не определять его постфактум.

Сначала должны быть зафиксированы:

- canonical identifiers;
- exact signing rules;
- обязательные processing steps;
- жесткие границы core.

Публикация расширенных `JSON Schema`-наборов и language bindings целесообразна после фиксации указанных нормативных оснований.
