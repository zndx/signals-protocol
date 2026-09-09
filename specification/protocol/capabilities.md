# Capability vocabulary — dual-constraint Complete (additive, 2026-08-30)

`Engine/Complete` may name a capability SET (`CompleteRequest.capabilities`):
a **conjunction** the serving engine must satisfy together — at most ONE
*method* capability plus at most ONE *model* capability. The engine plans the
fulfilment (method → optillm technique, model → a served endpoint) or fails
fast with `FAILED_PRECONDITION` `#EP.00000020.NOMIX`, whose message lists the
offered models × methods. An empty set is the legacy single `capability`
field, unchanged.

**The `.proto` files are the specification**; this page is the vocabulary
companion.

## Model capabilities

What `WorkloadOffer.capabilities` advertises today (per peer; empty list is
honest): `thinking`, `instruct`, `complete`, `open-embedding`, `extract`. Future entries
(`sae`, `clt`, `vision`) are intentionally undeclared until a peer serves
them — requesting one yields NOMIX, which is the honest answer.

A method-only conjunction (e.g. `["cot_reasoning"]`) defaults the model
capability to `thinking`.

## Operating profiles (additive, 2026-09-08)

"Capabilities, not models" also means **capabilities, not call parameters**.
One resident model serves several model capabilities; each is an **operating
profile** the serving engine **aligns** its call parameters to when a calling
engine names that capability. The caller never names a model or a kwarg.

| capability | model (today) | thinking | `reasoning_effort` | why |
|---|---|---|---|---|
| `thinking` | Qwen3.8-27B (Gaius) | on | `xhigh` (model default) | full trace; the corpus value-add |
| `instruct` | the same model | on | `low` | a brief trace with the same structure, less overhead |

`instruct` deliberately keeps thinking **on**: every fulfilment still yields a
model reasoning layer, so `hx_reasoning` rows stay structurally consistent
across capabilities while the trace cost drops. Effort levels are the model's
own vocabulary (Qwen3.8: `xhigh` \| `medium` \| `low`); a level the model does
not define is `INVALID_ARGUMENT` at the serving engine.

Rules:

- The serving engine **MUST** align `enable_thinking` / `reasoning_effort` (or
  the model's equivalents) to the requested capability's profile, and **MUST**
  report the profile it applied in `CompleteResponse.profile`.
- An engine advertises the profiles it serves in `WorkloadOffer.profiles`
  (`ServerQuery kind=WORKLOADS`); a capability without a profile is served at
  the model default — honest, never invented.
- An engine that does **not host** the model **MUST NOT** advertise the
  capability in `Status.endpoints` or `WORKLOADS`. It **forwards** `Complete`
  to a peer whose `Status` lists the capability healthy, or answers
  `FAILED_PRECONDITION` naming the peers it asked. **No silent fallback** to
  another capability (an `instruct` request is never quietly served as
  `thinking`, and vice versa).
- A request that also sets `json_schema` keeps the serving engine's
  structured-output rules (guided decoding may disable thinking); the applied
  profile reports what actually ran.

Reference: Gaius serves both profiles on its `thinking` endpoint; Ægir hosts
no model and forwards (`src/aegir/engine/forwarder.py`).

## Agent capabilities (2026-09-09)

The third class. An **agent** is the bounded assemblage — runtime, behaviours and tools,
workspace, and the model capability it thinks with — that takes an instruction and returns a
result with a trajectory. Agents are not `Complete` capabilities: they have their own service,
`zndx.agent.v1.Agents`, and are discovered by `ServerQuery kind=AGENTS` / `ListAgents`. See
[agent_grpc.md](agent_grpc.md). An agent's `model_capability` is one of the model capabilities
above, aligned to its operating profile by the serving engine.

## Method capabilities

Method capabilities name an optillm technique class. `WorkloadOffer.methods`
advertises the techniques a peer can actually serve. Synonyms:

| capability | technique |
|---|---|
| `cot_reasoning`, `reflection` | `cot_reflection` |
| `bon`, `moa`, `pvg`, `re2`, `self_consistency`, `rstar`, `plansearch` | themselves |

A request may also name the technique directly. Techniques a peer does not
advertise are NOMIX — a federated planner checks the models × methods mix
(`ServerQuery(kind=WORKLOADS)`) before routing.

## Reasoning layers

A fulfilment returns **every** reasoning layer it produced in
`CompleteResponse.reasoning` (model layer first when present):

- `layer="model"` — the model's native separated chain-of-thought
  (vLLM `reasoning_content`; e.g. Qwen's `<think>` block).
- `layer="method"` — the technique's scaffold (e.g. cot_reflection's verbatim
  `<thinking>…<reflection>…</reflection>…</thinking>` block).

Reasoning traces are corpus value-adds in this federation: retained, never
dropped. Single-call techniques (cot_reflection) are fulfilled by the engine
natively over its model backend so BOTH layers are captured; multi-call
techniques fulfilled through an optillm proxy may lose the model layer — the
serving engine then logs `#EP.00000021.METHODTRACE` and still returns the
method layer. Per-layer `tokens` is a best-effort share; `0` means unknown
(honest). `fulfilled_by` records the plan, e.g.
`cot_reflection@engine/Qwen3.8-27B@vllm:8081`.

## Restrictions (v1)

A method capability is one-shot text: combining it with `tools_json`,
`json_schema`, or `messages_json` is `INVALID_ARGUMENT` — the technique
scaffolds have no tool/multi-turn semantics.

Reference implementation: Gaius (`src/gaius/engine/capabilities.py` planner,
`src/gaius/engine/backends/backend_router.py` fulfilment,
`src/gaius/flows/lattice.py` client; consumer
`ArticleCurationFlow.select_article` → `hx.cot_reasoning`).
