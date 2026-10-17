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
honest): `thinking`, `instruct`, `complete`, `decisions`, `open-embedding`, `extract`. Future entries
(`sae`, `clt`, `vision`) are intentionally undeclared until a peer serves
them — requesting one yields NOMIX, which is the honest answer.

`decisions` is not an operating profile of the thinking model and it is
not a Complete capability. The capability token stays `decisions` (that
is the name a Metaflow workload requests). On Gaius it means the
deployment **thinking + dual-d3-flash**: Qwen3.8-27B stays on vLLM on the
heavy queue, and two `vllm-sr/d3-flash` replicas fill the light queue.
The fulfilment is `Engine/Decide` (d3_server `POST /v1/systemone`), which
returns a score for every allowed answer. Conjoining a method capability
with `decisions` is `#EP.00000020.NOMIX`. Calling `Complete` with
`decisions` is `#EP.00000022.NOTTEXT`. See §Structured decisions.

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

## Platform capabilities (2026-10-17)

The fourth class. A **platform** is an external vendor system an engine holds a
credential for. The vendor's own objects (Mistral's Agent, Conversation, Workflow,
Library, Document AI) are not model capabilities and not `zndx.agent.v1` agents.
Callers name vendor + product + verb. Vendor model strings stay inside the holder
and are not advertised on `Status.endpoints`. See [platform_grpc.md](platform_grpc.md).

## Structured decisions (additive, 2026-10-11)

A `decisions` request is a set of typed questions about one state. The
serving engine posts `{model, state, questions}` to d3_server
`POST /v1/systemone` on one `vllm-sr/d3-flash` replica and returns the
answers. The caller names the capability, not the model. Thinking stays
on the heavy queue for text; it is not the decision read. The decision
head is `readout.safetensors` under noncausal attention, not a next-token
logprob.

| type | criteria | answer |
|---|---|---|
| `choice` | object of option name to a description or `null` (1–255) | the runtime's answer object for that question |
| `noul` | `null`, or `true` / `false` descriptions | the runtime's answer object for that question |
| `score` | ordered list of 2–10 distinct level names | the runtime's answer object for that question |

Limits: 64 questions per request (client sanity bound). A question id is
a non-empty string. Instructions may be omitted; when set on a string
state the engine folds them into the state, because d3_server rejects a
top-level `instructions` field. A non-empty `cache_salt` is
`#EP.00000024.BADDECISION` (this runtime has no prefix-cache salt). HTTP
422 is the same code. Transport errors and HTTP 503 are
`#EP.00000023.NODECISION` and may try the other replica. An endpoint
that is not healthy is `#EP.00000016.NOTREADY` — the engine does not
start d3 inside `Decide` and does not fall back to thinking. One replica
answering while the other is down is the other copy of the same model.

`/v1/decisions` is the OpenAI-shaped name some gateways publish for a
different body. This federation's route is d3_server `/v1/systemone`.

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
