# signals-protocol

The shared wire contracts of the **zndx signals federation** — the protocols by which the
sibling projects' engines (Ægir, Atelier, Gaius) invoke each other's capabilities while
remaining independently operable. Patterned after the
[KServe Open Inference Protocol](https://github.com/kserve/open-inference-protocol)
repository: versioned proto packages + a specification document per protocol, evolved
additively, adopted by reference (each project vendors this repo as a submodule and
generates bindings from `proto/`).

## Why this exists

Measured 2026-07-03, minutes after the first live cross-engine call: the per-project
engines mirror each other's message *shapes*, but gRPC method paths embed
package+service — `aegir.engine.AegirEngine` vs `atelier.engine.AtelierEngine` — so a
foreign stub gets `UNIMPLEMENTED` against a wire-identical peer. "Identical wire
contract" requires an identical service path, not just identical messages. Each engine
therefore registers the shared `zndx.engine.v1.Engine` service **additionally**, beside
its native service: the native service remains the project-internal surface; the shared
service is the federation face. One stub, any engine.

## Protocols

| package | spec | status |
|---|---|---|
| `zndx.engine.v1` | [`specification/protocol/engine_grpc.md`](specification/protocol/engine_grpc.md) | v1 — `Complete` + `Status` + `Remediate` + **`Yield`** + `ServerQuery` + `RecordLineage` + `WatchWorkload` + **`Announce`** (TTL’d S2S join); warehouse `tx_id` is RFC 9562 UUIDv7 ([`tx_id.md`](specification/protocol/tx_id.md)); data products on shared RustFS ([`data_products.md`](specification/protocol/data_products.md)); primary UI discovery in [`surfaces.md`](specification/protocol/surfaces.md) |
| `zndx.scheduler.v1` | [`specification/protocol/scheduler_grpc.md`](specification/protocol/scheduler_grpc.md) | v1 — federated **scheduler** capability (queues, policy, projection, **queue-share requests**). Lab backend: YuniKorn; not a vendor in the service name |
| `zndx.supervision.v1` | [`specification/operations/nautilus_supervision.md`](specification/operations/nautilus_supervision.md) | v1 — the **supervision grammar**: supervision tree, state machines with phase gates and horizons, objectives on the final surfaced result, the position vocabulary engines report and the ledger stamps, and the **Nautilus** supervisor service. Grammar here; a project ships an *instance* (no project is named in the protocol) |
| `zndx.verify.v1` | — | RESERVED: verification artifacts on the wire (reasoner certificates, kvasir proof DAGs — the rase_types direction from Gaius) |

## Operations (identity, secrets, process coordination)

Wire protocols alone are not enough to use **Signals core services** or to
coordinate multi-engine host work. Adopters must follow:

| Document | Purpose |
|---|---|
| [`specification/operations/kerberos_and_secretspec.md`](specification/operations/kerberos_and_secretspec.md) | **Binding** Kerberos principal catalog, SecretSpec allowlists, `kinit` process wrappers, Ranger onboarding |
| [`specification/operations/minifi_sentinels.md`](specification/operations/minifi_sentinels.md) | **Binding design** — MiNiFi C++ sentinels (C2 + OTel); **Knative Serving** scale-to-zero + **YuniKorn** admission on RKE2 |
| [`specification/operations/nautilus_supervision.md`](specification/operations/nautilus_supervision.md) | **Binding design** — **Nautilus**, the deterministic external supervisor (Erlang-style tree, phase gates, Brier-scored ledger, write-ahead bookkeeping); **Overwatch** stays engine-native reasoning. Each project ships a `zndx.supervision.v1` instance |

**Reference implementation:** [weathership/signals](https://github.com/weathership/signals) is the first federation project on Kerberos + SecretSpec, and vendors **MiNiFi C++** (`components/minifi-cpp`) and **YuniKorn core** (`components/yunikorn-core`) for sentinel coordination. Sibling engines should implement these procedures—do not invent a parallel long-term identity or process-control path.

## Adopters

| project | native service | shared service | gRPC port |
|---|---|---|---|
| signals | `zndx.scheduler.v1.Scheduler` (platform) | `zndx.engine.v1.Engine` | :50551 (lab lattice) |
| aegir | `aegir.engine.AegirEngine` | `zndx.engine.v1.Engine` | :50151 |
| atelier | `atelier.engine.AtelierEngine` | `zndx.engine.v1.Engine` | :50251 |
| gaius | (see `gaius FEDERATION.md`) | `zndx.engine.v1.Engine` | :50051 |
| hermes-agent | ACP / plugins | planned (client of core + engines) | — |

GPU co-tenancy rides the shared advisory lease dir `/tmp/zndx-gpu-leases` (per-GPU-set
lock files, project-tagged owners) plus each engine's authoritative nvidia-smi probe.
Cross-project **admission** is YuniKorn (sentinel **is** the Application).
**Yield** is C2 HTTP → `Engine/Yield` gRPC. See
[`specification/operations/minifi_sentinels.md`](specification/operations/minifi_sentinels.md).

## Evolution rules

- **Additive-only within a version**: new fields get new numbers; nothing is renamed,
  renumbered, or removed. Breaking changes mean a new versioned package (`v2`).
- **Capabilities, not models**: requests name abilities ("instruct", "referee"); the
  serving engine chooses and reports the model.
- **Engine-private details stay private**: internal serving ports, log paths, and
  process details do not cross this boundary.
- Changes land here first, by PR/commit visible to all three projects, then propagate
  by submodule bump — a shared proto is only shared if there is exactly one of it.

## Relationship to the Open Inference Protocol

This repository borrows OIP's *form* (versioned spec + proto, additive evolution) and
reserves OIP itself as the **horizon peer protocol**: when heterogeneous serving stacks
(KServe, Triton, external clusters) join the federation, `ModelInfer`/`ServerReady` are
the lingua franca, and `zndx.engine.v1.Complete` maps onto them (chat-style completion
with reasoning retention is a convenience surface OIP does not define; the mapping note
lives in the spec doc). Until then, the zndx engines federate on this lighter contract.

## Codegen

Python (the current adopters):

    python -m grpc_tools.protoc -Iproto \
        --python_out=<dst> --grpc_python_out=<dst> \
        proto/zndx/engine/v1/engine.proto

Generated code is vendored per-project (committed beside the native stubs); the proto
here is the single source of truth.
