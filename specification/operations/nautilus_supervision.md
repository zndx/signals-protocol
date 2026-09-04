# Nautilus supervision: a deterministic supervisor for federated stacks

**Status:** Binding design for stack supervision and probe-efficacy scoring  
**Wire face:** `zndx.supervision.v1` (this spec) + `zndx.engine.v1` (`Status`, `WatchWorkload`, Overwatch via `Complete`)  
**Bookkeeping:** the project's devenv PostgreSQL (`probe_forecasts`, `probe_resolutions`, `fsm_transitions`, `objective_verifications`) behind a local write-ahead buffer  
**Reference audit:** Gaius, 2026-09-02 — the findings that motivate every rule below

This document defines how a federated project declares its stack as a supervised
state machine, how an external deterministic supervisor (**Nautilus**) keeps it
running and **keeps score**, and how the engine's in-process reasoning capability
(**Overwatch**) is invoked without ever becoming part of the supervisor.

| Piece | Role |
|-------|------|
| **Nautilus** | Standalone, agent-free supervisor (Rust). Loads a project's `Supervisor` instance; observes every process through kind-specific adapters; enforces restart strategies within budgets; stamps positions; records and resolves forecasts; escalates what determinism cannot settle |
| **Overwatch** | A reasoning capability **inside the engine** (ACP + an independent model). Renders judge verdicts — objective rubric final calls, root-cause analysis, optimization — as part of normal operations. Reached by Nautilus through `WatchEscalations`; scored by Nautilus like any observer |
| **The instance** | A `Supervisor` textproto shipped by the project (`config/supervision/<project>.textproto`). The protocol is the grammar; the instance is the only place a project name, a flow, or a rubric appears |

---

## 1. Problem

The 2026-09-02 audit of Gaius found the self-observation stack conflated and thin:

- **No source of truth for the FSM.** The stack is assembled ad hoc from Metaflow,
  Airflow, pg_cron, in-engine daemons, vLLM endpoints and systemd units. Nothing
  declares the whole, so nothing supervises the whole.
- **Positions were never stamped.** The ledger's `flow_step`/`flow_run_id` axes were
  declared and dead; lifecycle was hardcoded. Forecasts were resolved by proposition
  *text* within a time window — a segment-correct claim ("the flow reached `end`") was
  Brier-penalized for a downstream content failure it had no causal relation to.
- **The supervisor lived inside the supervised process.** Arming and budgets were
  process memory, zeroed by every restart; the scorekeeper scored the process it ran in.
- **Out-of-process supervision was invisible.** Unit recycles were written to flat files
  no engine code reads — an unmodelled exogenous shock that silently converted probe
  misses into false "self-recovered" credit.
- **The scoring epoch rotated on every source commit**, erasing calibration with each
  deploy of unrelated code.

## 2. Roles: deterministic supervisor, reasoning engine

**Nautilus is deterministic and agent-free.** It hosts no model and no agent loop. Its
trigger set is a pure function of a snapshot (`evaluate_triggers(snapshot, cfg)`),
unit-testable without I/O. Arming is keyed on a **coarse signature** (trigger + scope +
bucketed state), never free text — the audit's storm (a trigger re-dispatching every
poll because its arming key embedded a changing duration) is a class this rule forbids.

**Overwatch is engine-native.** Judgment is a capability of the running engine, invoked
in the normal course of operations: objective rubric final calls, RCA on escalations,
optimization proposals. It runs on a model and transport **outside the local thinking
stack's failure domain** (the circularity a local judge cannot escape). Nautilus reaches
it only through `Escalation`; the engine answers by recording a judge forecast in the
ledger; Nautilus resolves that forecast like any other. The supervisor's own actuations
are promoted (observe → propose → act) only as their Brier record earns it, and the
judge scoring them is a separate process — the promotion is honest by construction.

## 3. The supervision tree

Every process in the instance names a `parent` and a `RestartStrategy` it applies to
its children (`ONE_FOR_ONE`, `ONE_FOR_ALL`, `REST_FOR_ONE`, or `NONE` = observe and
score only). The tree is explicit: the engine unit supervises the engine daemons and
serving endpoints; a task-queue process supervises the task classes it claims; a flow
supervises nothing but is observed through its store.

Rules:
- **Warmup is grace, nets are nets.** A process within `warmup_seconds` of (re)start is
  `STARTING`, never a miss. A process past its `Cadence.net_seconds` without observation
  is `MISSING`; one observed without progress is `STALLED`. Nautilus never kills on
  wall-clock alone — a process showing progress is left alone (progress-over-timeouts).
- **Budgets bound actuation.** `recycle_k` per `recycle_window_seconds`, with cooldown;
  exhaustion trips a **breaker** that is surfaced in `SupervisorStatus`, escalated, and
  never silently retried. The breaker clears on confirmed recovery.
- **Dependencies order restarts.** `depends_on` is honored on restart and start; a child
  is not restarted into a missing dependency.

## 4. Machines, phases, gates, horizons

A `Machine` decomposes a process (a flow, a task class) into an ordered list of
`Phase`s. Each phase spans named source steps and is **closed by a gate** — an explicit
predicate (`GateKind` + `params`) with a stated rationale. A gate without a concrete
threshold is not a gate.

**The phase-gate rule.** Forecasts made inside a phase are resolved against **that
phase's gate**, within that phase's `Horizon`. With `resolve_within_phase_only=true`
nothing outside the phase may resolve them. The gate's own verdict is then a claim the
*next* phase resolves — resolution is a hierarchy, not a flat window. This is what
stops "file downloaded" being penalized for a vLLM failure three phases later, while
still letting the terminal surface verdict resolve the terminal gate's claim.

**Observe through engines.** A source that lives on another host or belongs to
another project — a peer's Metaflow store, the central Airflow metadata, a foreign
pg_cron — is read **via the owning engine's `zndx.engine.v1` face**
(`Observation.via` names the engine process), never by opening the store's own port
from across a host boundary. Truly distributed deployments can promise one
protocol port per host and nothing more; the supervisor is designed to that
promise from day one. Engines answer observation reads deterministically
(`ServerQuery`-class hints), so this adds no reasoning to the supervisor's path.

**Sentinels stay; sensors complement them.** The MiNiFi sentinel pattern
(`minifi_sentinels.md`) is the proven representation of host work inside the
cluster — the YuniKorn claim, the C2 channel, OTel as activity truth. Adopting a
central scheduler (Airflow) reinforces that pattern rather than replacing it:
DAG tasks are admitted through the same queues, sensors wait on the facts
sentinels already emit, and asset events carry data-product lineage. Nautilus
observes both — sentinel claims and scheduler state — through the same engine
faces.

**Positions are reported, not scraped, at gate boundaries.** A process with
`reports_positions=true` MUST call `ReportPosition` on each phase transition. A
process that declares it and never reports is `UNSUPERVISED` — surfaced loudly, never
inferred around. Adapters (`SourceKind`) fill in what stores can show (Metaflow steps,
Airflow task instances, pg_cron runs) as `OBSERVED` positions; `INFERRED` positions are
derived (lifecycle from timestamps) and carry the least weight.

## 4b. Resource intents: what a phase occupies and what it must not lose

GPU capacity is a small integer of scheduler tokens (six on a 6-GPU host), and
contention for the leftover tokens between federated workloads is **intentional**:
it is what makes the architecture prove itself. The grammar therefore treats the
outcome of contention as a declared decision, not the arbiter's coin toss.

A `ResourceIntent` (on a `Phase`, or on a long-lived `Process` as a standing
intent) separates three numbers that admission-time protocols conflated:

- **occupancy** — tokens resident while the intent holds;
- **floor** — the guaranteed, non-preemptible part (≤ occupancy; `0` = fully
  preemptible). A resident GPU worker with floor 0 is a legitimate victim;
- **priority** — victim ordering within a leaf; higher survives preemption.

Emission is Nautilus's job, because Nautilus holds the position: the intent is
emitted on phase entry and a zero floor on phase exit — ahead of the claim, not at
the instant the token is already needed. The arbiter (the federation's scheduler,
Signals in this deployment) merges floors per leaf by maximum and records the
declared priority; the peer stamps the priority on its sentinel so scheduler
preemption evicts the low-priority holder. The wire form is
`zndx.scheduler.v1.WorkloadIntent.floor/priority`; a legacy intent without a floor
keeps its footprint-derived protection until it migrates.

Every emission is a forecast — `share APPLIED within N seconds` — resolved by the
arbiter's own record (`QueueShareRecord.applied_at_ns/apply_ms`), and
`GATE_KIND_SHARE_APPLIED` lets a phase gate on the arbiter having honoured its
floors. The arbiter is Brier-scored like every other observer; sustained divergence
between declared floors and applied config is an objective failure, not a warning
line. The worked example on the 6-GPU host: thinking holds a standing 4/4; a
prospects run declares extract 1/1 and protects embedding (light 1/1, priority
above the CLT probe's standing light 1/0); the probe yields for the run's duration
and resumes — a deferral on record, which is a result, not a failure.

Consensus among Nautilus instances is not a decision protocol. The arbiter is
single and authoritative; what the instances agree on is the vocabulary (this
grammar) and the arbitration policy (floors by declared priority, ties by
horizon), and each verifies the outcome in its own ledger. Agreement on rules with
independent verification of results degrades gracefully: a dark arbiter turns
every peer's forecasts inconclusive at once, which is itself the signal.

## 5. Objectives on the final surfaced result

An `Objective` names the artifact the user receives (`surfaced_result`) and the gates
that judge it. Every intermediate check — a flow reaching `end`, a task row clean, a
file written — is a **forecast that the user receives the intended result**, never a
gate of the objective. Where sufficiency requires judgment, a `JUDGE_RUBRIC` gate with
`judge_final_call=true` hands the verdict to Overwatch; the gate's `tier` is `GOLD`
(outside the trust boundary) and its resolutions propagate as gold. A local model's
rubric score is itself a forecast, resolved by the judge's call. Fail-closed: judge
unavailable is an error verdict, never a local fallback.

Objectives are paired by convention: a **mechanics** objective (the artifact flows) and
an **intent** objective (the *right* artifact flows — the schedule's promise). The
audit's canonical case: publishing green for weeks while the surface served
three-week-old content.

`Objective.resolves` (proposition patterns) is **transitional** — it exists so an
instance can be derived from a stack that does not yet report positions, and is
retired per objective as its machines start reporting.

## 6. Bookkeeping and the write-ahead rule

Nautilus records to the project's PostgreSQL: forecasts (`probe_forecasts`),
resolutions (`probe_resolutions`, append-only — corrections are new rows, latest
wins), transitions (`fsm_transitions`), verifications (`objective_verifications`).
**Recording never blocks on the database.** Every write goes first to a local
write-ahead buffer (SQLite or an append-only file under the supervisor's own state
directory) and is replayed when postgres answers. `SupervisorStatus.buffered_records`
exposes the backlog. The supervised stack's database being down is the moment the
supervisor's memory matters most; it is never the reason the supervisor forgets.

Every side-effecting verdict Nautilus renders — a recycle, a reset, a hold, a breaker
trip — is a forecast stamped with the `Position` it was rendered at and resolved within
its horizon. No recycle is invisible to scoring again.

## 7. Scoring

Brier per (observer × call site × momentum bucket) within the **epoch**
`spec_version + engine_build`, pinned in the instance. A source commit that changes
nothing about the stack does not reset calibration; a new spec version or engine build
does. Discount `α = (n/(n+K))·(1−Brier) + (K/(n+K))·prior` (K=2, prior 0.5); an
unresolved group is uncalibrated, never trusted. Drift is allowed: a single miss changes
a score, not behavior. No probe is auto-disabled; no threshold auto-tuned; divergence is
surfaced (triggers) and judged (Overwatch), and the judge is scored too.

## 8. Federation

One Nautilus binary; one instance per project; one ledger per project, written only by
that project's supervisor. Peers watch peers: a Nautilus subscribes to sibling
`Status` streams and records what it observes **in its own ledger** — engines never
write into another engine's ledger. Engine death is a peer's recorded observation and
escalation, not the dead engine's own.

## 9. Adoption and migration

1. **Serialize the current stack** into an instance — processes, cadences, flows and
   their step graphs, objectives with explicit gates. The gaius instance is derived from
   the 2026-09-02 audit inventory.
2. **Run Nautilus read-only** (`RESTART_STRATEGY_NONE` everywhere) beside existing
   watchdogs: observe, stamp, score, escalate. No actuation.
3. **Promote actuation per process** as the supervisor's own Brier record for that
   scope earns it (the α-gated promotion the doctrine always intended).
4. **Retire** the in-engine watcher and the flat-file watchdog scripts; the engine keeps
   Overwatch and gains the reporting obligation.

## 10. Non-goals

- Nautilus does not schedule work (that is the scheduler protocol and YuniKorn).
- Nautilus does not host models, agents, or judgment.
- The protocol does not know any project: no package, message, or enum names one.

## 11. The engine supervision stream and the Operations Backlog (2026-09-04)

**Topology.** A resident Nautilus runs beside each project's engine as a first-class
process and speaks gRPC to that engine only. The ENGINE serves `EngineSupervision.
Supervise(stream SupervisorMessage) returns (stream EngineEvent)`; Nautilus dials it.
Engine death is therefore the supervisor's own observation (the stream drops), the
engine needs no supervisor address and runs identically with none connected, and
"observe through engines" (§4) holds literally — Nautilus never reads the engine's
tables. The engine propagates supervisory messages across the federated mesh over
peer messaging; peers observe peers, and no engine writes another engine's ledger.

**Events and replay.** `EngineEvent` carries the SOURCE timestamp (a row's own time)
so replayed and live events sort and dedupe identically; `seq` is a per-session gap
detector. On `Subscribe{since}` the engine replays its own tables (clamped to its
window, ≥ 36 h — the deepest Backlog slot is 34 h) and then streams live. Nautilus
acks `through_seq` only once the record is durable in its own store: that is the
write-ahead rule of §6 made explicit, with the engine's tables as the log until the
supervisor has taken the record.

**Directives.** Nautilus never kills. It measures SILENCE since a task's last progress
signal (never age since claim) and sends `ReclaimOrphan`; the engine re-checks the
precondition (a heartbeat after the snapshot voids it; a live child refuses it) and
records the reset as a forecast — this subsumes any wall-clock task watchdog. An
`Escalation` on the stream IS the typed Overwatch consult (judge stays in the engine,
fail-closed). Every directive is answered by a `DirectiveResult` carrying the ledger
row the engine recorded; the engine dedups on `directive_id`.

**Expectations and the Backlog.** Each cadenced `Process` declares an `Expectation`
(category, horizon slot, channel). The Backlog clock is Fibonacci hours, F0..F9 =
0, 1, 1, 2, 3, 5, 8, 13, 21, 34: slot 0 is first admission into the Backlog — the
moment a workflow fails its natural admission window (time inside the window is the
arbiter working, not Backlog time); slots 1 and 2 (both 1 h) are the miss observed
then confirmed on consecutive hourly evaluations (the repeat separates a transient
from an open item); the deepest slot holding a miss is the escalation level and
selects the channel (3 Agenda Event, 5 Reminder, 6 and 8 briefing, 9 discussion).
Computed horizons round DOWN — surface early, never late. A healthy workflow fills
`ok` at slot 0 each hour and slots 1–9 stay empty; a slot flipping back to `ok`
resolves the item and its history stays. `Nautilus.Tick` is idempotent on the hour;
a systemd timer is the second hand while the resident runs. Intra-workflow mechanics
(admission nets, idle detectors) stay in seconds — two scales, one clock.

**Bookkeeping (§6 amended).** The supervisor's own records (backlog cells, trigger
firings, directives and their results, positions) are protobuf messages of this
package written first to a write-ahead journal (nisshi, an object-store log) and
drained into the project's tiered store; `SupervisorStatus.buffered_records` is the
journal lag. Recording never touches a filesystem path and never blocks on the
supervised stack. Each project's Backlog is a governed data product of the
federation's data-products history.
