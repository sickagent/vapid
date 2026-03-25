# VAPID Protocol Gap List

## Статус

- Название: `VAPID Protocol Gap List`
- Версия: `0.1.0-draft`
- Дата: `2026-03-25`
- Статус: `Design Companion`

Документ фиксирует узкие места, которые остаются после выделения `Core Protocol` и `Security Model`. Его цель не расширять ядро, а показать, какие решения ещё нужны для устойчивого публичного стандарта.

## 1. Как читать этот документ

Категории:

- `Blocker for Public Core v1`: желательно закрыть до публичного стабильного релиза;
- `Extension Candidate`: допустимо выносить в отдельный профиль или расширение;
- `Operational Risk`: может не менять wire format, но критичен для production deployment.

## 2. Blockers for Public Core v1

| Gap | Почему это важно | Минимальное решение |
|-----|------------------|---------------------|
| **Публичное имя и namespace** | Рабочее имя `VAPID` конфликтует с RFC 8292 для Web Push. Это создаст путаницу в схемах, репозиториях и обсуждениях. | Выбрать новое публичное имя, namespace, schema IDs и URN strategy до stable release. |
| **Trust anchors для federation** | Даже при корректной подписи неясно, как одна сеть доверяет registry другой сети. | Зафиксировать federation trust model: trusted roots, resolver policy, cache TTL, revocation propagation. |
| **Application-level idempotency** | `message_id` защищает от replay, но не от повторных side effects одной и той же операции. | Определить guideline или companion profile для operation identifiers и duplicate suppression. |
| **Формат attestation evidence** | Без этого нельзя честно различать "ключ известен" и "runtime заслуживает доверия". | Ввести отдельный attestation profile либо явно зафиксировать, что Core не выражает runtime claims. |
| **Payload schema governance** | Envelope стандартизирован, но семантика payload быстро расползётся без registry и naming rules. | Ввести schema registry rules, versioning policy и reserved namespaces. |
| **Conformance suite** | Без тестовых векторов две реализации будут "совместимы на бумаге", но не на практике. | Подготовить canonical test vectors, negative tests, signing tests и interop fixtures. |

## 3. Operational Risks

| Risk | Что может пойти не так | Минимальное решение |
|------|------------------------|---------------------|
| **Stale revocation data** | Узлы продолжают принимать отозванные ключи или токены из кэша. | Определить maximum status age и fail-closed policy для привилегированных операций. |
| **Confused deputy** | Агент с валидным токеном выполняет действие не в том контексте или для не того актора. | Усилить binding между `actor`, `subject`, `audience`, `resource` и local policy. |
| **Оркестратор как единая точка риска** | Компрометация orchestrator/gateway даёт непропорционально большой blast radius. | Требовать separation of duties, audit trail и optional multi-party approval для критических действий. |
| **Poisoned artifacts** | Формально валидный артефакт может содержать вредный код, датасет или prompt injection payload. | Определить scanning/sanitization hooks как companion operational profile. |
| **Provenance graph explosion** | На длинных цепочках растут стоимость хранения, время верификации и сложность расследований. | Ввести snapshot compaction, lineage truncation rules и retention classes. |
| **Недостаточная observability** | Ошибки совместимости, replay rejections и policy denials трудно расследовать. | Зафиксировать минимальный набор audit events и operational metrics. |

## 4. Extension Candidates

| Candidate | Почему лучше extension | Что нужно определить |
|-----------|------------------------|----------------------|
| **Human Escalation** | Это workflow и policy, а не базовый wire format. | Pause/resume events, escalation bundle, operator decision format, SLA semantics. |
| **P2P Profile** | Discovery, NAT traversal и offline delivery усложняют ядро. | Noise/MLS profile, peer discovery, trust bootstrap, offline relay policy. |
| **Snapshot / Audit Extension** | Не всем реализациям нужен тяжелый provenance и verified state. | Snapshot manifest, verification levels, auditor roles, compaction rules. |
| **System Prompt / Policy Bundle Extension** | Prompt governance зависит от runtime и модели. | Signed policy bundle format, binding to runtime, compliance reporting semantics. |
| **Token Economics / Settlement** | Экономика не нужна для минимальной совместимости. | Fiat/credits profile, crypto settlement adapters, dispute and escrow semantics. |
| **TEE / Hardware Attestation** | Требует отдельного trust stack и vendor-specific evidence. | Evidence format, verifier API, trust roots, policy mapping. |

## 5. Дизайн-вопросы, которые ещё стоит закрыть письменно

1. Что считается "минимально достаточным" trust level для публикации агента в публичный registry?
2. Можно ли одному `agent_id` иметь несколько runtime instances одновременно, и как это отражается в audit trail?
3. Как profile negotiation работает для mixed deployments: `Core only`, `Enterprise PKI`, `P2P`, `Audit extension`?
4. Где проходит граница между `artifact descriptor` и `snapshot manifest`, если оба подписаны и оба содержат lineage?
5. Какие коды ошибок считаются canonical и резервируются стандартом, а какие остаются vendor-defined?
6. Какой минимальный privacy profile нужен для regulated deployments?

## 6. Практический next step

Перед следующим витком проектирования имеет смысл закрыть именно эти артефакты:

1. `Core Interop Test Vectors`
2. `Schema Registry and Naming Rules`
3. `Federation Trust Profile`
4. `Human Escalation Extension`
5. `Snapshot and Audit Extension`

Пока эти пять вещей не описаны, основной риск проекта не в криптографии, а в расползании семантики между реализациями.
