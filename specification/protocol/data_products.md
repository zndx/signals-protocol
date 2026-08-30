# Federation data products

**Status:** v1 additive — warehouse contract for autonomous products on
shared Signals object storage (RustFS). Complements
[`tx_id.md`](tx_id.md) and [`engine_grpc.md`](engine_grpc.md) (`Remediate`).

A **Signals Data Product** is a named, agent-facing inventory row plus the
immutable objects that row describes. Peers (Gaius, Ægir, Atelier, …)
**maintain their own products**. They do **not** stand up a second warehouse,
copy product rows into pglite/Postgres, or keep a local Metaflow datastore
as SoR once they are on the platform.

## Split of ownership

| Owns | Who |
|------|-----|
| Product identity, object bytes, Metaflow flow, ACP meaning of *this* product | **The peer** |
| Warehouse tables (`details` / `tx` / `hx`), Polaris, Impala views, Kudu→Iceberg settle | **Signals** |
| Object plane (S3 API) | **Signals RustFS** — peers are tenants, not operators |
| Lineage facet | **Atlas OL** — join from `tx`, not a second inventory |

pglite / peer Postgres is **administrative** (AGE, Ranger, engine metadata).
It is not a product SoR.

## Identity

`product_id` is a dotted name the peer publishes and keeps stable:

```text
{peer}.{domain}.{name}
```

Examples: `gaius.cognition.outputs`, `gaius.prospects.corpus`,
`signals.metaflow.snapshots`. Do not reuse another peer's prefix.

## Warehouse shape (logical)

Signals exposes three logical names. Physical storage is tiered (Kudu
`*_tier0` then Iceberg `*_tier1` on RustFS); peers write **facts**, not
files into Kudu.

| Name | Shape | Role |
|------|--------|------|
| `tx` | `(product_id, tx_id, ts_ns, kind, summary, source, ce_type)` | One assertion epoch |
| `details` | `(e, a, v, t, op)` + `epoch_hour` | Fact log. `e` = `product_id`, `t` = `tx_id` |
| `hx_exchange` / `hx_reasoning` | keyed by `tx_id` | ACP utterance + quality / lineage / delta |

Current inventory = latest assert per `(e, a)` (retract via `op=false`).
Use **run-qualified** attributes (`run.{flow}/{run_id}.*`) when many
immutable versions must stay visible on one product.

`tx_id` / `t` **MUST** be RFC 9562 UUID version 7. Non-v7 ids are refused
and remediable on the **source** (`TX_ID_NOT_UUIDV7`). See [`tx_id.md`](tx_id.md).

## Object storage (RustFS)

Lab S3 API: `http://127.0.0.1:9010` (in-cluster `signals-rustfs:9010`).
Do not bind this port. Do not use `/tmp` or `~/.metaflow` as SoR.

| Prefix | Purpose |
|--------|---------|
| `s3://metaflow/metaflow/` | Platform Metaflow snapshots (code, data, deps) |
| `s3://signals-dataproducts/` | Warehouse Iceberg + peer product blobs |

A peer **MAY** keep a sub-prefix (`s3://signals-dataproducts/{peer}/…`) for
bytes that are not Metaflow CAS. The `details` row **MUST** point at the
RustFS URI. Local paths are not product facts.

Platform Metaflow profile (Signals tree: `config/metaflow/platform.json`):

```text
METAFLOW_DEFAULT_DATASTORE=s3
METAFLOW_DATASTORE_SYSROOT_S3=s3://metaflow/metaflow
METAFLOW_S3_ENDPOINT_URL=http://127.0.0.1:9010   # or in-cluster rustfs
```

`local` / unset / foreign S3 / non-RustFS endpoint is **fail-closed**.

## Events and methods

| Kind | Name |
|------|------|
| CloudEvent | `dev.signals.dataproduct.updated` |
| Review method | `data-product.history-review` (`event_received` → `reviewing` → `understood`) |
| Settle method | `data-product.tier-upkeep` — **Signals-owned**; peers do not DROP Kudu ranges |

On each create / update / maintain the peer (or Signals on the peer's
behalf) records a `tx`, asserts `details`, and leaves `hx_*` for the
agent. The review brief **MUST** cover:

1. **Quality** — is this product still fit for downstream agents?
2. **Lineage** — which run / table / model produced this `tx` (Atlas OL)?
3. **Delta** — what changed vs the prior `tx`?
4. **Nominal** — when the product is an operational flow (upkeep, curate,
   prospects refresh), is the method proceeding on its legal path?
   Holding is not failed and not done.

Do not spawn an agent to re-inventory. The warehouse is the inventory;
the agent **observes**.

## What a peer implements

1. Pin this repo. Codegen `zndx.engine.v1`. Handle `Remediate` for
   `TX_ID_NOT_UUIDV7` (`capability=reauthor`, remint v7, resubmit).
2. Point Metaflow at the **platform** profile (service `:30180`, RustFS
   `:9010`). Do not treat engine-local Tilt Metaflow as SoR on the
   shared cluster.
3. Choose a stable `product_id`. Seed JSON in Signals is bootstrap only.
4. At flow end (or CE `dev.signals.dataproduct.updated`): emit UUIDv7
   `tx_id`, map objects + run identity into `details` facts, write
   `hx_reasoning` quality / lineage / delta (and a nominal verdict when
   the work is a method).
5. Stamp YK with a **resource-class** queue (not `root.{project}`) and
   `federation.project`. See operations/sentinels.
6. Never `DELETE` warehouse rows. Never copy `details` / `hx` into
   pglite. Never implement `data-product.tier-upkeep` in the peer.

Reference implementation: Signals product `signals.metaflow.snapshots`
and `signals.ops.history` / `signals.ops.metaflow_store` in
[weathership/signals](https://github.com/weathership/signals).

## Discovery over the wire (additive, 2026-08-30)

`ServerQuery(kind=PRODUCTS)` returns `repeated ProductHint products` — the
products a peer **publishes**, with the Iceberg `table_identifier` in the
shared Polaris catalog, the `data_uri` on the Signals object plane, the
producing `flow`/`step`, and how `history` is retained. This is discovery
only: the warehouse (`details` / `tx` / `hx_reasoning`) remains the
inventory of record and the agent still observes there. A hint lets any
engine find and read a product (with full Iceberg snapshot history) without
first reading Signals. Empty list is honest.

Flow-/step-scoped products (e.g. Gaius `gaius.curation.cot_reasoning`, the
chain-of-thought traces of `ArticleCurationFlow.select_article`) keep the
flow/step/subject **as columns** of one physical table
(`hx.cot_reasoning`), partitioned by flow and month — so one product carries
every subject and every run, and the path
`{subject}/{step}/hx/{table}` is a projection, never a namespace.

Reference implementation: Gaius `gaius.curation.cot_reasoning`
(`src/gaius/hx/cot_reasoning.py`, `src/gaius/flows/article_curation/publish.py`,
`src/gaius/engine/s2s.py::declared_products`).
