# MiNiFi sentinels: federation process coordination

**Status:** Binding design for federation process coordination  
**Sentinel substrate:** [Apache MiNiFi C++](https://nifi.apache.org/minifi/) — reference tree `weathership/oss-minifi-cpp` (vendored as `components/minifi-cpp` in Signals)  
**K8s target (system-wide RKE2):** **Knative Serving** (scale-to-zero) + **Apache YuniKorn** (admission/queues)  
**Wire face:** `zndx.engine.v1` (engines) + MiNiFi **C2** (command & control) + **OTel** (activity truth)

This document defines how federated projects (Ægir, Atelier, Gaius, Hermes/ACP,
Signals) expose **long-running host work** to a **common control plane** without
forcing every GPU/engine process into a kubelet.

| Piece | Role |
|-------|------|
| **MiNiFi C++** | Invariant **sentinel substrate** (agent, C2, flows, metrics) |
| **Knative Serving** | **Required** K8s runtime for sentinel Services — [scale-to-zero](https://knative.dev/docs/serving/autoscaling/scale-to-zero/) via KPA |
| **YuniKorn** | **Required** admission/scheduling on RKE2 (queues, fair-share, preemption of **claims**) |

**Product target is K8s-first:** system-wide **RKE2 + Knative Serving + YuniKorn**.
There is **no supported long-term “no K8s” or “MiNiFi-only on the host” deployment**.
Host-local MiNiFi, if used at all, is **temporary engineering scaffolding** while
landing the agent image/flow—not a fallback product path and not a second
architecture.

YuniKorn alone does not scale pods to zero; **Knative KPA** does.
Knative alone does not provide multi-tenant federation queues; **YuniKorn** does.
Neither replaces **MiNiFi C2 + OTel** for agent protocol and activity truth.

---

## 1. Problem

| Reality | Need |
|---------|------|
| Engines run as **non-K8s** host processes (gRPC, vLLM, Flink, …) | Cross-project admission, fairness, visibility |
| Advisory GPU leases (`/tmp/zndx-gpu-leases`) only **refuse** | Ability to **plan**, queue, and pre-empt claims |
| Users need orientation across many agents/engines | Single overwatch surface (flows + C2 + metrics) |
| Scale-to-zero of *claims* and control footprint | Idle sentinels must not consume cluster capacity |

---

## 2. Architecture

```text
                    ┌──────────────────────────────────────┐
                    │  Overwatch (MiNiFi C2 UI + OTel +    │
                    │  YK UI + Knative revisions)          │
                    └──────────────────┬───────────────────┘
                                       │ C2 heartbeats + commands
                                       │ OTel export
                    ┌──────────────────▼───────────────────┐
                    │  Knative Service / Revision          │
                    │  (MiNiFi C++ sentinel container)     │
                    │  KPA: scale 0 ↔ N when activated     │
                    └──────────────────┬───────────────────┘
                         │             │
           zndx.engine.v1│             │ scheduled by
           Status /      │             │
           BeginWorkload │             ▼
                         │    ┌────────────────────┐
                         │    │ YuniKorn           │
                         │    │ queues root.{proj} │
                         │    │ admits sentinel    │
                         │    │ pods / apps        │
                         │    └────────────────────┘
                         ▼
              Host engines (non-K8s)
              aegir :50151 · atelier :50251 · gaius :50051 · …
              Kerberos principals · GPU · Impala/Ranger
```

| Layer | Where | Job |
|-------|-------|-----|
| **Execution** | Host engine processes | Real work; `zndx.engine.v1` / OIP |
| **Sentinel process** | **MiNiFi C++** in container | C2 + flows + OTel; coordinate engines |
| **Scale-to-zero runtime** | **Knative Serving** on RKE2 | KPA scales sentinel **revisions** to zero when idle ([docs](https://knative.dev/docs/serving/autoscaling/scale-to-zero/)) |
| **Admission / multi-tenant scheduling** | **YuniKorn** on RKE2 | Queues, fair-share, preemption of sentinel **claims** |
| **Identity** | Kerberos + SecretSpec | Principals for engines/agents ([kerberos_and_secretspec.md](./kerberos_and_secretspec.md)) |
| **Overwatch** | C2 UI + OTel + YK UI + Knative | Orient operators to federation participants |

**Hard rule:** Sentinels are **not** the GPU workers. Knative scales the **sentinel Service**, not host model weights (unless a separate policy stops the engine).

---

## 3. Why MiNiFi C++ (not a bare sleep pod)

| Capability | MiNiFi C++ provides | Federation use |
|------------|---------------------|----------------|
| **C2** | HTTP REST C2: heartbeat, DESCRIBE, UPDATE ([C2.md](https://github.com/weathership/oss-minifi-cpp/blob/main/C2.md)) | Discrete command & control; inventory for overwatch |
| **Metrics** | Metric publishers ([METRICS.md](https://github.com/weathership/oss-minifi-cpp/blob/main/METRICS.md)) | Feed OTel / activation signals |
| **Flows** | Processors / extensions | Probe engines, emit phase, react to C2 |
| **Ops surface** | Agent + C2 server path | Overwatch-ready web orientation |
| **Footprint** | C++ agent, container-friendly | Knative Revision container |

---

## 4. Knative Serving (required K8s target)

Sentinel workloads on RKE2 **must** be deployable as **Knative Services** (or equivalent Serving abstractions) so the cluster can use the **KnativePodAutoscaler (KPA)** and [scale to zero](https://knative.dev/docs/serving/autoscaling/scale-to-zero/).

### 4.1 Why Knative (with YK)

| Concern | Knative | YuniKorn |
|---------|---------|----------|
| Pod count → 0 when idle | **Yes** (KPA + `enable-scale-to-zero`) | No (schedules pods; does not own request-driven scale-to-zero) |
| Multi-tenant queues / fair-share | Partial (namespace quotas) | **Yes** (hierarchical queues `root.{project}`) |
| Cold start of thin agent | Activator + scale-from-zero | Ensures pod can be placed when demand exists |
| GPU workers | **Out of scope** for sentinel Services | Same — do not request GPUs on sentinel pods |

**Composition:** YK **places and prioritizes** sentinel pods; Knative **creates and removes** them based on activation/idle. Both are required on the production K8s target; lab host-only MiNiFi (no Knative) remains valid for S1–S2.

### 4.2 Scale-to-zero configuration (cluster)

Follow Knative Serving autoscaler ConfigMap (`knative-serving` / `config-autoscaler`):

| Setting | Federation default | Notes |
|---------|-------------------|--------|
| `enable-scale-to-zero` | **`true`** | [Global only](https://knative.dev/docs/serving/autoscaling/scale-to-zero/); requires **KPA** (not HPA-only mode) |
| `scale-to-zero-grace-period` | default `30s` (tune if dropouts) | Upper bound for network programming before last replica removed |
| `scale-to-zero-pod-retention-period` | `0s` lab; optional short retention prod | Min time last pod stays after scale-to-zero decision; per-revision annotation supported |

Per-revision annotations (when needed):

```yaml
metadata:
  annotations:
    autoscaling.knative.dev/class: kpa.autoscaling.knative.dev
    # optional: keep last pod briefly after idle
    # autoscaling.knative.dev/scale-to-zero-pod-retention-period: "30s"
```

### 4.3 What drives “idle” for a sentinel Service

Knative scales on **request concurrency / RPS** to the Service by default. Federation must **align activation with real work**:

| Pattern | Use |
|---------|-----|
| **A. Request-driven (preferred)** | Each federated workload issues an HTTP activate (or long-poll) against the Knative Service while `loading`/`running`; ends when phase is `complete`/`failed` |
| **B. Min-scale during window** | Set `autoscaling.knative.dev/min-scale: "1"` only while admitted; clear on complete (controller-owned) |
| **C. Explicit terminate** | On `complete`/`failed`, delete Revision/Service or signal activator path so KPA sees zero traffic |

**Do not** treat missing OTel scrapes alone as idle during `loading` (cold model load). Prefer:

1. OTel `federation.phase` ∈ {`loading`,`running`} ⇒ keep activated  
2. C2 AgentStatus healthy + engine Status healthy ⇒ keep activated  
3. `complete` / `failed` / orphan timeout ⇒ stop activation ⇒ Knative → 0  

### 4.4 Sentinel Knative Service shape (normative intent)

```yaml
# Illustrative — not a full production manifest
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: minifi-sentinel-aegir   # or per-workload naming
  namespace: federation-aegir
  labels:
    federation.project: aegir
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/class: kpa.autoscaling.knative.dev
        # YK queue via pod labels/annotations per cluster convention
      labels:
        federation.project: aegir
        app.kubernetes.io/component: minifi-sentinel
    spec:
      containerConcurrency: 1   # one coordination session per pod when possible
      containers:
        - image: registry.example/minifi-sentinel:…
          resources:
            requests: { cpu: 50m, memory: 128Mi }
            limits:   { cpu: 500m, memory: 512Mi }
          # NO gpu limits
```

---

## 5. Lifecycle contract

### 5.1 Phases (OTel + C2 + Knative must agree)

| Phase | Meaning | Sentinel / Knative |
|-------|---------|---------------------|
| `requested` | Work intended | Create/activate Knative Service or route |
| `admitted` | YK admitted pod (if used) | Pod Running; may handshake engine |
| `loading` | Engine loading | **Keep activated** — not scale-to-zero |
| `running` | Computing / serving | Heartbeats + OTel; keep activated |
| `complete` | Success terminal | Stop activation → **KPA scale to zero** |
| `failed` | Error terminal | Stop activation → scale to zero |
| `orphan` | Engine gone | Timeout → failed → scale to zero |

### 5.2 Required OTel attributes

| Attribute | Example |
|-----------|---------|
| `federation.project` | `aegir` \| `atelier` \| `gaius` \| `signals` \| `hermes` |
| `federation.workload_id` | UUID |
| `federation.phase` | see table above |
| `federation.principal` | Kerberos principal (if known) |
| `federation.engine_endpoint` | `host:port` for `zndx.engine.v1` |
| `federation.gpu_ids` | e.g. `[0,1]` when bound |
| `federation.sentinel_id` | MiNiFi agent identifier |
| `federation.knative_service` | Knative Service name |
| `federation.knative_revision` | Active Revision (if known) |

### 5.3 Engine gate

Host engines **must not** start exclusive GPU work for a federated workload until:

1. Sentinel is **admitted** and **activated** (Knative min replicas ≥ 1, or lab host-local MiNiFi), and  
2. Protocol handshake succeeded (`BeginWorkload` / equivalent), and  
3. Kerberos ticket available (`secretspec run` → `kinit`).

### 5.4 Scale-to-zero (summary)

| Scale what | Mechanism |
|------------|-----------|
| **Sentinel pods** | **Knative KPA** → 0 ([scale-to-zero](https://knative.dev/docs/serving/autoscaling/scale-to-zero/)) when activation ends |
| **YK queue claim** | App/pod gone → capacity free for other projects |
| **Host engine** | Optional warm pool; not Knative’s job unless separately managed |

---

## 6. C2 as discrete command & control

MiNiFi C2 provides heartbeats, DESCRIBE, UPDATE/triggers. Federation mapping:

| C2 concept | Federation use |
|------------|----------------|
| Agent id | `federation.sentinel_id` |
| DeviceInfo / AgentInformation | Overwatch inventory |
| DESCRIBE manifest | Who is participating |
| UPDATE flow | Reconfigure coordination without redeploying engines |
| Heartbeat + phase | Complements Knative metrics for “still busy” |

C2 does **not** replace `zndx.engine.v1` for inference. It does **not** replace Kerberos for Impala/Ranger.

---

## 7. YuniKorn as admission instance

| Piece | Mapping |
|-------|---------|
| YK **queue** | `root.{project}` |
| YK-scheduled pod | Knative-created MiNiFi sentinel pod |
| Resources | Tiny CPU/memory — **no GPU** |
| Preemption | Prefer preempt **idle/low-priority claims**, not `loading`/`running` without policy |

**Without YK (lab):** host-local MiNiFi + C2 + OTel still valid; no multi-tenant queue fairness.

**Without Knative (lab only):** host MiNiFi process; scale-to-zero of cluster footprint N/A until S3+.

---

## 8. Overwatch UX

1. **MiNiFi/NiFi C2 UI** — agents, DESCRIBE, flows  
2. **OTel** — phase timelines, GPU binding  
3. **Knative** — Services/Revisions, activator, scale 0↔N  
4. **YuniKorn UI** — queues, apps, preemption  
5. **Marquez-web / Atlas** — lineage into Signals SoR  

---

## 9. Adopter procedures

### 9.1 Vendor

```bash
git submodule add git@github.com:weathership/oss-minifi-cpp.git components/minifi-cpp
# pin signals-protocol for this document
# K8s target: RKE2 + Knative Serving + (recommended) YuniKorn
```

### 9.2 Sentinel identity

| Item | Value |
|------|--------|
| Agent class | `minifi-sentinel` |
| K8s labels | `federation.project`, `app.kubernetes.io/component=minifi-sentinel` |
| Knative Service naming | `minifi-sentinel-{project}` or per-`workload_id` |
| Kerberos | On **engine**; sentinel only if it calls protected APIs |

### 9.3 Minimal flow responsibilities

1. Register with C2.  
2. Run under Knative Service (prod) or host process (lab S1).  
3. Stay activated while phase ∈ {`loading`,`running`}.  
4. Handshake engine; export OTel.  
5. On terminal phase: stop activation → Knative scales to zero; release YK claim.  

### 9.4 Anti-patterns

| Don’t | Do |
|-------|-----|
| Deploy sentinel as plain Deployment without Knative in prod | Knative Service + KPA scale-to-zero |
| Put GPUs on Knative sentinel | Host engines hold GPUs |
| Use HPA-only mode that blocks scale-to-zero | Use **KPA** (`enable-scale-to-zero: true`) |
| Scale to zero on scrape gaps during `loading` | Explicit phases + activation hold |
| Assume YK alone zeros idle pods | Knative owns pod scale-to-zero |

---

## 10. Relation to other ops specs

| Spec | Relation |
|------|----------|
| [Kerberos + SecretSpec](./kerberos_and_secretspec.md) | Principals for engines/ACP |
| Co-tenancy (GPU leases) | Host-local; YK+Knative+sentinels graduate cross-project claims |
| `zndx.engine.v1` | Capability face; sentinel coordinates |

---

## 11. Implementation phasing

| Phase | Deliverable | Exit criterion |
|-------|-------------|----------------|
| **S0** | This document + `components/minifi-cpp` (+ yunikorn-core pin) | Spec + submodule on Signals trunk |
| **S1** | Lab **host** MiNiFi: probe one engine Status + OTel phases + C2 heartbeat | Heartbeat visible; phases correct without K8s |
| **S2** | C2 DESCRIBE inventory conventions; overwatch checklist | Multi-agent inventory via C2 |
| **S3a** | **Knative Serving** installed on system RKE2; `enable-scale-to-zero: true` (KPA) | Cluster ConfigMap verified |
| **S3b** | MiNiFi sentinel as **Knative Service** (single project) | Idle → **0 pods**; activate → scale from zero |
| **S3c** | **YuniKorn** queues `root.{project}`; sentinel pods scheduled via YK | Queue metrics show project isolation |
| **S4** | Gate engine `BeginWorkload` / GPU start on sentinel **activated + admitted** | No exclusive GPU work without claim |
| **S5** | Terminal phase → drop activation → Knative zero; orphan timeouts; YK app cleanup | No stuck replicas after complete |
| **S6** | Multi-project queues + preemption policy; retention/grace tuned; retire refuse-only leases for cross-project **windows** | Two projects fair-share sentinel claims under load |
| **S7** | Harden: OTel↔activator coupling, C2 UPDATE flows, docs for adopters’ Knative Service templates | Runbook + smoke in CI or lab script |

---

## 12. Summary

- **MiNiFi C++** = sentinel substrate (C2 + flows + metrics + overwatch).  
- **Knative Serving** = required K8s target for sentinel **scale-to-zero** ([KPA](https://knative.dev/docs/serving/autoscaling/scale-to-zero/)).  
- **YuniKorn** = admission/fairness **instance** for those sentinel pods on RKE2.  
- **OTel + C2** = activity truth so scale-to-zero does not kill `loading` work.  
- **Host engines** = execution + Kerberos/Ranger.  
- **signals-protocol** binds the lifecycle so every federation project participates the same way.
