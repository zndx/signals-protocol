# zndx.engine.v1 — gRPC protocol specification

## Service

`zndx.engine.v1.Engine` — registered by every signals engine ADDITIONALLY, beside its
native project service. Insecure channels on the local fabric today; mTLS
engine-to-engine when the federation identity layer lands (per Gaius FEDERATION.md).

## Complete

Request an inference **capability**. The engine resolves the capability to a model it
manages (ensuring/loading it as needed) and serves the request internally; the caller
never addresses a model endpoint.

| field | semantics |
|---|---|
| `capability` | names an ability ("instruct", "referee"); empty = the engine's default |
| `prompt`, `system_prompt` | chat-style; system optional |
| `max_tokens` | generous defaults recommended — reasoning models trace verbosely; `finish_reason: "length"` signals truncation |
| `temperature` | sampling temperature |
| `json_schema` | optional JSON Schema (string) → engine-enforced structured output (e.g. vLLM guided-json); empty = unconstrained |

Response: `text` (the answer; reasoning inline when not separated), `model` (what
actually served), token counts, `latency_ms`, `reasoning_content` (separated
chain-of-thought when the model/parser splits it — retained, never dropped: thinking
traces are corpus value-adds in this federation), `finish_reason` (`stop` | `length`).

Errors surface as gRPC status codes; the engine stays up (INTERNAL for serving
failures, UNAVAILABLE while a capability is cold-loading if the engine chooses not to
block).

## Status

The federation-facing view: `project` (who answered), per-capability `Endpoint`
(capability, model, healthy, `gpu_ids` — lease visibility for co-tenancy planning),
`total_gpus`. Engine-private details (internal ports, log paths) are deliberately
absent.

## How the specification is consumed

**The `.proto` files are the specification** (same stance as
[OIP inference_grpc](https://github.com/kserve/open-inference-protocol/blob/main/specification/protocol/inference_grpc.md)).
Implementations **code-generate** stubs/clients from those protos. First-party
tooling (lattice CI, unit start probes, peer clients) **SHOULD** use generated
bindings — that is implementing the protocol properly.

| Consumer | Preferred mechanism |
|----------|---------------------|
| Engines / our scripts / libraries | **Codegen** from `proto/` (committed or build-step) |
| External operators / ad-hoc tooling | **gRPC server reflection** so bare `grpcurl` works |

Reflection does **not** replace the proto; it exposes the live service so tools
that do not ship our tree can still call `zndx.engine.v1.Engine` without local
descriptor flags.

## Server reflection (required on lattice ports)

Every engine that exposes `zndx.engine.v1.Engine` on the federation lattice
**MUST** enable [gRPC server reflection](https://github.com/grpc/grpc/blob/master/doc/server-reflection.md)
on that same listen port and advertise at least:

- `zndx.engine.v1.Engine`
- `grpc.reflection.v1alpha.ServerReflection` (or the reflection service name
  your stack registers)

| Requirement | Detail |
|-------------|--------|
| Python engines | depend on `grpcio-reflection` (or equivalent) and enable reflection at server start |
| External accept | bare `grpcurl -plaintext <host>:<port> zndx.engine.v1.Engine/Status` succeeds |
| First-party CI | generated-stub Status client **and** reflection check |

`grpcurl -proto …` against a live engine is fine for debugging, but first-party
automation should prefer **generated clients** from the same protos engines use.

## Co-tenancy conventions (informative)

Engines share one host and six GPUs today. The advisory lease layer is the shared lock
dir `/tmp/zndx-gpu-leases` (per-GPU-set `filelock` files with project-tagged owner
JSON); the authoritative check is each engine's nvidia-smi compute-process probe. A
foreign engine's resident capability is respected — capability requests are FORWARDED
over this protocol in preference to moving GPUs (model loads are expensive; requests
are cheap).

## OIP mapping (horizon)

| zndx.engine.v1 | Open Inference Protocol |
|---|---|
| `Complete` | no direct equivalent (chat-completion convenience); lowers to `ModelInfer` with a text tensor when bridging |
| `Status` | `ServerMetadata` + `ModelReady` (approximately) |
| capability names | model names resolved by the serving layer |
