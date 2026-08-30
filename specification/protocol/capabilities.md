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
honest): `thinking`, `complete`, `open-embedding`, `extract`. Future entries
(`sae`, `clt`, `vision`) are intentionally undeclared until a peer serves
them — requesting one yields NOMIX, which is the honest answer.

A method-only conjunction (e.g. `["cot_reasoning"]`) defaults the model
capability to `thinking`.

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
