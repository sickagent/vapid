# VAPID Platform Architecture

## Статус

- Название: `VAPID Platform Architecture`
- Версия: `0.1.0-draft`
- Дата: `2026-03-25`
- Статус: `Implementation Architecture Companion`

Документ описывает **систему, которая имплементирует протокол VAPID**, а не сам протокол. Это принципиально другой уровень:

- `VAPID Protocol` определяет, **как агенты обмениваются проверяемыми сообщениями**;
- `VAPID Platform` определяет, **как пользователи регистрируются, создают агентов, управляют принадлежностью, соединяют агентов друг с другом и запускают экосистему поверх протокола**.

## 1. Назначение системы

Система нужна для того, чтобы:

- регистрировать пользователей и рабочие пространства;
- создавать, подключать и каталогизировать агентов;
- различать ownership и origin агента;
- "подружить" агентов между собой через peering и доверенные каналы;
- маршрутизировать локальные и внешние VAPID-сообщения;
- запускать hosted-агентов и фоновые workflow;
- строить экосистему, где агенты разных владельцев и разных систем могут взаимодействовать через единый протокол.

## 2. Главный архитектурный принцип

Нельзя смешивать три разные вещи:

1. `Protocol`
2. `Platform`
3. `Runtime`

Поэтому:

- `Protocol` отвечает только за interoperable message layer;
- `Platform` отвечает за пользователей, ownership, registry, peering, routing и orchestration;
- `Runtime` отвечает за исполнение конкретных hosted-агентов и задач.

## 3. Базовая доменная модель

### 3.1 Основные сущности

- `User` — человек, который входит в систему.
- `Tenant` — рабочее пространство, команда или организация.
- `Agent` — зарегистрированный участник экосистемы.
- `Peering` — доверительная или разрешённая связь между агентами.
- `Arena` — рабочий контекст задачи или взаимодействия.
- `Capability` — право на действие в арене, канале или ресурсе.
- `Delivery` — попытка доставки сообщения или артефакта.

### 3.2 Ownership model

Принадлежность агента должна быть отдельной осью модели:

- `user_owned` — агент принадлежит конкретному пользователю;
- `tenant_owned` — агент принадлежит команде, проекту или организации;
- `platform_owned` — агент принадлежит самой платформе и предоставляется как встроенный системный сервис.

Это и есть три типа принадлежности, которые лучше зафиксировать как `ownership_scope`.

### 3.3 Origin / Runtime model

От ownership надо отдельно отделить то, **где и как живёт агент**:

- `hosted` — агент исполняется внутри платформы в worker runtime;
- `connected` — агент живёт вне платформы, но владелец зарегистрировал endpoint и ключи;
- `federated` — агент находится в другой VAPID-сети или внешней системе и общается через federation/gateway.

Один и тот же ownership scope может сочетаться с разными runtime mode. Например:

- `tenant_owned + hosted`
- `user_owned + connected`
- `platform_owned + hosted`

Именно это разделение снимает путаницу между "чей агент" и "где он запущен".

## 4. Минимальная архитектура платформы

Для вашего стека `Go + NATS + MongoDB` и уже существующих сущностей `worker` и `gate` рациональная минимальная архитектура выглядит так:

```text
                        ┌──────────────────────────────┐
                        │        External Agents       │
                        │      / Remote VAPID Nets     │
                        └──────────────┬───────────────┘
                                       │
                                 VAPID / HTTP
                                       │
                           ┌───────────▼───────────┐
                           │         gate          │
                           │ API + VAPID Gateway   │
                           └───────┬─────┬─────────┘
                                   │     │
                         sync API   │     │ async events
                                   │     │
                           ┌───────▼─────▼─────────┐
                           │         NATS          │
                           │   commands / events   │
                           └───────┬─────┬─────────┘
                                   │     │
                      ┌────────────▼─┐ ┌─▼────────────┐
                      │     core     │ │    worker    │
                      │ control plane│ │ runtime/jobs │
                      └──────┬───────┘ └──────┬───────┘
                             │                │
                             └──────┬─────────┘
                                    │
                             ┌──────▼──────┐
                             │   MongoDB   │
                             │ metadata/db │
                             └─────────────┘
```

## 5. Роли сервисов

### 5.1 `gate`

`gate` — это внешняя граница платформы.

Его задачи:

- принимать HTTP/gRPC/WebSocket запросы от UI, SDK и внешних систем;
- принимать входящие VAPID-сообщения от других сетей;
- выполнять быструю валидацию, auth gate и protocol gate;
- публиковать команды и события в `NATS`;
- отправлять исходящие VAPID-сообщения наружу;
- не хранить сложную бизнес-логику оркестрации.

`gate` должен быть "тонким": проверка, трансляция, маршрутизация.

### 5.2 `core`

`core` — это control plane системы.

Его задачи:

- регистрация пользователей;
- создание и управление tenant'ами;
- регистрация агентов;
- хранение ownership и runtime mode;
- хранение registry metadata;
- управление peering-записями;
- создание arena/session records;
- выпуск capability tokens;
- хранение message metadata и delivery metadata;
- принятие policy-решений для платформенного уровня.

`core` — это источник истины по метаданным.

### 5.3 `worker`

`worker` — это runtime и orchestration слой.

Его задачи:

- запуск hosted-агентов;
- выполнение длинных задач;
- обработка fan-out/fan-in workflow;
- retries и компенсационные действия на уровне runtime;
- публикация task/result событий;
- подготовка outbound deliveries;
- вызов platform-owned system agents;
- в будущем — auditors, snapshots, human escalation workflows.

`worker` не должен быть владельцем бизнес-метаданных. Он исполняет и сообщает результат обратно.

## 6. Почему именно такая декомпозиция

Этот разрез хорош потому что:

- `gate` держит внешний периметр и протокольную совместимость;
- `core` держит модель системы;
- `worker` держит выполнение;
- `NATS` убирает жёсткую связанность;
- `MongoDB` хорошо подходит для документов и гибкой эволюции схем на ранней стадии.

Это проще, чем делать сразу 10 микросервисов, но уже достаточно чисто, чтобы масштабироваться.

## 7. Основные пользовательские сценарии

### 7.1 Регистрация пользователя и создание первого агента

1. Пользователь регистрируется в платформе.
2. `core` создаёт `user` и, при необходимости, персональный `tenant`.
3. Пользователь создаёт агента.
4. У агента фиксируются:
   - `ownership_scope`
   - `runtime_mode`
   - `agent_id`
   - `display_name`
   - `capabilities`
   - `endpoint`, если агент `connected` или `federated`
5. `core` публикует `agent.registered` в `NATS`.
6. `worker` или `gate` подхватывает дальнейшую инициализацию.

### 7.2 Добавление hosted-агента

1. Пользователь создаёт агента с `runtime_mode = hosted`.
2. `core` создаёт registry metadata.
3. `worker` резервирует runtime slot и загружает конфигурацию.
4. Агент становится доступным для локальных арен и peering.

### 7.3 Добавление connected-агента

1. Пользователь регистрирует внешний endpoint и identity record агента.
2. `gate` проверяет доступность endpoint и базовый protocol handshake.
3. `core` сохраняет агента как `connected`.
4. Такой агент виден в локальном каталоге, но исполняется вне платформы.

### 7.4 Добавление federated-агента

1. Пользователь или tenant добавляет ссылку на внешний агент или внешний registry.
2. `gate` делает federation handshake по VAPID.
3. `core` создаёт локальную federated-запись.
4. Общение с агентом дальше идёт через outbound/inbound `gate`, а не через hosted runtime.

### 7.5 "Подружить" агентов

Для экосистемы важна отдельная сущность `Peering`.

`Peering` должен описывать:

- кто с кем может общаться;
- в каком направлении;
- на каких ресурсах;
- требуется ли явное одобрение;
- есть ли доверенный канал или только mediated routing через gate.

Минимальные состояния peering:

- `pending`
- `active`
- `suspended`
- `revoked`

Без этого система будет знать об агентах, но не будет знать, кому реально можно доверенно писать.

## 8. NATS как внутренняя нервная система

Внутреннее взаимодействие сервисов не должно использовать сырой внешний VAPID. Внутри платформы всё должно идти через `NATS` события и команды.

Рекомендуемые subject families:

- `vapid.user.*`
- `vapid.tenant.*`
- `vapid.agent.*`
- `vapid.peering.*`
- `vapid.arena.*`
- `vapid.capability.*`
- `vapid.message.*`
- `vapid.delivery.*`
- `vapid.system.*`

Примеры:

- `vapid.user.registered`
- `vapid.agent.register`
- `vapid.agent.registered`
- `vapid.peering.requested`
- `vapid.peering.activated`
- `vapid.arena.created`
- `vapid.message.outbound`
- `vapid.message.inbound`
- `vapid.delivery.failed`

Для надёжности лучше сразу использовать `JetStream`, а не чистый ephemeral pub/sub.

## 9. MongoDB как источник метаданных

Минимальные коллекции:

- `users`
- `tenants`
- `memberships`
- `agents`
- `peerings`
- `arenas`
- `capabilities`
- `messages`
- `deliveries`
- `artifacts`

### 9.1 Поле `agents`

У агента минимум должны быть:

- `agentId`
- `ownershipScope`
- `ownerRef`
- `runtimeMode`
- `status`
- `identityRef`
- `endpoint`
- `capabilityProfile`
- `trustProfile`
- `createdAt`

### 9.2 Важное правило

`MongoDB` хранит **метаданные**, а не должна быть основным хранилищем сырого секретного ключевого материала в открытом виде.

Если отдельного secret store пока нет, то хотя бы:

- приватные ключи должны быть зашифрованы;
- доступ к ним должен быть ограничен `gate/core`;
- `worker` должен получать только runtime-bound access, а не полный экспорт ключа по умолчанию.

## 10. Где проходит граница между local и external messaging

### Local-to-local

- оба агента известны локальной платформе;
- `core` решает policy;
- `worker` или `gate` выполняет доставку;
- внешний federation не нужен.

### Local-to-federated

- `core` проверяет peering и policy;
- `gate` сериализует внешний VAPID envelope;
- outbound идёт наружу;
- ответ возвращается через inbound `gate`.

### Hosted-to-hosted

- protocol boundary может быть внутренней, но события всё равно лучше вести через единый envelope model или внутренний message wrapper, совместимый с VAPID semantics.

## 11. Что не должно жить в первой версии системы

Чтобы не перегрузить платформу, в `v0.1` лучше не включать:

- marketplace навыков;
- on-chain staking;
- reputation scoring как критическую зависимость;
- сложный multi-region federation mesh;
- тяжёлые snapshot/audit pipelines как обязательный baseline;
- обязательный pure P2P.

Сначала нужна рабочая платформа, которая:

- регистрирует users и tenants;
- управляет агентами;
- умеет peering;
- умеет local/local и local/federated message routing;
- исполняет hosted agents через worker;
- сохраняет метаданные в Mongo;
- общается внутри через NATS.

## 12. Практический MVP

### MVP-состав

- `gate`
- `core`
- `worker`
- `nats`
- `mongodb`

### MVP-функции

- регистрация пользователя;
- создание tenant;
- создание hosted/connected agent;
- базовый agent registry;
- peering между двумя локальными агентами;
- отправка `task.request` и `task.result`;
- публикация `artifact descriptor`;
- outbound в federated agent через `gate`.

## 13. Следующий уровень после MVP

После работающего MVP можно добавлять:

- `human escalation`;
- snapshots and audit extension;
- tenant policy packs;
- federated registry sync;
- platform-owned agent marketplace;
- billing and settlement;
- `P2P profile`.

## 14. Ключевой вывод

Для вашей системы правильная базовая форма — это не "один сервис с агентами", а платформа из трёх логических слоёв:

- `gate` — внешний и протокольный периметр;
- `core` — реестр, ownership, tenants, policy и sessions;
- `worker` — исполнение и orchestration.

Три типа принадлежности агента лучше фиксировать как:

- `user_owned`
- `tenant_owned`
- `platform_owned`

И отдельно от этого хранить способ существования агента:

- `hosted`
- `connected`
- `federated`

Именно такая модель даст вам экосистему, где агентов можно не только зарегистрировать, но и реально соединять друг с другом без смешивания ownership, транспорта и runtime.
