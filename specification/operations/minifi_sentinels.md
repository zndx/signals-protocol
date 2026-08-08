# MiNiFi sentinels: federation process coordination

**Status:** Binding design for federation process coordination  
**Sentinel substrate:** [Apache MiNiFi C++](https://nifi.apache.org/minifi/) — reference tree `weathership/oss-minifi-cpp` (vendored as `components/minifi-cpp` in Signals)  
**Scheduler instance (optional):** Apache YuniKorn on system-wide RKE2  
**Wire face:** `zndx.engine.v1` (engines) + MiNiFi **C2** (command & control) + **OTel** (activity truth)

This document defines how federated projects (Ægir, Atelier, Gaius, Hermes/ACP,
Signals) expose **long-running host work** to a **common control plane** without
forcing every GPU/engine process into a kubelet.

YuniKorn is **one admission/scheduling instance** of this approach—not the
definition of it. The invariant substrate is **MiNiFi C++ sentinels**.

---

## 1. Problem

| Reality | Need |
|---------|------|
| Engines run as **non-K8s** host processes (gRPC, vLLM, Flink, …) | Cross-project admission, fairness, visibility |
| Advisory GPU leases (`/tmp/zndx-gpu-leases`) only **refuse** | Ability to **plan**, queue, and pre-empt claims |
| Users need orientation across many agents/engines | Single overwatch surface (flows + C2 + metrics) |
| Scale-to-zero of *claims*, not necessarily of model weights | Control footprint must exit when work completes |

---

## 2. Architecture

```text
                    ┌──────────────────────────────────────┐
                    │  Overwatch (NiFi / MiNiFi C2 UI +    │
                    │  metrics / OTel dashboards)          │
                    └──────────────────┬───────────────────┘
                                       │ C2 heartbeats + commands
                                       │ OTel export
                    ┌──────────────────▼───────────────────┐
                    │  MiNiFi C++ sentinel(s)               │
                    │  • thin agent per workload or engine │
                    │  • flow: probe engine, emit phase    │
                    │  • optional: run as K8s pod on RKE2  │
                    └──────────────────┬───────────────────┘
                         │             │
           zndx.engine.v1│             │ admitted by (optional)
           Status /      │             │
           BeginWorkload │             ▼
                         │    ┌────────────────────┐
                         │    │ YuniKorn (instance)│
                         │    │ queues per project │
                         │    │ admits sentinel    │
                         │    │ apps / placeholders│
                         │    └────────────────────┘
                         ▼
              Host engines (non-K8s)
              aegir :50151 · atelier :50251 · gaius :50051 · …
              Kerberos principals · GPU · Impala/Ranger
```

| Layer | Where | Job |
|-------|-------|-----|
| **Execution** | Host engine processes | Real work; `zndx.engine.v1` / OIP |
| **Sentinel** | **MiNiFi C++** agent (host or RKE2 pod) | Represent work; **C2** + metrics + OTel; gate start/stop signals |
| **Admission (optional instance)** | **YuniKorn** on RKE2 | Queue/fair-share/preempt **sentinel apps** (claims), not necessarily GPU pods |
| **Identity** | Kerberos + SecretSpec | Principals for engines/agents (see [kerberos_and_secretspec.md](./kerberos_and_secretspec.md)) |
| **Overwatch UI** | MiNiFi/NiFi C2 + OTel UIs | Orient operators to processes and agents in the federation |

**Hard rule:** Sentinels are **not** the GPU workers. They are discrete **command, control, and telemetry** agents that **coordinate** host engines.

---

## 3. Why MiNiFi C++ (not a bare sleep pod)

| Capability | MiNiFi C++ provides | Federation use |
|------------|---------------------|----------------|
| **C2** | HTTP REST C2: heartbeat, DESCRIBE, UPDATE, triggers ([C2.md](https://github.com/weathership/oss-minifi-cpp/blob/main/C2.md)) | Discrete command & control protocol to start/stop/reconfigure sentinels and surface agent inventory |
| **Metrics** | System + processor metrics publishers ([METRICS.md](https://github.com/weathership/oss-minifi-cpp/blob/main/METRICS.md)) | Activity for scale-to-zero and overwatch |
| **Flows** | Processors / extensions (incl. inference-related metrics in upstream) | Probe engines, scrape OTel, write phase events |
| **Ops surface** | Deployable agent + C2 server UI path (NiFi/MiNiFi ops) | Overwatch-ready web orientation for federated systems |
| **Footprint** | C++ agent, suitable as thin host or container process | Sentinel class process |

YuniKorn remains valuable for **multi-tenant admission** of sentinel *applications* on RKE2. It is **not** required to run a single-engine lab sentinel on the host.

---

## 4. Lifecycle contract

### 4.1 Phases (OTel + C2 must agree)

| Phase | Meaning | Sentinel behavior |
|-------|---------|-------------------|
| `requested` | Work intended | Sentinel created / C2 registered |
| `admitted` | YK (if used) admitted app; else local admit | May signal engine to start |
| `loading` | Engine loading model/weights | **Not idle** — do not scale-to-zero |
| `running` | Serving / computing | Heartbeats + OTel activity |
| `complete` | Success terminal | Sentinel exits; release claim |
| `failed` | Error terminal | Sentinel exits with failure; release claim |
| ` orphan` | Engine gone without complete | Timeout → failed; free queue |

### 4.2 Required OTel attributes

Every span/metric from sentinel or engine for a coordinated workload:

| Attribute | Example |
|-----------|---------|
| `federation.project` | `aegir` \| `atelier` \| `gaius` \| `signals` \| `hermes` |
| `federation.workload_id` | UUID |
| `federation.phase` | see table above |
| `federation.principal` | Kerberos principal (if known) |
| `federation.engine_endpoint` | `host:port` for `zndx.engine.v1` |
| `federation.gpu_ids` | e.g. `[0,1]` when bound |
| `federation.sentinel_id` | MiNiFi agent identifier |

### 4.3 Engine gate

Host engines **must not** start exclusive GPU work for a federated workload until:

1. Sentinel is **admitted** (or lab mode: sentinel local-running), and  
2. Protocol handshake succeeded (`BeginWorkload` / equivalent), and  
3. Kerberos ticket available (`secretspec run` → `kinit` — see kerberos doc).

### 4.4 Scale-to-zero

| Scale what | How |
|------------|-----|
| **Sentinel process / YK app** | Exit on `complete`/`failed`; YK app finishes; queue slot free |
| **Host engine** | Optional: engine may remain warm; only **claim** scales to zero unless policy says stop endpoint |

Scale-to-zero **never** relies on missing OTel scrapes alone during `loading`. Prefer explicit phase or C2 agent status.

---

## 5. C2 as discrete command & control

MiNiFi C2 (see agent `C2.md`) provides:

- **Heartbeats** — agent alive, optional full manifest then lightweight  
- **DESCRIBE** — inventory (manifest, device, flow, queues)  
- **UPDATE / triggers** — reconfigure flows, file triggers  

Federation mapping:

| C2 concept | Federation use |
|------------|----------------|
| Agent id | `federation.sentinel_id` |
| DeviceInfo / AgentInformation | Host + project labels in overwatch |
| FlowInformation | Which probe/coordination flow is active |
| DESCRIBE manifest | Overwatch “who is participating” |
| UPDATE flow | Push new coordination flow without redeploying engines |
| Heartbeat absence | Orphan detection (with grace) |

C2 does **not** replace `zndx.engine.v1` for model inference; it coordinates
**sentinels and flows**. Engines remain the capability face.

---

## 6. YuniKorn as an instance

When RKE2-wide multi-tenant admission is required:

| Piece | Mapping |
|-------|---------|
| YK **application** | One federated workload claim |
| YK **queue** | `root.{project}` (aegir, atelier, gaius, …) |
| **Placeholder / thin pod** | Runs **MiNiFi C++ sentinel** (not vLLM) |
| Pod resources | Tiny CPU/memory only — **no GPU** on sentinel by default |
| Admission | Engine start gated on app Running |
| Complete | Sentinel exits → app done → capacity free |

Gang-scheduling placeholders (YK terminology) map cleanly to “sentinel up before workers,” even when workers are **off-cluster** engines.

**Without YK:** host-local MiNiFi sentinels + C2 + OTel still provide overwatch and local coordination; co-tenancy may still use `/tmp/zndx-gpu-leases` until YK is on.

---

## 7. Overwatch UX

Operators orient via:

1. **MiNiFi/NiFi C2 UI** — agents, heartbeats, DESCRIBE inventory, flow status  
2. **OTel** (Grafana/etc.) — phase timelines, GPU binding, error rates  
3. **YuniKorn UI** (when enabled) — queues, apps, preemption, fairness  
4. **Marquez-web / Atlas** — lineage of work that hit Signals SoR  

Together these answer: *who is participating, what are they doing, who is admitted, what data did they touch.*

---

## 8. Adopter procedures

### 8.1 Vendor

```bash
# In each federation project (or only signals as core host):
git submodule add git@github.com:weathership/oss-minifi-cpp.git components/minifi-cpp
# pin signals-protocol for this document
```

### 8.2 Sentinel identity

| Item | Value |
|------|--------|
| Agent class | `minifi-sentinel` |
| Labels | `federation.project`, `federation.workload_id` |
| Kerberos | Sentinel rarely needs GSSAPI; **engine** must (see kerberos doc) |
| SecretSpec | Only if sentinel calls protected APIs; prefer engine holds data-plane principal |

### 8.3 Minimal flow responsibilities

1. Register with C2 (heartbeat + agent id).  
2. Optionally wait for YK admission (if pod).  
3. Call engine lifecycle (gRPC) to start workload.  
4. Export OTel phase + scrape engine Status.  
5. On complete/fail/timeout: signal engine if needed, exit cleanly.  

### 8.4 Anti-patterns

| Don’t | Do |
|-------|-----|
| Put GPU in sentinel pod “for convenience” | Keep GPUs on host engines |
| Start engine GPU work without admit/handshake | Gate on sentinel + protocol |
| Scale-to-zero on scrape gaps during load | Use explicit `loading`/`running` phases |
| Replace Kerberos with C2 auth for Impala | C2 for agents; Kerberos for data plane |
| Invent a second overwatch protocol | Prefer C2 + OTel conventions here |

---

## 9. Relation to other ops specs

| Spec | Relation |
|------|----------|
| [Kerberos + SecretSpec](./kerberos_and_secretspec.md) | Principals for engines/ACP; sentinel usually does not replace them |
| Co-tenancy (GPU leases) | Host-local mutual exclusion; YK+sentinels graduate to planned admission |
| `zndx.engine.v1` | Capability inference and Status; sentinel is a client/coordinator |

---

## 10. Implementation phasing

| Phase | Deliverable |
|-------|-------------|
| **S0** | This document + `components/minifi-cpp` pin in Signals |
| **S1** | Lab host MiNiFi sentinel flow: probe one engine Status + OTel phases + C2 heartbeat |
| **S2** | C2 DESCRIBE inventory used in overwatch; document agent id conventions |
| **S3** | RKE2 Deployment of thin MiNiFi sentinel; optional YK queue `root.{project}` |
| **S4** | Gate `BeginWorkload` (or engine start) on sentinel admission |
| **S5** | Scale-to-zero sentinels on complete; timeout orphans; YK app cleanup |
| **S6** | Multi-project queues + preemption policy; retire pure refuse-only leases for cross-project windows |

---

## 11. Summary

- **MiNiFi C++ sentinels** = federation process coordination substrate (C2 + metrics + flows + overwatch).  
- **OTel** = activity truth for phases and scale-to-zero of **claims**.  
- **YuniKorn** = optional **instance** of admission/scheduling for sentinel apps on RKE2.  
- **Host gRPC engines** = actual work; Kerberos/Ranger unchanged.  
- **signals-protocol** binds lifecycle, attributes, and procedures so every project participates the same way.
