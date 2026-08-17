# Status.surfaces — advertise a primary UI

**For:** Gaius, Signals, Ægir, Atelier, and any later lattice engine.

**Do not** ship a static roster of peer URLs. A waffle / app-launcher
lists only engines that **advertise** a surface and answer `Engine/Status`.

## Wire

Additive `zndx.engine.v1.StatusResponse.surfaces[]`:

| field | meaning |
|-------|---------|
| `kind` | `primary` (product UI), `health`, or `gateway` |
| `url` | absolute HTTP(S) URL a browser can open |
| `healthy` | last-known liveness of that surface |

Empty `surfaces` is honest: this engine has no public UI.

`ServerQuery` kind `SURFACES` returns the same list for the answering
engine. `PEERS` returns Engine `host:port` only — not UI URLs.

## Discovery (S2S, not gossip)

1. Seed engine targets from **config** and/or `SIGNALS_ENGINE_TARGET`
   (lattice hub **gRPC**, not a canned UI).
2. `Engine/Status` each target. If `surfaces` has `kind=primary`, list it.
3. One-hop: `ServerQuery PEERS` on whoever answered; Status those too.
4. Skip self. Skip peers that omit `surfaces`. Do not invent
   `http://127.0.0.1:9889` for Signals, or any other project URL.

A third-party engine that registers `zndx.engine.v1`, answers Status,
and fills `surfaces` appears without a Gaius/Signals code change.

## What each peer should ship

| Project | Typical primary (self-advertised, not assumed) |
|---------|-----------------------------------------------|
| Gaius | `GAIUS_PRIMARY_UI` / `GAIUS_UI_BIND` → e.g. `http://127.0.0.1:9890` |
| Signals | `SIGNALS_UI_URL` → e.g. `http://127.0.0.1:9889` |
| Ægir | own env, when the product UI exists |
| Atelier | own env, when the workbench UI exists |

Implementation: fill `StatusResponse.surfaces` in **your** Status
servicer. Regenerate stubs from this proto. Do not hard-code other
projects' ports in a chrome waffle.

## Local copies

This file lives in the protocol tree. Lab checkouts may also keep a
working copy at:

- `~/local/src/zndx/gaius/build/dev/current/federation/status-surfaces.md`
- `~/local/src/wxs/signals/build/dev/current/federation/status-surfaces.md`
