# zndx.scheduler.v1 — gRPC protocol specification

## Service

`zndx.scheduler.v1.Scheduler` — registered on the **Signals engine** (and only
there for product paths). Thin clients (signals-ui, MCP, CLI) call this
service. The engine privately talks to a **backend** (lab: Apache YuniKorn;
later PBS, SLURM, …).

Registered **beside** `zndx.engine.v1.Engine` on one listen port (multi-service
gRPC). Reflection **MUST** advertise both services on lattice ports.

YuniKorn (or any other product) is an **implementation**. It MUST NOT appear in
the service name, package, or public RPC vocabulary.

## Why a separate service

Queue / job admission is not inference. Crowding `Engine` with policy lifecycle
RPCs blurs capability boundaries. A dedicated service keeps Complete/Status/Remediate
stable while Signals grows a thick platform capability.

## Vocabulary

| Term | Meaning | Backend examples |
|------|---------|------------------|
| **partition** | Scheduling domain | YK partition, SLURM partition, PBS server |
| **queue** | Named share / ACL / quota node | YK queue, SLURM QOS+partition, PBS queue |
| **application** | Submitted workload | YK app, SLURM/PBS job |
| **task** | Running allocation | YK container, SLURM step, PBS task |
| **node** | Schedulable host | kube node, SLURM node, PBS vnode |
| **resource keys** | Backend-defined | `memory`/`vcore`/`pods` vs `ncpus`/`mem`/`gres` |

Declared policy is a `PolicyDocument` (`media_type` + `body`), not a vendor file
name. Lab YuniKorn uses `text/yaml` + a partitions document (queues.yaml shape).

## Runtime RPCs

| RPC | Role |
|-----|------|
| `ListPartitions` | Scheduling domains |
| `GetQueueTree` / `GetQueue` | Queue tree / single queue |
| `ListQueueApplications` | Workloads in a queue |
| `ListNodes` | Schedulable hosts |
| `GetPlacementPolicy` | How work is placed (rules + params) |
| `GetDeclaredConfig` / `ValidateConfig` | Policy document get / check |
| `Health` | Backend health (`backend` field names the implementation) |
| `GetDashboard` | Landing strip + history + node utilization |

Clients format resource quantities for display. The wire carries raw integers.

## Projection RPCs (Signals-owned)

Roots mirror Aegir lineup:

| Root | Meaning |
|------|---------|
| `CURRENT` | Active policy view + live overlay |
| `SCRATCH` | Edit/review pending promote |
| `ARCHIVE` | Freezes of past applied configs |

| RPC | Behavior |
|-----|----------|
| `SyncProjection` | Rebuild **current** from live ops + declared config; never clobber scratch |
| `GetProjectionIndex` / `GetProjectionNote` | Lineup navigation |
| `WriteScratchConfig` | Update scratch policy document |
| `DiffConfig` | scratch vs current (optional vs live) |
| `PromoteScratch` | validate → apply (engine-private adapter) → archive previous current → sync |
| `ListArchives` / `RestoreArchiveToScratch` | freeze catalog / restore |

Apply is **not** a public backend REST PUT. The engine’s promote adapter owns
ConfigMap / GitOps / `qmgr` / `scontrol` details.

## Capability advertising

`zndx.engine.v1.Engine/Status` **MUST** advertise:

- `capability` = `scheduler`
- `model` = backend id (`yunikorn`, `slurm`, `pbs`, …)
- `healthy=true` when the backend is reachable and the projection root is writable

## How the specification is consumed

| Consumer | Mechanism |
|----------|-----------|
| Engine / CLI / MCP / first-party UI | **Codegen** from `proto/` |
| Operators | **Reflection** + `grpcurl` |

## Lab backend (YuniKorn) — engine-private

Not part of the public contract. Signals’ first adapter maps:

| RPC | YuniKorn REST (private) |
|-----|-------------------------|
| `ListPartitions` | `GET /ws/v1/partitions` |
| `GetQueueTree` | `GET /ws/v1/partition/{p}/queues` |
| `GetQueue` | `GET /ws/v1/partition/{p}/queue/{q}` |
| `ListQueueApplications` | `GET …/queue/{q}/applications` |
| `ListNodes` | `GET /ws/v1/partition/{p}/nodes` |
| `GetPlacementPolicy` | `GET …/placementrules` |
| `GetDeclaredConfig` | `GET /ws/v1/config` → `PolicyDocument{text/yaml, body}` |
| `ValidateConfig` | `POST /ws/v1/validate-conf` |
| `Health` | `GET /ws/v1/scheduler/healthcheck` |
| `GetDashboard` | clusters + partitions + history/apps + history/containers + node-utilizations |

Promote apply: ConfigMap `yunikorn-configs` / `queues.yaml` (see Signals engine).
