# zndx.agent.v1 — agents as federation capabilities

**Status:** v1 draft (2026-09-09), additive. The proto is the specification; this page is
the vocabulary and the rules. No engine serves it yet — adoption order at the end.

## Vocabulary

An **agent** is the bounded assemblage — a runtime, its behaviours and tools, a workspace,
and the model capability it thinks with — that receives an instruction across its boundary,
acts, and returns a result. The framing is Holland's *Signals and Boundaries* (an agent is
known by its boundary and the signals it admits and emits) and JADE/FIPA (the platform is
middleware for "aspects that are not peculiar of the agent internals": transport, encoding,
life-cycle). The runtime is the platform's business and the model is a `zndx.engine.v1`
capability under an operating profile. Neither is the noun. The agent is.

The LLM's thinking is not given a name of its own: an agent **has a model capability**
(`instruct`, `thinking`, …) stated as a field, served under the operating profile the serving
engine aligns to. (`reasoner` is taken — HermiT — and stays taken.)

| zndx.agent.v1 | JADE / FIPA | Holland | Harbor |
|---|---|---|---|
| `Agents/ListAgents`, `Engine/ServerQuery kind=AGENTS` | DF yellow pages | tags an agent advertises | `harbor agent list`, `agent schema` |
| `AgentOffer.agent_id` = `{project}/{name}@{version}` | AID | boundary identity | `--agent <name>` / `module:Class` / `acp:<x>` |
| `RunRequest.model_capability` | — (internals) | the mind, unnamed | `--model` |
| `RunRequest.kwargs_json` | agent arguments | — | `--ak` / `--agent-kwarg` |
| `RunRequest.mcp_servers` | Behaviours the caller lends | signals admitted | `BaseAgent(mcp_servers=…)` |
| `Agents/Run → stream AgentEvent` | ACL request → inform | signals crossing the boundary | `BaseAgent.run(instruction, environment, context)` |
| run = coordination Activity `agent_run` | AMS life-cycle | the boundary's lifetime | — |
| `RunDone.trajectory_uri` (ATIF) | the message log | the record of signals processed | the populated `AgentContext` / trajectory |

Three capability classes now exist: **model** (`zndx.engine.v1`, operating profiles),
**method** (optillm technique classes, `Complete.capabilities[]`), **agent** (this package).
A run is a session with tools and a deliverable, never a one-shot `Complete`.

## Service

`zndx.agent.v1.Agents` — registered **beside** `zndx.engine.v1.Engine` on the lattice port by
every engine that hosts at least one agent; reflection MUST list it there. Engines that host
none do not register it (a foreign stub gets `UNIMPLEMENTED` — honest).

| RPC | Role |
|---|---|
| `ListAgents` | the agents THIS engine hosts (`AgentOffer[]`); empty is honest |
| `Run` | run one agent on an instruction; the stream is every signal back out; ends with `done` |
| `Cancel` | stop a run before its horizon; releases its Activity; idempotent |
| `GetRun` | state / usage / trajectory handle for callers not holding the stream |

`Engine/ServerQuery kind=AGENTS` returns the light `AgentHint[]` (id, name, version,
transport kind, model capabilities, billing) so a launcher or planner can find agents across
the federation by walking `PEERS`; `ListAgents` is the full offer.

## Rules

- **Hosting.** The engine that hosts an agent runs it. The agent's own transport — an ACP
  stdio process, an SDK handle, a CLI — never crosses this boundary; the caller sees
  `AgentEvent`s, not plumbing. An engine that does not host an agent MUST NOT advertise it.
  No silent fallback from one agent to another (a `vibe-acp` request is never quietly run on
  `grok-build`).
- **The mind is a capability.** `RunRequest.model_capability` names a `zndx.engine.v1` model
  capability. The hosting engine resolves it — its own serving endpoint or a peer's, by
  `Status` — and the SERVING engine aligns the operating profile (`capabilities.md`). An agent
  that cannot think with the requested capability answers `FAILED_PRECONDITION`. Empty means
  the agent's default; an agent that brings its own model (a subscription runtime) advertises
  `model_capabilities` empty and `billing`.
- **Tools belong to the caller.** `mcp_servers[]` are the membranes the caller lends the agent
  (the agent can check, never fake). Cross-host tools are HTTP/SSE; an engine MAY refuse stdio
  tool servers from a foreign caller.
- **A run is an Activity.** On `Run` the hosting engine declares a coordination Activity of kind
  `agent_run` (`coordination_activities.md`): owner = `run_id`, claims = the driven model's YK
  leaf (0 GPU when the model is a peer's or the runtime brings its own), horizon =
  `budget.max_seconds` (engine default when 0), reason = `RunRequest.reason`. Renewed while
  running, released at `done`. Supervisors read runs through the engine (`Progress` events,
  `GetRun`), never the runtime.
- **The trajectory is a data product.** Every run writes an **ATIF** document (Agent
  Trajectory Interchange Format, as Harbor's agents populate their context) to the Signals
  object plane under `RunRequest.workspace_uri` (or the engine's `{peer}.agent.trajectories`
  prefix), records a `tx` for `tx_id`, and asserts one `hx_reasoning` row per turn carrying the
  model layer and the agent layer. `RunDone.trajectory_uri` / `trajectory_format` name it.
  Reasoning is retained — corpus value-add, never dropped.
- **Billing is declared.** `AgentOffer.billing` is `local`, `subscription`, or `token-metered`.
  A caller MAY refuse token-metered agents; an engine MUST NOT run a token-metered path under
  an agent advertised otherwise (the `keyframe.md` rule generalised).
- **Identity.** `run_id` and `tx_id` are RFC 9562 UUIDv7 (`tx_id.md`); a caller-minted `run_id`
  is idempotent (a retry attaches to the running stream). `AgentEvent.seq` is a per-run gap
  detector; `at_unix_ms` is the SOURCE time (supervision convention).
- **Budgets bound actuation.** `RunBudget` exhaustion ends the run with
  `RUN_BUDGET_EXHAUSTED` and a stop reason; it is a result, not a failure.

## Events

`Run` streams, in order: `accepted` (what the boundary resolved: agent_id, model and its
serving peer, the aligned profile, the Activity, the effective workspace), then any mix of
`thought` (a `ReasoningLayer` — `layer=model` for the mind's trace, `layer=agent` for the
runtime's scaffold), `message`, `tool_call` / `tool_result`, `deliverable`, `progress`, and
exactly one terminal `done`. A stream that ends without `done` is a transport failure the
caller treats as `RUN_FAILED`; `GetRun` is the record of truth.

## Harbor

Harbor's `--agent` is `RunRequest.agent`; `--model` is `model_capability`; `--ak` is
`kwargs_json`; `BaseAgent(mcp_servers=…)` is `mcp_servers[]`; `run(instruction, environment,
context)` is `Run(instruction, workspace_uri)`; the populated `AgentContext` is `RunDone`.
Two adapters follow (not protocol): a Harbor `BaseAgent` subclass that calls `Agents/Run` on
an engine (`--agent zndx_signals:FederationAgent --model instruct`), so Harbor can benchmark
any federation agent × model pair; and, on the hosting side, Harbor's installed agents and
its `acp:*` registry are `AGENT_TRANSPORT_CLI` / `AGENT_TRANSPORT_ACP` offers.

## What each engine brings (2026-09-09 inventory)

| Engine | Agents to offer | Transport | Thinks with |
|---|---|---|---|
| Ægir | `vibe-acp` (the Mistral fork with MCP tool injection — kvasir), `grok-build` | ACP | `instruct` / `thinking` via the engine; grok-build brings its own (subscription) |
| Metabase | `metabot` (today `acp:grok`, `acp:vibe` in Status — converge on this vocabulary) | ACP | `thinking` / `instruct` |
| Atelier | `claude-agent-sdk` (overwatch / terminal); the agent-mediated `referee` judge | SDK | referee = model capability `referee`; SDK brings Claude (subscription or metered — declare) |
| Hermes | `hermes` (the hsengine loop) | SDK | `thinking` / `instruct` |
| Gaius | `cognition`; its ACP client (grok-build escalation) as a caller | ENGINE / ACP | `thinking` |

**Follow-up in Hermes:** its `Status` capability `agent` is today an inference route to a
peer's `thinking` — the mind leaking out as if it were the agent. Under this vocabulary the
agent is `hermes` (an `AgentOffer`), and the inference route is `thinking`.

## Adoption order

1. This tree: proto + page (done), stubs regenerated by adopters.
2. Ægir: `Agents` servicer over the existing `BaseACPClient` (`vibe-acp`, `grok-build`),
   trajectory product, Activity per run.
3. Metabase: `metabot` on this face; retire the `acp:*` local names.
4. Atelier: `claude-agent-sdk`; Hermes: `hermes` (+ the `agent` capability follow-up).
5. Harbor adapters (separate package).

Additive within v1: new agents are new offers, new event kinds are new `oneof` fields.
