# VAPID

Russian version: [README.md](./README.md)

VAPID Core Protocol defines a core protocol for secure interaction and execution between agents. In the current edition, the protocol treats an `Arena` as the shared space where agents gather, communicate, and solve tasks together, while also defining mandatory mechanisms for `Key Revocation` and `Resource Quotas` for `execute` actions.

The main specification is available in [VAPID_Core_Protocol_Standard_v1.0.md](./VAPID_Core_Protocol_Standard_v1.0.md).

## Table of Contents

- [What It Is](#what-it-is)
- [Arena and Agent Roles](#arena-and-agent-roles)
- [Core Protocol Properties](#core-protocol-properties)
- [Repository Structure](#repository-structure)
- [Specification Contents](#specification-contents)
- [Status](#status)

## What It Is

VAPID Core Protocol provides a minimal normative layer for trusted message exchange and controlled task execution between agents.

The protocol defines:

- what an `Arena` is as a shared coordination space;
- how agents distribute roles and coordinate work on a task;
- how key status is published and verified;
- how a `Capability Token` constrains execution budget and resource usage;
- in what order a `Receiver` must validate an incoming message;
- which error codes and conformance checks are mandatory for compatible implementations.

## Arena and Agent Roles

In VAPID Core Protocol, an `Arena` is the shared execution context in which multiple agents work on the same task, exchange messages and artifacts, and operate under common security and resource constraints.

The baseline role model includes:

- `orchestrator` — accepts the task, decomposes it into steps, and manages the work flow;
- `worker` — executes an assigned subtask and returns its result;
- `auditor` — verifies correctness, safety, and completeness of the result.

Typical task flow inside an arena:

1. `orchestrator` receives the task and defines the arena boundaries.
2. `orchestrator` assigns subtasks to `worker` agents.
3. `worker` agents execute the work and exchange intermediate results.
4. `auditor` reviews an intermediate or final result.
5. `orchestrator` makes the final decision for the task.

## Core Protocol Properties

- `Arena` acts as the base space for agent coordination and message routing.
- Agent roles separate orchestration, execution, and auditing responsibilities.
- Key revocation is enforced through `status_endpoint` checks before critical operations.
- Execution budget is constrained through `resource_quota` in the `Capability Token`.
- Resource tracking must be atomic within a `quota_scope`.
- A strict `Receiver Pipeline` prevents interpretation of `body.data` before validation completes.
- Canonical error codes cover authentication, quotas, policy failures, and protocol violations.
- A conformance checklist defines what `Core Receiver` and `Secure Agent` implementations must support.

## Repository Structure

- [README.md](./README.md) — Russian overview and repository navigation.
- [README.en.md](./README.en.md) — English overview and repository navigation.
- [VAPID_Core_Protocol_Standard_v1.0.md](./VAPID_Core_Protocol_Standard_v1.0.md) — the main normative specification.

## Specification Contents

- [Статус документа](./VAPID_Core_Protocol_Standard_v1.0.md#статус-документа)
- [Область применения](./VAPID_Core_Protocol_Standard_v1.0.md#область-применения)
- [5. Arena Model](./VAPID_Core_Protocol_Standard_v1.0.md#5-arena-model)
- [6. Identity Record](./VAPID_Core_Protocol_Standard_v1.0.md#6-identity-record)
- [7. Capability Token](./VAPID_Core_Protocol_Standard_v1.0.md#7-capability-token)
- [11. Обработка сообщения (Receiver Pipeline)](./VAPID_Core_Protocol_Standard_v1.0.md#11-обработка-сообщения-receiver-pipeline)
- [14. Error Object](./VAPID_Core_Protocol_Standard_v1.0.md#14-error-object)
- [18. Security Considerations](./VAPID_Core_Protocol_Standard_v1.0.md#18-security-considerations)
- [19. Conformance Testing](./VAPID_Core_Protocol_Standard_v1.0.md#19-conformance-testing)
- [20. Границы первой редакции](./VAPID_Core_Protocol_Standard_v1.0.md#20-границы-первой-редакции)

## Status

This repository contains the first consolidated edition of the VAPID Core Protocol for a baseline secure execution environment between agents. Additional capabilities such as `Envelope Encryption`, `Billing & Settlement`, and `P2P Profile` are treated as future extensions rather than part of the mandatory core.
