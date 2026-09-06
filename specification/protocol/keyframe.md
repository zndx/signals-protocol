# Render / keyframe (additive v1)

**Status:** design — not on the wire until the proto lands.

Hermes (and later any peer) asks **Gaius** to render a conversation graph
as a LuxCore still. LuxCore stays on Gaius. The federation face is
`zndx.engine.v1`, not a Gaius HTTP one-off and not WebRTC.

## Split

| Owns | Who |
|------|-----|
| Graph snapshot (nodes, edges, optional embedding refs) | Caller (Hermes plugin) |
| PATHOCL / cards machinery / GPU eviction | **Gaius** |
| Object bytes | **Signals RustFS** (`data_uri`) |
| Warehouse row | Signals (`tx` / `details`) when the still is a product fact |
| Forward-sim (P-frames) | **Caller engine** (Hermes `hsengine`). Asks YuniKorn for **one 4090**, `RESOURCE_CLASS_LIGHT`. Gaius does not interpolate. |

Engines that are not a renderer return `UNIMPLEMENTED` (honest).

## Why not bytes-as-SoR

`data_products.md`: object plane is RustFS; the protocol carries identity
and URI. A gRPC still as the only copy would skip the warehouse and blow
message size. The RPC **may** include a small `preview_jpeg` (cap
256 KiB) so the morph can start without a second S3 GET. The SoR is
`data_uri`.

## RPC (proposed)

```
rpc Render(RenderRequest) returns (RenderResponse);
```

`RenderRequest`:

| field | meaning |
|-------|---------|
| `capability` | `viz` (Gaius chooses the card variant / camera) |
| `graph_json` | UTF-8 conversation graph (nodes, edges, layout hints) |
| `product_id` | optional existing `gaius.viz.keyframes` id to supersede |
| `tx_id` | UUIDv7 for the request epoch; Gaius mints if empty |
| `max_preview_bytes` | 0 = no preview on the wire |

`RenderResponse`:

| field | meaning |
|-------|---------|
| `accepted` | false + `error` guru when refused (empty graph, no GPU, …) |
| `product_id` | `gaius.viz.keyframes` (or the peer’s viz product) |
| `data_uri` | `s3://signals-dataproducts/gaius/viz/keyframes/{tx_id}.webp` |
| `content_type` | `image/webp` (or jpeg) |
| `width` / `height` | pixels of the **full** still |
| `kappa_json` | optional curvature overlay (edge κ); morph must not recompute Ricci |
| `preview_jpeg` | optional ≤256 KiB; empty if `max_preview_bytes=0` |
| `tx_id` | UUIDv7 of the still |

`Complete` is the wrong shape (inference). `Yield` is sentinel teardown.
Do not overload `ServerQuery`.

Forward-simulation is **not** Gaius. After `Render`, the caller streams
the still (`data_uri`) onto its own object plane if it wants a replica,
then runs interpolator code **in its engine**. GPU: declare a LIGHT
workload (one 4090); YuniKorn admits it (`WatchWorkload` / queue share).
The dashboard only views the resulting frames. The interpolator does not
run in browser JS and does not import pyluxcore.

A caller MAY instead animate the still with an **off-lattice** image-to-video
service (Hermes: xAI Grok Imagine via subscription OAuth, not an API key).
That clip is a replica on the caller’s object plane, not a Gaius product.
`Render` still returns only the still handle.

## Hermes plugin

`signals-listen` `plugin_api` is a gRPC **client** of Gaius `:50051`
`Engine/Render` (same stub as `signals-oip`). It does not import pyluxcore.
The dashboard JS is a **viewer** of Hermes-engine forward-sim frames
(via plugin HTTP/WS). It does not morph.

## WebRTC

Not this RPC. Duplex audio/video is engine-local later. This message is
a **still**, at turn/topic cadence.
