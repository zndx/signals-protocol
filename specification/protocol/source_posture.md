# Source posture

**Status:** v1 additive — `ServerQueryKind.SERVER_QUERY_KIND_SOURCE_POSTURE` (8), `ServerQueryResponse.posture`.

A peer answers `Engine/ServerQuery(kind=SOURCE_POSTURE)` with a `SourcePosture`:
**what code the engine is actually running** — commit, dirtiness, submodule pins,
and unapplied migrations. It extends the existing `REMOTES` view (git remotes +
`head`) so any federation engine can know the source posture of Ægir, Atelier,
Gaius, Signals, … and vice-versa, without shell access to the peer's host.

Adoption is incremental and honest, exactly like `REMOTES`: a peer that has not
implemented this kind returns an **empty `posture`** (or `UNIMPLEMENTED`), and the
caller **skips** it. A light peer (e.g. Metabase) can therefore participate later;
this document is the target it implements against.

## Message

```proto
message SourcePosture {
  string project = 1;      // who this posture is for
  string checkout = 2;     // host-local checkout path (may be empty; honest)
  string branch = 3;       // current branch, empty if detached
  string head = 4;         // checkout tip, read live (git rev-parse HEAD)
  string running_sha = 5;  // commit the LIVE process started from
  bool dirty = 6;          // superproject working tree has uncommitted changes
  string upstream = 7;     // tracking ref, e.g. "origin/trunk"; empty if none
  int32 ahead = 8;         // commits ahead of upstream
  int32 behind = 9;        // commits behind upstream
  repeated SubmodulePosture submodules = 10;
  repeated MigrationPosture migrations = 11;
}

message SubmodulePosture {
  string path = 1;             // submodule path within the superproject
  string name = 2;             // configured submodule name
  string pinned_sha = 3;       // superproject gitlink (git ls-tree HEAD <path>)
  string checked_out_sha = 4;  // commit actually checked out on disk
  bool dirty = 5;              // submodule working tree has uncommitted changes
}

message MigrationPosture {
  string source = 1;             // "dbmate" | "alembic" | "flyway" | ...
  string current = 2;            // latest applied migration id (empty if none)
  repeated string unapplied = 3; // migration ids present but not applied
}
```

## Field semantics

| Field | Rule |
|-------|------|
| `head` vs `running_sha` | `head` MUST be read live; `running_sha` MUST be stamped **once at process start**, so a long-lived engine reveals when its checkout moved under it (`running_sha != head`). |
| `dirty` | MUST reflect **tracked** modifications only (`git status --porcelain --untracked-files=no`). Untracked scratch/notes/IDE files MUST NOT set `dirty` — they do not change what runs. |
| `submodules[].pinned_sha` vs `checked_out_sha` | `pinned_sha` is the superproject gitlink; `checked_out_sha` is on disk. `pinned_sha != checked_out_sha` ⇒ the submodule is **moved off its pin**. Both MUST be full 40-hex when known. |
| `submodules[].dirty` | tracked modifications inside the submodule, same rule as the superproject. |
| `migrations[].unapplied` | migration ids **present in the repo but not recorded applied** (e.g. in `schema_migrations`). A broken/partial tracking table is itself posture — report it, do not hide it. |
| absent / zero | means **not reported**, never "clean". A peer MUST NOT invent values it cannot determine. |

## Rules

- **MUST** be additive — this kind and the `posture` field never change number or meaning within v1.
- **MUST** return an empty `posture` for `SOURCE_POSTURE` if unimplemented, so callers skip the peer (do not error).
- **MUST NOT** cross engine-private details already excluded from `zndx.engine.v1` (internal ports, log paths, secrets).
- **SHOULD** report every field the peer can determine cheaply from its own checkout; partial posture is valid and useful.
- **SHOULD** keep the response fast (local `git` + one migration query); this is a status probe, not a scan.

## Reference implementation

Gaius: `src/gaius/engine/s2s.py` — `build_source_posture()` (gatherers:
`working_tree_dirty`, `current_branch`, `upstream_ahead_behind`,
`submodule_postures`, `migration_postures`, `stamp_running_sha`), wired into
`local_response(SOURCE_POSTURE)`; the fleet sweep is `fleet_source_posture()`.
The running-SHA is stamped in the engine's startup path.
