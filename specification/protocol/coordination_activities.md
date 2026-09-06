# Coordination Activities — inter-project intent with a lifetime

*Added 2026-09-06. Additive within `zndx.engine.v1`, `zndx.scheduler.v1`,
`zndx.supervision.v1`.*

## Why

A queue-share floor is a number with a validity window: it cannot say who or
why, and a peer never read another peer's. A direct `Engine/Yield` at a peer is
a poke with no record and no lifetime; on 2026-09-06 one started a supervisor
kill-loop at gaius. The federation needs a way for one project to say, with
purpose and structure, "this is what I am doing, for this long, and this is
what I need the rest of you to hold" — and for every project's supervisor to
read the same statement.

An **Activity** is that statement: inter-project intent with an owner, a
reason, a horizon, a state machine and a history. It is materialised as a run
of a Signals-owned DAG in the shared Signals Airflow, which supplies all of
those for free (plus a UI, sensors and Assets for ordering). The Airflow run
state is the truth every peer reads.

## Topology — two hops, never one

```
local process ──(local engine RPC)──▶ project ENGINE ──(zndx.scheduler.v1)──▶ Signals ENGINE ──(REST)──▶ Airflow
   interactive.py / flows /                hsengine :50651                        :50551             coord_activity DAG
   Nautilus resident                       gaius   :50051
```

- **Local processes speak only to their own local engine.** An interactive
  session, a flow, a Nautilus resident never open a connection to Signals or
  to Airflow. They ask their engine (`Engine/ServerQuery kind=ACTIVITIES`) and
  they hear about activities on `EngineSupervision` as `ActivityEvent`s.
- **Federated engines speak to the Signals engine** over
  `Scheduler/{Declare,Renew,Release,List,Watch}Activity`. The engine is the
  peer; it declares on behalf of its local processes and re-exposes what it
  learns.
- **Only the Signals engine speaks to Airflow.** No peer holds an Airflow URL
  or token.
- **Only Signals gRPC crosses hosts.** This category of coordination is critical
  functionality for Signals and for the protocol itself: it abstracts Airflow
  from the federated engines, so every project coordinates over the gRPC
  interface even when Signals + Airflow run on a remote host with nothing but
  the Signals gRPC port open between hosts. Airflow, the lease endpoint the
  sensor polls, the DAG ConfigMap and the host bridge are Signals-internal
  topology behind that port — never a requirement on a peer's network.

## Vocabulary (`zndx.engine.v1.Activity`)

| field | meaning |
|---|---|
| `activity_id` | stable identity across renewals (Signals-minted uuidv7) |
| `kind` | `interactive_session` \| `curation_window` \| `restart_window` \| `maintenance_pause` \| … |
| `peer` / `owner` | declaring project / the declarer within it (session id, process id) |
| `dag_id` / `run_id` | the Signals DAG (`coord_activity`) and the CURRENT run carrying it |
| `state` | `QUEUED` → `RUNNING` → `RELEASED` \| `EXPIRED` \| `FAILED`; `SUPERSEDED` when renewed |
| `horizon_ns` | the run's own timeout; renew before it or the intent expires |
| `claims[]` | YK leaves the activity occupies (informational — YuniKorn still admits) |
| `precludes[]` | YK leaf FQNs peers must NOT admit new work into while RUNNING |
| `postures{}` | `"<project>.endpoint.<alias>" → "hold-uptime"`: that project cedes the endpoint's intent until the activity ends — no restart chase, no unit recycle; other keys are additive vocabulary |
| `reason` / `note` | why it exists / Signals-written outcome |

Intent is **in force** only while `RUNNING`. `QUEUED` is declared-not-yet-
started; peers may prepare but must not yet cede.

## Lifecycle at Signals — Airflow OBSERVES the activity

Airflow's model for external work is a Sensor: the task is RUNNING because the
process is observed alive, not because someone said so. Signals therefore holds
a **lease** per activity (heartbeat, horizon, TTL 180 s, released flag) and the
run's `hold` task is a deferrable **`SignalsActivitySensor`** whose trigger polls
that lease (through the Signals engine's control HTTP, bridged into the
cluster) every few seconds. Nobody patches a run's state.

| RPC / event | Signals lease | Airflow |
|---|---|---|
| `DeclareActivity` | create lease (heartbeat = now) | trigger ONE run `act-<activity_id>` of the kind's DAG (`coord_interactive_session` → pool `agent_rtc`, 1 slot, deferred-inclusive; other kinds → `coord_activity`); `conf` = declaration + `lease_url`; `declare` emits Asset `zndx.coord.activity` |
| `RenewActivity` | **heartbeat**: heartbeat = now, horizon = request; no Airflow call, no new run | `hold` keeps deferring |
| `ReleaseActivity` | released = true, outcome, ended | the trigger observes it → `hold` completes with outcome `released` → `close` emits Asset `zndx.coord.activity.ended` → run `success` → state `RELEASED` |
| heartbeats stop for the TTL, or the horizon passes | lease **lapsed** | the trigger observes it → outcome `lapsed` → run `success` → state `EXPIRED` |
| Signals unreachable from the trigger | unknown | the trigger keeps waiting (a dark Signals is not a lapse); the sensor `timeout` (24 h) is the outer net |
| run fails | — | `FAILED` — intent NOT in force |
| `ListActivities` | join runs ↔ leases by `activity_id` | run state is the truth; `RELEASED` vs `EXPIRED` read from the lease |
| `WatchActivities` | | server stream; every event is the FULL in-force set plus recently-ended (≥ 5 min window); emitted on change and at least every 60 s — a silent stream is a dead stream; a client REPLACES its view on every event, absence = ended |

The declarer heartbeats well under the TTL (Hermes: every 60 s) with the horizon
as the session's outer bound. `ACTIVITY_SUPERSEDED` remains in the vocabulary
for a scheduler that renews by re-running; Signals does not.

`DeclareActivityRequest.request_id` is idempotent: a retry returns the same
activity. Unknown peers are refused (`accepted=false`, guru in `error`).
Airflow unreachable is an error, never a local substitute (fail-fast).

## What peers do with an Activity

- **Cede.** For every RUNNING activity with posture `<me>.endpoint.<alias>:
  hold-uptime`, the engine marks the endpoint's intent *ceded* (owner,
  activity, horizon): the orchestrator does not restart it, the workload
  profile carries no unmet intent for it, status reads `CEDED`. When the
  activity ends the desired set is restored.
- **Admit around it.** Admission of a local class whose YK leaf is in a
  RUNNING activity's `precludes[]` is deferred (a deferral, not a failure).
  Leaves not precluded keep admitting — light and medium sentinels continue
  unless the activity says otherwise.
- **Tell the supervisor.** Every observed change becomes an `ActivityEvent`
  on `EngineSupervision` (`transition`: observed | started | renewed |
  released | expired | failed | declared; `ceded[]`: this engine's process
  ids ceded to it). `SOURCE_KIND_AIRFLOW_METADATA` is observed through this
  path only. A process ceded to an activity is `ceded`, never missed; an
  activity past its horizon without release is a Backlog item
  (`EXPECTATION_CATEGORY_COORDINATION`).

## Non-goals (for now)

Declaring project schedules as Airflow DAGs (`ScheduleHint` → DAG factory),
Asset-driven cross-project ordering, and a token resource key for
`token-metered` leaves follow the same door later.
