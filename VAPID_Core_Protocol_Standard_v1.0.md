# VAPID Core Protocol Standard

## Статус документа

Этот документ следует читать как первую цельную редакцию нормативных требований VAPID Core Protocol для отзыва ключей и ограничения ресурсов. Он описывает базовое поведение, а не инкрементальное обновление к другой редакции.

Нумерация разделов сохранена в формате полной спецификации, поэтому документ начинается с разделов `6`, `7`, `11`, `14`, `18`, `19` и `20`.

Термины **MUST**, **SHOULD** и **MAY** используются как нормативные.

Обязательные требования этого документа распространяются на следующие классы соответствия:

| Класс соответствия | Требование |
| :----------------- | :--------- |
| `Core Receiver`    | **MUST**   |
| `Secure Agent`     | **MUST** при обработке и выполнении действий `execute` |

## Область применения

Документ определяет:

- как публикуется и проверяется статус ключей;
- как `Capability Token` ограничивает использование ресурсов;
- в каком порядке `Receiver` обязан валидировать входящее сообщение;
- какие ошибки и проверки соответствия обязательны для совместимой реализации.

---

## 6. Identity Record

### 6.6 Проверка статуса ключа (Key Revocation)

Перед выполнением чувствительных операций `Receiver` **MUST** убедиться, что ключ, используемый для проверки подписи, не отозван. Механизм проверки строится на опубликованном `status_endpoint`.

#### 6.6.1 Поле `status_endpoint`

`Identity Record` **MUST** содержать поле `status_endpoint` для агентов, которые могут выполнять действие `execute` на чувствительных ресурсах.

```json
"status_endpoint": "https://registry.vapid.org/status/did:vapid:agent:planner-01"
```

#### 6.6.2 Формат ответа статуса

`status_endpoint` **MUST** возвращать JSON следующей структуры:

```json
{
  "did": "did:vapid:agent:planner-01",
  "status": "active",
  "updated_at": "2026-03-25T10:00:00Z",
  "revoked_keys": [
    {
      "key_id": "did:vapid:agent:planner-01#key-2",
      "revoked_at": "2026-03-24T15:30:00Z",
      "reason": "key_compromise"
    }
  ],
  "proof": {
    "type": "Ed25519Signature2026",
    "verification_method": "did:vapid:registry:mainnet#key-1",
    "created": "2026-03-25T10:00:00Z",
    "value": "z5mP..."
  }
}
```

| Поле           | Тип       | Описание                         | Статус   |
| :------------- | :-------- | :------------------------------- | :------- |
| `did`          | DID       | Идентификатор агента             | **MUST** |
| `status`       | String    | `active`, `suspended`, `revoked` | **MUST** |
| `updated_at`   | Timestamp | Время последнего обновления      | **MUST** |
| `revoked_keys` | Array     | Список отозванных ключей         | **MUST** |
| `proof`        | Proof     | Подпись реестра                  | **MUST** |

#### 6.6.3 Частота проверки

`Receiver` **MUST** проверять статус ключа по следующему правилу:

| Сценарий                                     | Требование проверки            |
| :------------------------------------------- | :----------------------------- |
| Первое взаимодействие с агентом              | **MUST** проверить статус      |
| Критические операции `execute`               | **MUST** проверять перед каждым действием |
| Длительные обычные операции (сессия > 1 часа) | **MUST** перепроверять каждые 15 минут |
| Read-only операции                           | **SHOULD** проверять при первом контакте |

#### 6.6.4 Кэширование статуса

- Статус `active` **MUST NOT** кэшироваться дольше `15 минут` для критических операций.
- Статус `active` **MUST NOT** кэшироваться дольше `1 часа` для обычных операций.
- Статус `revoked` **MUST NOT** кэшироваться: он применяется немедленно.
- `Receiver` **MUST** использовать синхронизацию времени; допустимое рассогласование часов не должно превышать `300 секунд`.

#### 6.6.5 Встраивание статусного ответа в `Envelope` (опционально)

Чтобы уменьшить задержку, `Sender` **MAY** вложить недавно полученный статусный ответ в `Envelope`:

```json
"extensions": {
  "org.vapid.security.ocsp_staple": {
    "status_endpoint": "https://registry.vapid.org/status/did:vapid:agent:planner-01",
    "status_response": {
      "did": "did:vapid:agent:planner-01",
      "status": "active",
      "updated_at": "2026-03-25T10:00:00Z",
      "revoked_keys": [],
      "proof": {
        "type": "Ed25519Signature2026",
        "verification_method": "did:vapid:registry:mainnet#key-1",
        "created": "2026-03-25T10:00:00Z",
        "value": "z5mP..."
      }
    },
    "fetched_at": "2026-03-25T10:00:00Z"
  }
}
```

- `Receiver` **MAY** использовать вложенный статус, если тот получен не более `5 минут` назад.
- `Receiver` **MUST** независимо проверить подпись в `status_response`.

---

## 7. Capability Token

### 7.1 Назначение

`Capability Token` определяет, какое действие субъект имеет право выполнить, и задаёт пределы потребления ресурсов. Это защищает арену и исполнителя от бюджетных атак, бесконечных циклов и неконтролируемого расхода модели.

### 7.2 Обязательные поля

| Поле             | Тип       | Описание | Статус |
| :--------------- | :-------- | :------- | :----- |
| `capability_id`  | URN       | Уникальный идентификатор токена | **MUST** |
| `issuer`         | DID       | Издатель токена | **MUST** |
| `subject`        | DID       | Субъект, которому выдан токен | **MUST** |
| `resource`       | URI/URN   | Ресурс, к которому предоставляется доступ | **MUST** |
| `actions`        | Array     | Список разрешённых действий | **MUST** |
| `issued_at`      | Timestamp | Время выпуска | **MUST** |
| `expires_at`     | Timestamp | Время истечения | **MUST** |
| `proof`          | Proof     | Подпись `issuer` | **MUST** |
| `resource_quota` | Object    | Ограничения ресурсов для `execute` | **MUST** для `execute` |

### 7.3 Структура `resource_quota`

Для действий типа `execute` токен **MUST** содержать поле `resource_quota`:

```json
"resource_quota": {
  "max_token_usage": 50000,
  "max_cost_usd": 10.0,
  "max_duration_seconds": 3600,
  "max_calls": 100,
  "allowed_models": ["gpt-4o", "claude-3.5"],
  "quota_scope": "urn:vapid:arena:finance-q3"
}
```

| Поле                   | Тип     | Описание | Статус |
| :--------------------- | :------ | :------- | :----- |
| `max_token_usage`      | Integer | Максимальное число токенов модели (`input + output`) | **MUST** для `execute` |
| `max_cost_usd`         | Number  | Предельная стоимость выполнения в USD | **MUST** для `execute` |
| `max_duration_seconds` | Integer | Максимальная длительность выполнения | **SHOULD** |
| `max_calls`            | Integer | Максимальное число внешних вызовов | **SHOULD** |
| `allowed_models`       | Array   | Список разрешённых моделей | **RECOMMENDED** |
| `quota_scope`          | URN     | Область, в которой ведётся единый учёт квоты | **MUST** |

### 7.4 Правила применения квоты

`Receiver` **MUST** отклонить задачу до начала выполнения, если заранее известно, что:

1. выполнение превысит `max_token_usage`;
2. стоимость превысит `max_cost_usd`;
3. длительность превысит `max_duration_seconds`;
4. запрошенная модель не входит в `allowed_models`.

При исполнении задачи `Receiver`:

- **MUST** вести учёт потреблённых ресурсов в рамках `quota_scope`;
- **MUST** обеспечивать атомарность учёта, чтобы исключить race condition;
- **SHOULD** отправлять предупреждение `system.warning` при достижении `90%` квоты;
- **MUST** отклонять дальнейшее выполнение с ошибкой `POLICY_QUOTA_EXCEEDED` при достижении `100%` квоты.

### 7.5 Интеграция с биллингом

Поле `max_cost_usd` задаёт техническую границу затрат и может использоваться как крюк для платёжной логики между независимыми агентами.

- `Receiver` **MAY** требовать подтверждение оплаты до выполнения задачи, если `max_cost_usd` превышает локальный порог доверия.
- Фактические расходы **MUST** быть зафиксированы в `execution_metadata.cost_usd` и подписаны.
- Этот раздел описывает интеграционную точку; сам по себе он не стандартизирует расчёты, инвойсинг и взаиморасчёты.

---

## 11. Обработка сообщения (Receiver Pipeline)

`Core Receiver` **MUST** выполнять проверки в указанном порядке. До завершения этих шагов `body.data` интерпретировать нельзя.

| Шаг | Действие | Статус |
| :-- | :------- | :----- |
| 1   | Проверить `protocol_version` | **MUST** |
| 2   | Проверить обязательные поля `Envelope` | **MUST** |
| 3   | Проверить временное окно (`timestamp`, `expires_at`) | **MUST** |
| 4   | Выполнить replay-check по `(sender, message_id)` | **MUST** |
| 5   | Разрешить `sender` через `Identity Record` | **MUST** |
| 6   | Найти `verification_method`, указанный для подписи | **MUST** |
| 7   | Проверить, что ключ не отозван | **MUST** |
| 8   | Канонизировать `Envelope` | **MUST** |
| 9   | Проверить `digest` и подпись | **MUST** |
| 10  | Проверить `Capability Token`: действия, ресурс, арену и срок действия | **MUST** |
| 11  | Проверить доступный остаток `resource_quota` | **MUST** |
| 12  | Проверить `critical_extensions` | **MUST** |
| 13  | Только после этого интерпретировать `body.data` | **MUST** |

Если любой шаг завершился неуспешно, сообщение **MUST** быть отклонено с `system.error`.

---

## 14. Error Object

### 14.4 Канонические коды ошибок

Реализация **MUST** поддерживать как минимум следующие семейства и коды ошибок:

| Код или семейство     | Назначение | Примеры |
| :-------------------- | :--------- | :------ |
| `AUTH_*`              | Ошибки аутентификации и авторизации | `AUTH_INVALID_SIGNATURE`, `AUTH_TOKEN_EXPIRED`, `AUTH_FORBIDDEN` |
| `AUTH_KEY_REVOKED`    | Использованный ключ отозван | `AUTH_KEY_REVOKED` |
| `QUOTA_*`             | Превышение ресурсных лимитов | `QUOTA_TOKEN_LIMIT_EXCEEDED`, `QUOTA_COST_LIMIT_EXCEEDED`, `QUOTA_DURATION_EXCEEDED` |
| `PROTO_*`             | Нарушение правил протокола | `PROTO_UNSUPPORTED_VERSION`, `PROTO_REPLAY_DETECTED` |
| `ARTIFACT_*`          | Ошибки работы с артефактами | `ARTIFACT_NOT_FOUND`, `ARTIFACT_HASH_MISMATCH` |
| `POLICY_*`            | Нарушение локальных политик | `POLICY_FORBIDDEN`, `POLICY_QUOTA_EXCEEDED` |
| `AI_*`                | Ошибки ИИ-исполнения | `AI_CONTENT_SAFETY_VIOLATION`, `AI_CONTEXT_WINDOW_EXCEEDED` |

### 14.5 Примеры ошибок

#### 14.5.1 Key Revoked

```json
{
  "code": "AUTH_KEY_REVOKED",
  "message": "The verification key used for signature has been revoked",
  "retryable": false,
  "details": {
    "key_id": "did:vapid:agent:planner-01#key-2",
    "revoked_at": "2026-03-24T15:30:00Z",
    "reason": "key_compromise"
  },
  "related_message_id": "urn:vapid:message:task-req-001"
}
```

#### 14.5.2 Quota Exceeded

```json
{
  "code": "QUOTA_COST_LIMIT_EXCEEDED",
  "message": "Executing this task would exceed the cost quota for this arena",
  "retryable": false,
  "details": {
    "quota_scope": "urn:vapid:arena:finance-q3",
    "max_cost_usd": 10.0,
    "current_cost_usd": 9.85,
    "requested_cost_usd": 0.5,
    "capability_id": "urn:vapid:cap:32af7b01"
  },
  "related_message_id": "urn:vapid:message:task-req-001"
}
```

---

## 18. Security Considerations

### 18.1 Обязательная подпись

Все объекты, пересекающие границу доверия, **MUST** быть подписаны:

- `Envelope`;
- `Capability Token`;
- `Artifact Descriptor`;
- `Status Response`.

### 18.2 Отзыв ключей

- `Identity Record` **MUST** содержать `status_endpoint` для агентов с правами `execute`.
- `Receiver` **MUST** проверять статус ключа перед критическими операциями.
- Статус `active` нельзя кэшировать дольше, чем это разрешено в разделе `6.6.4`.
- Статус `revoked` **MUST** применяться немедленно.
- Эта мера защищает от использования скомпрометированных ключей в окне уязвимости.

### 18.3 Квоты ресурсов

- `Capability Token` **MUST** содержать `resource_quota` для действий `execute`.
- `Receiver` **MUST** отслеживать потребление ресурсов в `quota_scope`.
- При достижении `100%` квоты задача **MUST** быть отклонена.
- Эта мера защищает от исчерпания бюджета (`budget exhaustion`) и бесконечных циклов.

### 18.4 Рассогласование времени

- Допустимое рассогласование часов: `300 секунд`.
- Сообщения вне допустимого временного окна **MUST** быть отклонены.
- `Receiver` **SHOULD** использовать синхронизацию времени через NTP или эквивалентный механизм.

### 18.5 Replay-атаки

- Пара `(sender, message_id)` **MUST** кэшироваться.
- Повторное использование идентичного сообщения **MUST** отклоняться.
- TTL replay-cache должен быть не меньше `24 часов`.

### 18.6 Конфиденциальность

- Базовая редакция Core Protocol не шифрует `Envelope` по умолчанию.
- Конфиденциальность обеспечивается транспортным уровнем, например TLS.
- Для end-to-end шифрования следует использовать отдельное расширение с `key_agreement`.
- Шифрование не заменяет проверку подписи, квот и статуса ключей.

---

## 19. Conformance Testing

### 19.1 Тестовые векторы

Для проверки совместимости реализации должны быть доступны:

- тестовые `Identity Record` с известными ключами;
- тестовые `Capability Token` с известными подписями и квотами;
- тестовые `Envelope` с известными `digest` и `signature`;
- тестовые `Status Response` для сценариев `active` и `revoked`;
- сценарии исчерпания квоты.

### 19.2 Контрольный список соответствия

| Требование                     | Проверка |
| :----------------------------- | :------- |
| DID-формат                     | Соответствует pattern |
| URN-формат                     | Соответствует pattern |
| Timestamp                      | RFC 3339 с timezone |
| Signature                      | Ed25519, multibase |
| Canonical JSON                 | Совместимо с JCS |
| Domain prefix                  | Корректен для типа объекта |
| Replay cache                   | Реализован |
| Error codes                    | Поддерживаются канонические семейства |
| Key Revocation Check           | Реализован (**MUST**) |
| Quota Enforcement              | Реализован (**MUST**) |
| Status Endpoint Validation     | Реализован (**MUST**) |
| Quota Tracking (Atomic)        | Реализован (**MUST**) |

---

## 20. Границы первой редакции

### 20.1 Что входит в базовую редакцию

- обязательное поле `status_endpoint` для чувствительных исполнителей;
- формат `Status Response` и правила его проверки;
- частота проверки статуса ключа и правила кэширования;
- обязательное поле `resource_quota` для действий `execute`;
- атомарный учёт квоты в рамках `quota_scope`;
- порядок валидации сообщения в `Receiver Pipeline`;
- канонические коды ошибок для отзыва ключей и превышения квоты;
- обязательные тестовые векторы и контрольный список соответствия.

### 20.2 Что вынесено в расширения

| Расширение                   | Приоритет | Статус |
| :--------------------------- | :-------- | :----- |
| Envelope Encryption (HPKE)   | High      | Planned |
| Quantum-Resistant Signatures | Medium    | Research |
| ZK Capability Proofs         | Medium    | Research |
| Billing & Settlement         | High      | Planned |
| Human Escalation             | Medium    | Planned |
| P2P Profile                  | Low       | Considered |

---

## Краткое резюме требований

| Область | Базовое требование | Статус |
| :------ | :----------------- | :----- |
| Отзыв ключей | `status_endpoint` и обязательная проверка перед критическими операциями | **MUST** |
| Квоты ресурсов | `resource_quota` в `Capability Token` и атомарный учёт | **MUST** |
| Кэширование статуса | `15 минут` для критических операций, `1 час` для обычных, `0` для `revoked` | **MUST** |
| Ошибки | `AUTH_KEY_REVOKED`, `QUOTA_*`, `POLICY_QUOTA_EXCEEDED` | **MUST** |
| Биллинг-интеграция | `max_cost_usd` и `execution_metadata.cost_usd` | **MUST** для фиксации расходов |

Первая редакция VAPID Core Protocol определяет протокол не только как транспорт сообщений, но и как контролируемую среду исполнения с изоляцией ресурсов и защитой от компрометации ключей.
