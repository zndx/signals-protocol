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
                         │    │ resource-class     │
                         │    │ queues (not        │
                         │    │ root.{project})    │
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

**Hard rule:** A MiNiFi sentinel **is** the YuniKorn Application (1:1 with a
schedulable process). YK ends that Application; the Signals C2 server calls
`zndx.engine.v1.Engine/Yield` over gRPC; the owning engine ends the host
process. The sentinel is not a second catalog beside Applications. The
Application resource vector is the claim (`federation.zndx.org/gpu` for host
engine GPU work). CUDA stays on the host engine unless the work is an
in-cluster GPU pod (`nvidia.com/gpu` **and** the federation token).

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
| Multi-tenant queues / fair-share | Partial (namespace quotas) | **Yes** (resource-class queues, e.g. `root.internal.inference.extract`) |
| Cold start of thin agent | Activator + scale-from-zero | Ensures pod can be placed when demand exists |
| GPU workers | Sentinel is the **claim**; request `federation.zndx.org/gpu` | Host CUDA; in-cluster CUDA also requests `nvidia.com/gpu` |

**Composition (always the target):** YK **places and prioritizes** sentinel pods;
Knative **creates/removes** them based on activation/idle. Implement against
this pair from the start (manifests, labels, queue names, KPA annotations)—even
when early milestones only exercise a subset of the stack.

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

**Inactivity, not activity.** KPA scale-to-zero is the idle form of the same
YuniKorn Application (proxy sentinel). After an **extended inactivity**
window (stable-window + `scale-to-zero-grace-period`; 30s is a thin-agent
default, not a 27B vLLM default):

1. KPA takes the last replica to **0** (SIGTERM on the sentinel pod).
2. MiNiFi **C2 last-gasp** (`phase=preempted` / idle-stop) reaches the
   **federated engine that owns the vLLM process** (`Engine/Yield`).
3. That engine stops or checkpoints vLLM. The proxy Application
   **completes**. `federation.zndx.org/gpu` is released.
4. Queued work (e.g. 6×1.7B+CLT or 3×SAE) can admit on occupancy leaves.

Knative does **not** stop a host vLLM. The sentinel is the claim; C2 is the
wire to the engine that owns the process. If the pod hits 0 and the
Application stays Running, tokens stay claimed.

While **admitted and serving** (`loading` / `running`): `min-scale: "1"`
(or keep the activate HTTP). Do not STZ a model that is answering.

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
| `admitted` | YK admitted pod | Pod Running; may handshake engine |
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

Host engines **must not** start exclusive host work for a federated workload until:

1. Sentinel is **admitted** (YuniKorn Application) and **activated** (Knative revision ≥ 1), and  
2. C2 has a heartbeat for that `workload_id`, and  
3. Kerberos ticket available (`secretspec run` → `kinit`) when the work needs it.

YK preemption **or** idle STZ: last-gasp → C2 → `Engine/Yield` on the
federated engine that owns vLLM → process ends → Application completes.

### 5.4 Scale-to-zero (summary)

| Scale what | Mechanism |
|------------|-----------|
| **Sentinel pods** | **Knative KPA** → 0 after **extended inactivity** ([scale-to-zero](https://knative.dev/docs/serving/autoscaling/scale-to-zero/)) |
| **Host vLLM** | **Not** Knative. MiNiFi C2 last-gasp → `Engine/Yield` on the owning federated engine |
| **YK GPU claim** | Application **complete** after Yield — pod count 0 is not enough |

---

## 6. C2 as discrete command & control

One **Signals-owned C2 server** (HTTP) for all sentinels. It is not NiFi
and not each engine. Two wires:

| Wire | Direction | Protocol |
|------|-----------|----------|
| Heartbeat / last-gasp | sentinel → C2 | MiNiFi C2 HTTP |
| **Yield** | C2 → owning engine | `zndx.engine.v1.Engine/Yield` gRPC |

| C2 concept | Federation use |
|------------|----------------|
| Agent id | `federation.sentinel_id` |
| `workload_id` | Shared key with YK Application + Yield |
| Last-gasp / `phase=preempted` | YK ended the Application → C2 Yields |
| Missed heartbeat | `YIELD_REASON_ORPHAN` fallback |
| DESCRIBE / UPDATE | Later; not required for yield |

C2 does **not** replace `zndx.engine.v1` for inference. Engines do **not**
implement C2 REST. NiFi canvas is an optional later OTel/wiring fielding,
not the C2 server and not signals-ui.

---

## 7. YuniKorn as admission instance (required on target)

| Piece | Mapping |
|-------|---------|
| YK **queue** | Resource class (`root.internal.inference.*`, `root.internal.compute`, `root.platform`, `root.external.*`). Project is a label, not the parent. |
| YK-scheduled pod | Knative-created MiNiFi sentinel pod |
| Resources | Claim vector of the process (cpu/mem + `federation.zndx.org/gpu` when the work is GPU) |
| Preemption | YK ends the Application → C2 last-gasp → Engine/Yield |

YK and Knative are **co-equal** parts of the K8s target, not progressive
optional extras. Do not ship code paths that treat “plain Deployment without
Knative” or “host MiNiFi without YK queues” as supported configurations.

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
# Required K8s target: RKE2 + Knative Serving + YuniKorn
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
2. Run as **Knative Service** (YK-scheduled).  
3. Stay activated while phase ∈ {`loading`,`running`}.  
4. Handshake engine; export OTel.  
5. On terminal phase: stop activation → Knative scales to zero; release YK claim.  

### 9.4 Anti-patterns

| Don’t | Do |
|-------|-----|
| Support a permanent host-only / no-K8s product path | RKE2 + Knative + YK is the target; no dual architecture |
| Deploy sentinel as plain Deployment “for simplicity” | Knative Service + KPA scale-to-zero (Job/Pod ok for a one-shot proof) |
| Treat sentinels as a UI catalog beside Applications | Applications **is** the sentinel fleet (YK rows only) |
| Engine implements C2 REST | Signals C2 HTTP; engine implements **Yield** gRPC |
| Fold NiFi canvas into signals-ui | NiFi+canvas only when fielded on K8s; independent |
| Use HPA-only mode that blocks scale-to-zero | Use **KPA** (`enable-scale-to-zero: true`) |
| Scale to zero on scrape gaps during `loading` | Explicit phases + activation hold |
| Assume YK alone zeros idle pods | Knative owns pod scale-to-zero; YK owns multi-tenant admission |

---

## 10. Relation to other ops specs

| Spec | Relation |
|------|----------|
| [Kerberos + SecretSpec](./kerberos_and_secretspec.md) | Principals for engines/ACP |
| Co-tenancy (GPU leases) | Host-local; YK+Knative+sentinels graduate cross-project claims |
| `zndx.engine.v1` | Capability face; sentinel coordinates |

---

## 11. Implementation phasing

**Orientation:** Every phase is toward **RKE2 + Knative + YuniKorn + MiNiFi**.
Do not open a parallel “works without K8s” product track. If early work runs
MiNiFi on a developer host to harden flows before the image is schedulable,
treat it as **scaffolding that must die** once S2 lands—same flow config,
labels, phase schema, and C2 agent ids as the Knative Service will use.

| Phase | Deliverable | Exit criterion |
|-------|-------------|----------------|
| **S0** | Spec + `minifi-cpp` / `yunikorn-core` pins; **`zarf/federation`** package skeleton (`signals-federation`, independent of cybersec-dask) | Spec + package tree on trunk; cybersec adopts these standards later |
| **S1** | MiNiFi **flow + image** oriented to Knative (probe engine Status, OTel phases, C2 heartbeat). May run the **same** binary/flow under a throwaway local process only to debug agent logic | Flow + phase schema freeze; artifact is a **container image** destined for Knative, not a host package product |
| **S2** | Vendor charts/images into **`signals-federation`** Zarf package; deploy via Zarf (+ converge FSM) on RKE2; first MiNiFi sentinel **Knative Service** on a resource-class queue | Package create closed; idle → **0 pods** (KPA); YK shows app/queue |
| **S3** | C2 DESCRIBE inventory + overwatch against **live** Knative-scaled agents | Multi-project agents visible via C2 while scaled in |
| **S4** | Gate engine process start on sentinel **YK-admitted + C2 heartbeat**; C2 last-gasp → `Engine/Yield` | No exclusive host work without a YK Application; Yield ends the process |
| **S5** | Terminal phase → drop activation → Knative zero; orphan timeouts; YK cleanup | No stuck replicas after complete |
| **S6** | Multi-project fair-share + preemption; grace/retention tuned; host leases only for **intra-node** co-tenancy under YK windows | Two projects fair-share sentinel claims under load |
| **S7** | Harden activator↔OTel, C2 UPDATE, adopter Service templates + smoke | Runbook; CI or lab script against RKE2 |

**Explicitly rejected as milestones:** “ship host-only sentinels,” “YK later,”
“Knative later,” or dual code paths that skip admission/scale-to-zero forever.

---

## 12. Summary

- **MiNiFi C++** = sentinel substrate (C2 + flows + metrics + overwatch).  
- **Knative Serving** = K8s scale-to-zero for sentinel Services ([KPA](https://knative.dev/docs/serving/autoscaling/scale-to-zero/)).  
- **YuniKorn** = multi-tenant admission/fairness for those pods on RKE2.  
- **K8s-first** = product path; no permanent no-K8s fallback architecture.  
- **OTel + C2** = activity truth so scale-to-zero does not kill `loading` work.  
- **Host engines** = execution + Kerberos/Ranger.  
- **signals-protocol** binds the lifecycle so every federation project participates the same way.
