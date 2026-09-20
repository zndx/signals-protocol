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

## Aspects (SHACL Core, additive 2026-09-20)

A Data Product is a **concrete instance**. An **Aspect** is a reusable
semantic contract — a [SHACL Core](https://www.w3.org/TR/shacl/)
`NodeShape` — not a product:

\[
A \cap P = \varnothing
\]

| Layer | SHACL | Wire | Warehouse |
|-------|--------|------|-----------|
| Aspect specification | `sh:NodeShape` | `AspectSpec.shape` (`shape.id` = `signals.aspect.*`) | facts on catalog instance `signals.aspects.catalog` |
| Product specification | `sh:NodeShape` whose `sh:and` lists required aspects | `ProductSpec.shape` (`shape.id` = `signals.spec.*`) | `spec.<id>.requires.<aspect_id>` |
| Product instance | focus node | `ProductHint` + `AspectBinding[]` | `product_id` details / tx / hx |

The aspect catalog is the **shapes graph**. The warehouse projection of
one `product_id` (latest `details` asserts) is the **data graph**.
Validation produces a `ValidationReport` (`conforms`, `result[]`) on
each `AspectBinding`. Claim ≠ proof: `hx_reasoning` quality / lineage /
delta remains the ACP observation; evidence URIs live on `details`
(`aspect.<id>.evidence`).

This is **SHACL Core only** (Rec §§2–4). No `sh:sparql`, no
`sh:entailment`, no RDF store on the wire. v1 `PropertyPath` is a
**predicate path**. Attribute names are IRIs under
`https://signals.zndx.org/ns/dp#` (or the `details.a` local name).
`optional int32 max_count` so `0` is a real upper bound, not proto3
default-unset.

Do not mint `product_id`s for aspects. `signals.aspects.catalog` **is**
a product (it inventories shapes); the shapes it describes are not.

### Catalog (Signals-owned)

Architecture-neutral names. v1 **enforces** property constraints only
on `signals.spec.session` and on warehouse-product bindings already
fillable from existing facts (`peer`, `title`, `data_uri`, `flow`).

| `aspect_id` | Intent |
|-------------|--------|
| `signals.aspect.identifiable` | stable id, title, owner, version |
| `signals.aspect.discoverable` | description, tags, catalog visibility |
| `signals.aspect.contracted_interface` | what consumers may rely on |
| `signals.aspect.accessible` | how approved consumers obtain it |
| `signals.aspect.quality_assured` | fitness + evidence |
| `signals.aspect.observable` | freshness, lineage, incidents |
| `signals.aspect.governed` | roles, classification, retention |
| `signals.aspect.interoperable` | shared concepts, encodings |
| `signals.aspect.lifecycle_managed` | maturity, sunset |
| `signals.aspect.provenanced` | inputs, transforms, activity |
| `signals.aspect.cost_transparent` | quotas / economics when relevant |
| `signals.aspect.risk_managed` | continuity, restrictions |
| `signals.aspect.prepared_session_materials` | session **input**; immutable from the session’s POV |
| `signals.aspect.transcript` | session **output**; `in_force` then `sealed` |
| `signals.aspect.can_have_attachment` | invite capability; **upper bound** |

| `spec_id` | `shape.and` |
|-----------|-------------|
| `signals.spec.warehouse_product` | identifiable, discoverable, provenanced, accessible |
| `signals.spec.session` | identifiable, prepared_session_materials, transcript, can_have_attachment, provenanced, accessible |

Existing warehouse rows claim `warehouse_product`. Agenda sessions
**illustrate** `session`; they are not automatically inserted as
warehouse products.

### Worked shape: `can_have_attachment`

Upper bound is SHACL, not an ad-hoc bool rule.

```
NodeShape id=signals.aspect.can_have_attachment
  property path=dp:attachmentsAllowed  datatype=xsd:boolean min_count=1 max_count=1
  or = [ …allowed, …forbidden ]

NodeShape id=signals.aspect.can_have_attachment.allowed
  property path=dp:attachmentsAllowed  has_value=true
  property path=dp:attachments         min_count=0  node=signals.aspect.attachment

NodeShape id=signals.aspect.can_have_attachment.forbidden
  property path=dp:attachmentsAllowed  has_value=false
  property path=dp:attachments         max_count=0
```

`attachments_allowed=true` and an empty list **conforms**.
`attachments_allowed=false` and a nonempty list is
`sh:MaxCountConstraintComponent` (`sh:Violation`).

Attachment **pointers** may ride `PutAgendaItem` / `AGENDA`
(`AgendaHintItem.attachments`). Bytes stay on rustfs
(`s3://<origin_project>/resources/<note_id>/`) and are fetched at
Connect with `RESOURCES`. Fields `session_prompt` (16) and
`session_materials` (17) are deprecated — do not set on new writes.

### Discovery

`ServerQuery(kind=ASPECTS)` returns the shapes graph (`aspect_catalog`,
`product_specs`). `kind=PRODUCTS` returns instances with `spec_id` and
`AspectBinding[]`. Empty is honest. The warehouse remains SoR.
