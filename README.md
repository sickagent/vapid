# VAPID

VAPID Core Protocol описывает базовый протокол безопасного взаимодействия и исполнения между агентами. В текущей редакции акцент сделан на двух обязательных механизмах: отзыве ключей (`Key Revocation`) и ограничении ресурсов (`Resource Quotas`) для действий типа `execute`.

Основная спецификация находится в файле [VAPID_Core_Protocol_Standard_v1.0.md](./VAPID_Core_Protocol_Standard_v1.0.md).

## Оглавление

- [Что это](#что-это)
- [Ключевые свойства протокола](#ключевые-свойства-протокола)
- [Структура репозитория](#структура-репозитория)
- [Содержание спецификации](#содержание-спецификации)
- [Статус](#статус)

## Что это

VAPID Core Protocol задаёт минимальный нормативный слой для доверенного обмена сообщениями и контролируемого выполнения задач между агентами.

Протокол определяет:

- как публикуется и проверяется статус ключей;
- как `Capability Token` ограничивает бюджет и ресурсы выполнения;
- в каком порядке `Receiver` обязан валидировать входящее сообщение;
- какие коды ошибок и тесты соответствия обязательны для совместимой реализации.

## Ключевые свойства протокола

- Проверка отзыва ключей через `status_endpoint` перед критическими операциями.
- Ограничение бюджета выполнения через `resource_quota` в `Capability Token`.
- Атомарный учёт ресурсов в пределах `quota_scope`.
- Жёсткий `Receiver Pipeline`, который запрещает интерпретацию `body.data` до завершения проверок.
- Канонические коды ошибок для аутентификации, квот, политик и протокольных нарушений.
- Контрольный список соответствия для реализаций `Core Receiver` и `Secure Agent`.

## Структура репозитория

- [README.md](./README.md) — краткое описание протокола и навигация по репозиторию.
- [VAPID_Core_Protocol_Standard_v1.0.md](./VAPID_Core_Protocol_Standard_v1.0.md) — основная нормативная спецификация.

## Содержание спецификации

- [Статус документа](./VAPID_Core_Protocol_Standard_v1.0.md#статус-документа)
- [Область применения](./VAPID_Core_Protocol_Standard_v1.0.md#область-применения)
- [6. Identity Record](./VAPID_Core_Protocol_Standard_v1.0.md#6-identity-record)
- [7. Capability Token](./VAPID_Core_Protocol_Standard_v1.0.md#7-capability-token)
- [11. Обработка сообщения (Receiver Pipeline)](./VAPID_Core_Protocol_Standard_v1.0.md#11-обработка-сообщения-receiver-pipeline)
- [14. Error Object](./VAPID_Core_Protocol_Standard_v1.0.md#14-error-object)
- [18. Security Considerations](./VAPID_Core_Protocol_Standard_v1.0.md#18-security-considerations)
- [19. Conformance Testing](./VAPID_Core_Protocol_Standard_v1.0.md#19-conformance-testing)
- [20. Границы первой редакции](./VAPID_Core_Protocol_Standard_v1.0.md#20-границы-первой-редакции)

## Статус

Этот репозиторий содержит первую цельную редакцию VAPID Core Protocol для базовой безопасной среды исполнения между агентами. Дополнительные возможности, такие как `Envelope Encryption`, `Billing & Settlement` и `P2P Profile`, рассматриваются как будущие расширения, а не как часть обязательного core.
