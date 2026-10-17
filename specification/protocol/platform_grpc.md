# zndx.platform.v1 — external platforms

**Status:** v1 draft (2026-10-17), additive. The proto is the specification; this page is
the vocabulary and the rules.
**Precedent:** [flight_data_plane.md](flight_data_plane.md) adopts an external wire and
defines only the lattice rules around it. [keyframe.md](keyframe.md) already forbids a
token-metered key on an off-lattice path that was declared subscription.
**Proto:** `proto/zndx/platform/v1/platform.proto`. `ServerQuery` kind
`PLATFORMS` = **21** and `ServerQueryResponse.platforms` = **23** are allocated.
Kind 19 is the `fmp_hint` hole; 20 is `ASPECTS`.

Reviewed against the public Mistral docs on 2026-10-17. Vendor behavior below is that
reading. A preview API on their side does not rename a lattice product.

## The fourth class

| Class | Package | What the caller names |
|---|---|---|
| Model | `zndx.engine.v1` | `thinking`, `instruct`, … — an operating profile |
| Method | `Complete.capabilities[]` | an optillm technique |
| Agent | `zndx.agent.v1` | an assemblage **this lattice hosts** ([agent_grpc.md](agent_grpc.md)) |
| Platform | this package | a vendor product behind a credential an engine holds |

A **platform** is an external system. The engine that holds the credential is the
**holder**. The platform's own objects keep their own ids. The holder is who peers
call. The vendor is who runs the object.

This class exists because a vendor will reuse our nouns. Mistral's Agent is a saved
preset (`model` + `instructions` + `tools` + `completion_args`). A Conversation is a
stored history and can be started with no Agent. A Handoff is one preset invoking
another inside that history. A Workflow is a durable orchestrator (Temporal, public
preview) whose workers run in the holder's environment. None of those is a
`zndx.agent.v1` agent. Stuffing them into `AgentOffer` makes `{project}/{name}@{version}`
collide with `ag_…` ids, with workflow names, and with `peer_target_for_project`'s
walk of agent names.

## Service

`zndx.platform.v1.Platforms` — registered **beside** `Engine` and `Agents` on the
lattice port by every engine that holds at least one platform credential. Reflection
MUST list it there. An engine that holds none does not register it
(`UNIMPLEMENTED` is honest).

| RPC | Role |
|---|---|
| `ListPlatforms` | the platforms THIS holder advertises (`PlatformOffer[]`). Empty is honest. |
| `Describe` | one platform's product catalog: which verbs are live, and the foreign objects the holder is willing to name. |
| `Call` | one verb. Server stream of `PlatformEvent`, exactly one terminal `done`. |
| `Cancel` | stop a call before its horizon. Idempotent. |
| `GetCall` | state for a supervisor that does not hold the stream. |

`Engine/ServerQuery kind=PLATFORMS` (reserved **21**) returns `PlatformHint[]`
(field **23**): vendor, holder project, account label, billing, product kinds.
`ListPlatforms` is the full offer. A launcher walks `PEERS` the same way it walks
`AGENTS`.

## Identity

```text
platform_id = "{holder}/{vendor}@{account}"
```

`holder` is the engine project (`hermes`). `vendor` is a stable lattice name
(`mistral`). `account` is a non-secret label the holder chooses for one credential
(a workspace name). It is not the key, not a key id, and not an email.

`foreign_id` is the vendor's id, opaque, unmodified: `ag_…`, `conv_…`, `handoff_…`,
a workflow name, a library id, a connector name or UUID, an execution id. The
lattice does not mint `{project}/{name}@{version}` for these. It does not parse them.

`call_id` and `tx_id` are RFC 9562 UUIDv7 ([tx_id.md](tx_id.md)). A caller-minted
`call_id` is idempotent: a retry attaches to the open call.

## Rules

- **Holding.** The holder runs the call and speaks the vendor wire. Peers see
  `PlatformEvent`s. Vendor SDK handles, worker processes, orchestrator addresses,
  and HTTP bodies do not cross this boundary. A holder that cannot fulfil a product
  MUST NOT advertise it. No silent fallback from one vendor, account, or product
  to another.
- **The credential stays on the holder.** Offers, events, `input_json`, logs, and
  `Status` carry no key, token, or `Authorization` header. The holder reads its own
  secret store. The lattice does not proxy the vendor's secrets manager.
- **Capabilities, not their model names.** A caller names `vendor` + `product` +
  `verb`. A vendor model string (`mistral-medium-latest`, `mistral-ocr-latest`)
  stays inside the holder. It is not a `zndx.engine.v1` capability and it is not
  written on `Status.endpoints`. An engine that does not host the weights does not
  advertise the capability ([capabilities.md](capabilities.md)).
- **Not an agent.** A foreign platform object MUST NOT appear as an `AgentOffer`,
  an `AgentHint`, or a `ServerQuery kind=AGENTS` row. `peer_target_for_project`
  resolves lattice agent names. A vendor agent named `document`, or a vendor named
  like a peer project, would steal that walk.
- **Billing is the account's real bill.** `local`, `subscription`, or
  `token-metered`, same enum as agents. A pay-per-call API key is
  `token-metered`. A holder MUST NOT label that key `subscription`. A caller MAY
  refuse `token-metered` (the [keyframe.md](keyframe.md) rule). `Describe` and
  `Call` on a refused bill return `FAILED_PRECONDITION` before any vendor call.
- **A call is an Activity.** Kind `platform_call`. Owner = `call_id`. Horizon =
  `budget.max_seconds` (holder default when 0). Reason = `{product}.{verb}`.
  Claims are what the holder actually spends. A call that only waits on the
  vendor's cloud claims **0 GPU** — it does not borrow `extract`, `light`, or
  `heavy`. A local worker that needs a leaf declares that leaf. Released at
  `done`. Supervisors read `GetCall` and `progress` events, never the vendor.
- **Their store is not our trajectory.** Default: the stream is the record, and
  the holder does not copy vendor state into SessionDB, the wiki, or
  `hx_reasoning`. `store=false` on a conversation asks the vendor not to retain
  the history (Mistral's flag). `workspace_uri` set means the holder ALSO writes
  a lattice data product under that URI and returns `trajectory_uri`. Unset
  means no lattice write. A foreign handle (`mistral:conv_…`,
  `mistral:exec_…`) may be `trajectory_uri` with `trajectory_format=foreign`.
  That handle is not ATIF.
- **Projection is labeled.** When the holder maps a vendor entry onto an event,
  the event carries `vendor` and `foreign_id`. A handoff is a `handoff` event.
  It is not a nested `Agents/Run` and it is not a second lattice agent.
- **Inline size.** `input_json` and a single event's `json` are at most 1 MiB.
  Larger bytes are a data-product URI the holder fetches. Mistral's workflow
  activity inline limit is 2MB, after which their SDK offloads to the holder's
  blob. The holder obeys that limit. The lattice does not raise it.
- **Preview vendors.** Mistral marks Workflows and Connectors as public preview.
  Product kinds on this face stay stable. `input_json` / result JSON is the
  vendor's shape, validated by the holder. A preview break is the holder's
  `FAILED_PRECONDITION` or `done.error`, not a new protocol version.

## Call

`Call` streams, in order: `accepted` (platform_id, product, verb, foreign_id if
already known, activity_id, billing), then any mix of `progress`, `entry`,
`handoff`, `needs_input`, `deliverable`, and exactly one `done`. A stream that
ends without `done` is a transport failure. `GetCall` is the record.

`needs_input` is terminal for v1 (`done.state = NEEDS_INPUT` follows it). The
vendor asked the holder to run a client function or a client handoff. A later
`Call` with verb `reply`, the same `foreign_id`, and the function result
continues it. v1 has no bidirectional reply on the same stream. Their HTTP API
is request/response. The lattice matches that.

`done.state` is `COMPLETED`, `FAILED`, `CANCELLED`, `NEEDS_INPUT`, or
`BUDGET_EXHAUSTED`. Budget exhaustion is a result. Vendor HTTP status stays in
`done.error` as text the holder writes. It must not include the key or a
request header.

## Mistral binding

Public docs, read 2026-10-17:

- [Agents introduction](https://docs.mistral.ai/studio/agents/introduction)
- [Agents and Conversations](https://docs.mistral.ai/studio/agents/agents-api)
- [Handoffs](https://docs.mistral.ai/studio/agents/handoffs)
- [Tools](https://docs.mistral.ai/agents/tools)
- [Connectors](https://docs.mistral.ai/agents/mcp)
- [Libraries](https://docs.mistral.ai/studio/search/libraries)
- [Workflows overview](https://docs.mistral.ai/studio/workflows/getting-started/overview)
- [Workflow core concepts](https://docs.mistral.ai/studio/workflows/getting-started/core_concepts)
- [Document annotations](https://docs.mistral.ai/capabilities/OCR/annotations)
- [Document QnA](https://docs.mistral.ai/studio-api/document-processing/document_qna)

Their nouns, and the product they map to:

| Their object | What the docs say it is | `foreign_id` | Product |
|---|---|---|---|
| Agent | A preset: `model`, `name`, `description`, `instructions`, `tools`, `completion_args`, optional `handoffs`. Creating one does not run it. | `ag_…` | `agent` |
| Conversation | A history of entries. Start with `agent_id` **or** with `model` and no agent. The two APIs are independent. Owned by the API key that created it. Their docs say each append returns a new conversation id. | `conv_…` | `conversation` |
| Entry | `message.output`, `tool.execution`, `agent.handoff`. | entry id | an event on the conversation call, not a product |
| Handoff | One agent calls another mid-conversation. `handoff_execution=server` (default) runs the chain on their cloud. `client` returns control to the caller. Their prose also calls a handoff graph an "agentic workflow." | `handoff_…` | not a product. An event. |
| Built-in tool | `web_search`, `web_search_premium`, `code_interpreter`, `image_generation`, `document_library`. Execution is in their environment. `web_search` and `code_interpreter` are on Agents and Conversations, not on Chat Completions. | tool type | not a product. A tool type listed on `agent.get`. |
| Function tool | A function the **client** executes. | function name | `needs_input`, then `conversation.reply` |
| Connector | A registered MCP server, public preview. Tools run server-side. A key is scoped to workspace-shared connectors or to private plus shared. Human confirmation and direct tool call exist. | name or UUID | `connector` (list only in v1) |
| Library | A persistent document store. `document_library` on an agent takes `library_ids`. | library id | `library` |
| Workflow | Durable orchestrator they host. Workers run in **our** environment and connect outbound. Activities are the side effects. Public preview. Python ≥ 3.12. Inline activity payload 2MB, then offload to the holder's blob. | workflow name; execution id on a run | `workflow` |
| Document AI | OCR, `document_annotation`, `bbox_annotation`, document QnA. A `document_url` must be public. This is not an Agent. | none, or the annotation id they return | `document` |

Batch, moderation, and their secrets manager are real Studio products. v1 does
not open verbs for them. Moderation rides on a conversation when the preset has
guardrails. Their secrets manager is not a lattice secret store.

Workflow, Agent, and Handoff stay three things. A planner MUST NOT satisfy a
`workflow.execute` with a handoff, or an `agent.list` with `ListAgents`.

### Verbs the holder implements

`input_json` is an object. Unknown fields are `INVALID_ARGUMENT`. The holder
adds the credential. The caller does not pass `model`.

| Product | Verb | Required input | Result `deliverable.json` |
|---|---|---|---|
| `document` | `analyze` | `url` (public `https` arXiv, bioRxiv, or medRxiv). Optional `annotation_schema` (JSON Schema). Optional `kind` (`arxiv`, `biorxiv`, `medrxiv`); the holder classifies when it is empty. | `url` (the PDF URL the vendor fetched), `title`, `summary`, `claims`, `method`, `limits`, `text`, `model` as they spelled it. No wiki write unless `workspace_uri` is set. |
| `workflow` | `execute` | `name` (one token, `[A-Za-z0-9_.:-]{1,128}`). `input` object. `wait` bool, default true. | `execution_id`, `workflow`, `status`. `result` when `wait` and the run completed. `wait: false` ends after the execution id is known. |
| `workflow` | `get` | `foreign_id` = execution id | `status`, `result` when completed |
| `workflow` | `list` | optional `tags` (AND, their filter) | the names THIS holder can run. A name with no live worker and no managed deployment is omitted. |
| `agent` | `list` | — | `{foreign_id, name, description, tool_types}[]`. No model string. No instructions body. |
| `agent` | `get` | `foreign_id` | the list row plus `handoff_ids` (other `ag_…`) and `tool_types` |
| `conversation` | `start` | `agent_foreign_id`, `inputs` (string or message list), `handoff_execution` (`server` default, or `client`), `store` (default true) | `foreign_id`, then one `entry` per vendor entry |
| `conversation` | `append` | `foreign_id`, `inputs` | the new `foreign_id` their API returned, then entries |
| `conversation` | `reply` | `foreign_id`, `function_name`, `result` | continues a call that ended `NEEDS_INPUT` |
| `library` | `list` | — | `{foreign_id, name}[]` |
| `library` | `describe` | `foreign_id` | name and the document ids the vendor lists. Bytes stay on their side. |
| `connector` | `list` | — | `{foreign_id, name, visibility}[]` where the vendor sent visibility. v1 does not invoke a connector except as a tool inside a conversation the preset already has. |

`document.analyze` rejects any URL the holder cannot classify as arXiv, bioRxiv,
or medRxiv. Other PDFs are `FAILED_PRECONDITION`, not a silent fetch. The PDF
URL sent to the vendor keeps the preprint version. An unversioned bioRxiv or
medRxiv URL only redirects, and their fetcher will not follow it.

`workflow.execute` sends `input` as the vendor's workflow input. For a paper
workflow the holder rewrites an abstract URL to the public versioned PDF before
the POST. The credential is the header only.

`conversation.start` with a raw `model` and no `agent_foreign_id` is
`INVALID_ARGUMENT` on this face. Their API allows it. The lattice does not,
because that is a model name. A holder that wants a model-only conversation
exposes a preset (`ag_…`) it created for that purpose.

`handoff_execution=server`: the holder waits, projects each `agent.handoff` and
`tool.execution`, claims 0 GPU. `client`: the first handoff or function tool
becomes `needs_input` and the call ends `NEEDS_INPUT`. The holder does not
invent the next agent.

### Events projected from a conversation

| Vendor entry | Platform event |
|---|---|
| `message.output` | `entry` role `assistant`, text, `foreign_id` = message id, `agent_foreign_id` set |
| `tool.execution` | `entry` role `tool`, name = tool name, `foreign_id` = tool execution id |
| `agent.handoff` | `handoff` with `agent_foreign_id` and `agent_name` |

`fulfilled_by` on a tool entry is `{vendor}/{tool type}` (`mistral/web_search`),
so a trajectory does not read as a lattice tool the caller lent.

## What Hermes serves today

As of 2026-10-17 Hermes advertises two `zndx.agent.v1` offers, `mistral-document`
and `mistral-workflow`, billing recorded as subscription, empty
`model_capabilities`. They are facades over `document.analyze` and
`workflow.execute`. They are not Mistral Agents.

When the holder serves `Platforms` for `vendor=mistral`:

1. `ListPlatforms` includes that offer.
2. The two facade `AgentOffer`s are removed. No further `hermes/mistral-*` agent
   names are added.
3. The account's real billing is what `PlatformOffer.billing` says. A
   pay-per-token key is `token-metered`.

Until that cut, the facades stay. New Mistral presets (`ag_…`) are not added
beside them.

## Adoption order

1. This tree: this page + `platform.proto`. Implementation PR adds
   `SERVER_QUERY_KIND_PLATFORMS = 21` and `repeated PlatformHint platforms = 23`,
   then adopters regenerate stubs.
2. Hermes: `Platforms` servicer. Move `mistral-document` / `mistral-workflow`
   off `Agents`. `document.analyze` and `workflow.execute` / `get` first.
   `agent.list` / `conversation.start` next. Workflow advertise only when a
   worker or a managed deployment exists.
3. Any other holder (a second Mistral account, or another vendor) is a new
   `PlatformOffer` on the engine that holds the key. The product table for a
   new vendor is a new section of this page, not a new package.

Additive within v1: new products and verbs are new string values the holder
documents here. New event kinds are new `oneof` fields. A breaking vendor
change does not renumber the lattice.
