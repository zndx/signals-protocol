# OIP stream hop

**Status:** additive v1 (2026-10-09). The messages are
[`proto/inference/open_inference_grpc.proto`](../../proto/inference/open_inference_grpc.proto).
This page is the frame vocabulary and the routing rules. Do not add a second
streaming RPC, and do not fork the tensor schema.

`ModelStreamInfer` is a bi-directional stream of `ModelInferRequest` and
`ModelInferResponse`. `Engine/Complete` stays the unary capability convenience
and lowers onto the same tensors. A product UI that wants tokens while the
model works holds this stream open on its **local** engine.

## Topology

```
client  --ModelStreamInfer-->  local engine  --ModelStreamInfer-->  hosting engine  --private HTTP-->  vLLM
```

- The client names a **capability** (`thinking`, `instruct`), never a model id
  and never a vLLM URL.
- The local engine hosts the capability only when it runs the model. Otherwise
  it forwards. `Status.endpoints` lists hosted capabilities only. Empty is
  honest. No silent substitution (`instruct` is never served as `thinking`).
- The hosting engine is the only process that dials vLLM. That URL is
  engine-private and does not appear in `Status`, `ServerQuery`, or a frame.
- vLLM's own token stream stops at the hosting engine. On the lattice every
  frame is a `ModelInferResponse`.

## Routing

This is a pairwise query, the Matrix federation Query shape: one request, one
response, no gossip and no room DAG
([Server-Server API](https://spec.matrix.org/latest/server-server-api/)).

Seeds, in order, skipping self: `directory_targets`, `SIGNALS_ENGINE_TARGET`,
then the peer contract. For capability `C`:

1. `Engine/Status` on the seed. Keep it when some `endpoints[]` entry has
   `capability == C` and `healthy`.
2. `ModelReady(C)` on that same `host:port`. A miss tries the next seed.
3. Open `ModelStreamInfer` to the advertised target. Do not rewrite a foreign
   host onto this machine.

No healthy peer is `FAILED_PRECONDITION` naming the targets asked. The
forwarder then emits one `progress` frame (`phase=routing`) and relays every
upstream frame unchanged.

`Announce` remains the TTL join hint. It is not the model hop and it is not
written to Atlas.

## Frames

All of these are BYTES tensors on the existing `ModelInferResponse`. A frame
carries one of them. The final frame of a generation carries `completion` and
the full `reasoning_content`.

| tensor | when |
|---|---|
| `progress` | JSON object `{"phase","detail"}`. `routing` is the forwarder. `generating` is the host, before the first token. |
| `reasoning_delta` | the next piece of the thinking trace |
| `text_delta` | the next piece of the answer |
| `completion` | the answer so far, on the final frame |
| `reasoning_content` | the trace so far, on the final frame. Readers also accept `reasoning`. |

`parameters.finish_reason` is set on the final frame:

| value | meaning |
|---|---|
| `stop` | the model finished |
| `length` | the token budget finished |
| `cancelled` | the client sent `control=cancel` |
| `superseded` | the client sent a new prompt while this generation was running |

`parameters.capability` echoes the request. `parameters.model` is the hosted
model id, not a routing key.

## Reverse direction

The client may send another `ModelInferRequest` on the same stream while
frames are arriving.

| request | effect |
|---|---|
| `parameters.control = "cancel"` | the host stops, then a final frame with `finish_reason=cancelled` |
| a new `prompt` or `messages` | the host stops the in-flight generation (`finish_reason=superseded`) and starts this one |

The forwarder does not interpret these. It copies the request upstream.

## Operating profile

The **hosting** engine aligns the profile from the capability
([capabilities.md](capabilities.md)). `thinking` is thinking on, effort
`xhigh`. `instruct` is thinking on, effort `low`. The forwarder does not set
effort and does not name the model.

## What is not this hop

`WatchWorkload` is the workload-profile stream. `zndx.agent.v1.Agents/Run` is
an agent session. Neither carries model tokens. Product HTTP is not a
substitute for `ModelStreamInfer`.
