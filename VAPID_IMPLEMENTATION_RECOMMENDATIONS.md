# VAPID Implementation Recommendations

## Статус

- Название: `VAPID Implementation Recommendations`
- Версия: `0.1.0-draft`
- Дата: `2026-03-25`
- Статус: `Implementation Companion`

## 1. Краткий вывод

Текущий набор документов уже стал заметно лучше исходной версии:

- `Core Protocol` отделён от `Platform`;
- тяжёлые вещи вроде `consensus`, `reputation`, `staking`, `TEE` и `prompt governance` вынесены из ядра;
- есть разумная декомпозиция на `gate`, `core`, `worker`.

Но для первой реальной имплементации протокол всё ещё немного тяжелее, чем нужно для рабочего `MVP`.

Главная практическая рекомендация:

- оставить `Core Protocol` как внешний wire format;
- внутри платформы не тащить весь `Core` как внутренний язык системы;
- сначала реализовать `agent registry + discovery + peering + task routing + artifact exchange`;
- capability delegation, federation trust profiles и расширенные attestation-механизмы оставить как этап `v1.1+`.

## 2. Что уже не выглядит избыточным

Следующие решения выглядят оправданно и должны сохраниться:

- единый signed `envelope`;
- отдельный `Artifact Descriptor` вместо inline binary payload;
- разделение `ownership_scope` и `runtime_mode`;
- отдельная сущность `Peering`;
- отделение `Protocol`, `Platform` и `Runtime`;
- отказ от попытки стандартизировать `exactly-once`.

Это хорошая база именно для системы, где агенты должны:

- регистрироваться;
- находить друг друга;
- доверенно обмениваться задачами;
- возвращать результат;
- публиковать артефакты.

## 3. Где ещё есть избыточность

### 3.1 Capability model слишком широкая для первого запуска

Сейчас `Capability Token` уже включает:

- `resource`;
- `actions`;
- `audience`;
- `constraints`;
- `delegation`;
- `arena_scope`.

Для `MVP` это слишком широкий policy surface.

Для первой реализации достаточно:

- `issuer`;
- `subject`;
- `resource`;
- `actions`;
- `issued_at`;
- `expires_at`;
- `proof`.

Что лучше отложить:

- chain delegation;
- `max_depth`;
- сложные `constraints`;
- многоуровневые delegation paths.

Иначе вы начнёте строить policy engine раньше, чем заработает сама сеть агентов.

### 3.2 DID-first подход хорош для external identity, но тяжёл для local discovery

Для federation это разумно.
Для локальной платформы discovery по одному только DID неудобен.

Нужен отдельный платформенный слой discovery:

- локальный `Agent Registry`;
- индекс по возможностям агента;
- индекс по ownership и tenant visibility;
- индекс по `runtime_mode`;
- endpoint/service metadata.

То есть:

- `DID` нужен как глобальный identity anchor;
- но поиск агентов должен идти не через DID resolution, а через `core registry`.

### 3.3 Внешний протокол и внутренний transport нельзя делать одним и тем же слоем

В документах это в целом уже сказано, но это стоит зафиксировать ещё жёстче:

- внешний `VAPID envelope` нужен на границе доверия;
- внутри платформы лучше использовать внутренние команды и события;
- не каждый внутренний `NATS` event должен быть полным VAPID message.

Иначе появится лишняя сериализация, дублирование валидации и слишком сильная связанность внутренних сервисов с wire format.

### 3.4 `Arena` может стать слишком центральной сущностью слишком рано

Сейчас `Arena` фигурирует как важный контекст почти для всего.
Это полезно, но для `MVP` стоит трактовать её проще:

- как task/session namespace;
- как удобный контейнер policy scope;
- не как глобальную онтологическую единицу всех взаимодействий.

Если сделать `Arena` обязательной почти для каждого действия, она быстро превратится в источник лишней сложности.

Для `MVP` лучше разрешить:

- сообщения с `arena_id`;
- сообщения без `arena_id`, если задача точечная и не требует общего контекста.

### 3.5 Слишком ранний упор на federation trust profile

Для публичного стандарта это blocker.
Для первой работающей платформы это не blocker, а следующий этап.

Правильный порядок:

1. local agents;
2. connected agents;
3. limited federated agents с allowlist trust;
4. только потом полноценный federation trust model.

Иначе вы рискуете потратить много времени на trust fabric до того, как local orchestration вообще заработает.

## 4. Что должно войти в реальный MVP

Если цель практическая: "сервисы находят друг друга и решают задачи", то минимальный рабочий контур такой:

- `gate`
- `core`
- `worker`
- `registry`
- `artifact store`

Где:

- `gate` принимает внешние запросы и внешний VAPID;
- `core` хранит агентов, peerings, arenas, capabilities и policy;
- `worker` исполняет hosted-агентов и workflow;
- `registry` отвечает за discovery и identity metadata;
- `artifact store` хранит результаты и большие payloads.

Практически `registry` и `artifact store` на первом этапе могут быть частью `core`, а не отдельными сервисами.

Тогда MVP реально сводится к трём сервисам:

- `gate`
- `core`
- `worker`

## 5. Минимальная модель discovery

Чтобы агенты "находили друг друга", нужен не только protocol envelope, а конкретный механизм discovery.

Для `MVP` достаточно трёх операций:

### 5.1 Register Agent

`core` сохраняет:

- `agent_id`;
- `display_name`;
- `ownership_scope`;
- `runtime_mode`;
- `endpoint`;
- `trust_profile`;
- `capability_profile`;
- `status`.

### 5.2 Resolve Agent

По `agent_id` или alias система возвращает:

- identity metadata;
- routing mode;
- endpoint;
- supported message kinds;
- status;
- peering requirements.

### 5.3 Search Agents

Поиск должен работать не только по ID, но и по атрибутам:

- `tenant`;
- `kind` / role;
- supported capabilities;
- `runtime_mode`;
- tags.

Без этого агенты формально адресуемы, но практически не discoverable.

## 6. Минимальная модель peering

`Peering` для первой реализации должен быть проще, чем полноценный trust graph.

Достаточно полей:

- `source_agent_id`;
- `target_agent_id`;
- `status`;
- `direction`;
- `allowed_kinds`;
- `allowed_resources`;
- `approval_mode`.

Минимальные состояния:

- `pending`
- `active`
- `revoked`

`suspended` можно оставить, но это уже nice-to-have.

## 7. Минимальный task lifecycle

Чтобы протокол реально решал задачи, а не только передавал сообщения, нужен короткий обязательный task lifecycle:

1. `task.request`
2. `task.accepted`
3. `task.progress` optional
4. `task.result`
5. `system.error`

Сейчас в `Core` есть `task.request` и `task.result`, но для эксплуатации полезно добавить хотя бы один из двух сигналов:

- `task.accepted`
- или внутренний runtime status event вне wire protocol.

Иначе инициатору сложно отличить:

- задача ещё не начата;
- задача принята и исполняется;
- сообщение потеряно;
- политика отклонила исполнение.

Если не хочется расширять `Core`, тогда:

- оставить `task.request` и `task.result` снаружи;
- внутри платформы ввести обязательные `delivery/task status events`.

Это более практичный вариант для первого этапа.

## 8. Рекомендуемая внутренняя событийная модель

Внутри платформы лучше опираться не на полный внешний `envelope`, а на внутренние доменные события:

- `agent.registered`
- `agent.updated`
- `peering.requested`
- `peering.activated`
- `task.submitted`
- `task.accepted`
- `task.started`
- `task.completed`
- `task.failed`
- `artifact.published`
- `delivery.requested`
- `delivery.succeeded`
- `delivery.failed`

Это даст:

- более простую оркестрацию;
- меньшую связанность;
- понятную observability;
- возможность позже менять внешний transport без переписывания внутренних workflow.

## 9. Практический порядок имплементации

### Phase 1

- локальный registry агентов;
- hosted и connected agents;
- peering внутри одной платформы;
- `task.request` и `task.result`;
- artifact descriptor;
- replay protection;
- базовые capability tokens без сложной delegation.

### Phase 2

- outbound/inbound через `gate`;
- federated agents по allowlist;
- trust profile для удалённых registry;
- richer delivery metadata;
- retries и idempotency contracts.

### Phase 3

- advanced delegation;
- audit/snapshot extension;
- human escalation;
- attestation profiles;
- schema governance registry;
- полноценный federation profile.

## 10. Самое важное архитектурное решение

Если сформулировать в одном предложении:

`VAPID` должен быть внешним протоколом совместимости, а не внутренней операционной моделью всей платформы.

Тогда система получится проще:

- `core` знает, кто есть кто;
- `core` знает, кому можно писать;
- `worker` знает, как исполнять задачи;
- `gate` знает, как говорить наружу по VAPID;
- registry знает, как найти агента;
- артефакты живут отдельно от сообщений.

Именно в такой форме протокол проще довести до рабочего состояния, чем если сразу проектировать его как универсальный стандарт для discovery, trust, orchestration, governance и runtime attestations одновременно.
