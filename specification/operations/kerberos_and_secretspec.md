# Kerberos + SecretSpec: participating in Signals core services

**Status:** Binding procedures for federation adopters  
**Reference implementation:** [weathership/signals](https://github.com/weathership/signals) (first project to adopt both)  
**Audience:** Ægir, Atelier, Gaius, Hermes/ACP agents, and any future federated engine

This document is the **cross-project contract** for identity and secret material
when calling **Signals core services** (Atlas, Ranger, Impala, Kudu, Postgres
GSSAPI / impala_fdw, OpenLineage `/api/v1`, RustFS). Wire protocols
(`zndx.engine.v1`, OIP) are separate; without Kerberos + SecretSpec alignment,
engines cannot safely use the data plane or receive Ranger authorizations.

---

## 1. Roles of each layer

| Layer | Responsibility | Not responsible for |
|-------|----------------|---------------------|
| **Kerberos** | Authenticate principals (users, engines, agents, services) | Authorization decisions |
| **Ranger** | Authorize principals against resources/tags (Atlas classifications) | Issuing tickets |
| **Atlas** | Governance SoR (types, entities, classifications, OL surface) | Holding long-lived secrets |
| **SecretSpec** | **Declare** which secrets a project needs; **provision** keytab paths (or vault refs) per environment | Kerberos protocol, Ranger policies |
| **Non-secret config** | Realm, host FQDN, ports, profile names | Passwords, keytab bytes |

SecretSpec **does not** replace Kerberos. It ensures every federated process can
`kinit` (or start with a keytab) as a **well-known principal** that Ranger already
understands—without ambient shell pollution or per-repo secret dialects.

---

## 2. Shared non-secret identity (must match everywhere)

Copy these into each project's devenv / docs. Values are **not** secrets.

| Variable | Example (lab) | Meaning |
|----------|---------------|---------|
| `KRB5_REALM` | `DEV.VISTA.ZNDX.ORG` | Realm = `{ENV}.{LOCATION}.ZNDX.ORG` |
| `SIGNALS_KRB_ENV` | `dev` | Environment segment (lower) |
| `SIGNALS_KRB_LOCATION` | `vista` | Location segment |
| `SIGNALS_KRB_HOST` | `tinybox.dev.vista.zndx.org` | FQDN for SPNs — **never** `127.0.0.1` for GSSAPI |
| `KRB5_KDC_PORT` | `8848` | Lab KDC (loopback) when using Signals-hosted KDC |

Naming:

- **FQDN:** `{host}.{env}.{location}.zndx.org`
- **Realm:** `{ENV}.{LOCATION}.ZNDX.ORG` (uppercase)
- **PG / Unix-style role:** Kerberos **primary** only (`alice@REALM` → role `alice` via `pg_ident`)

SecretSpec **profile** should align with env:

| Profile | Typical realm |
|---------|----------------|
| `default` / `dev` | `DEV.VISTA.ZNDX.ORG` |
| `stage` | `STAGE.VISTA.ZNDX.ORG` |

```bash
devenv --secretspec-profile dev shell
# CI:
devenv --secretspec-provider env --secretspec-profile stage shell
```

---

## 3. Principal catalog (federation)

All principals live in the **same realm** (or a trusted realm). Ranger policies
and Impala/Kudu GSSAPI use these names.

### 3.1 Signals stack services (owned by signals repo)

Minted by Signals KDC bootstrap (`just bootstrap` / `kdc-init` in weathership/signals).

| Principal | Role |
|-----------|------|
| `postgres/$SIGNALS_KRB_HOST@REALM` | Postgres GSSAPI |
| `impala/$SIGNALS_KRB_HOST@REALM` | Impala HS2 |
| `kudu/$SIGNALS_KRB_HOST@REALM` | Kudu |
| `HTTP/$SIGNALS_KRB_HOST@REALM` | Atlas / Ranger SPNEGO |
| `signals@REALM` | Lab user + FDW outbound default |

### 3.2 Engine services (each federated project)

Long-running `zndx.engine.v1` (and native) processes **must** have a principal.

| Convention | Example |
|------------|---------|
| Preferred | `{project}-engine@REALM` |
| Host-bound alternative | `{project}/$SIGNALS_KRB_HOST@REALM` |

Examples: `aegir-engine@DEV.VISTA.ZNDX.ORG`, `atelier-engine@…`, `gaius-engine@…`.

### 3.3 ACP-enabled agents

| Pattern | Principal | When |
|---------|-----------|------|
| **User delegation** | Human `alice@REALM` | Interactive Hermes/ACP acting for an operator |
| **Agent service** | `hermes-acp@REALM` or `{project}-acp@REALM` | Unattended / batch agents |

Ranger policies for agent service identities **must be narrower** than human
admin principals. Prefer user delegation when a human is in the loop.

### 3.4 Who mints principals?

| Environment | Authority |
|-------------|-----------|
| Lab (Signals devenv KDC) | Signals operators extend `kdc-init` / document add-principal steps |
| Shared org KDC | Identity team; same principal naming as above |

Federated projects **do not** invent parallel realms (e.g. project-local MIT KDC
for Impala) if they need Signals Impala/Ranger.

---

## 4. SecretSpec procedures (adopter checklist)

### 4.1 Add SecretSpec to the project

1. Add `secretspec.toml` at repo root (declare **names only**, never values).
2. Enable in `devenv.yaml`:

```yaml
secretspec:
  enable: true
  provider: keyring   # recommended local; dotenv only if isolated .env
  profile: default    # or dev / stage matching realm
```

3. Prefer **not** loading secrets into the interactive shell. Run:

```bash
secretspec run -- <command that needs GSSAPI>
```

4. **Do not** use `provider: env` for day-to-day local work if you want to avoid
   ambient shell secrets. Use `env` only in CI where the job injects secrets.

### 4.2 Required secret **names** (harmonized)

Declare the subset your process needs. Names below are the **federation
contract**; use the same strings across repos so shared keyring/sops entries match.

#### Client material (engines, ACP, tools calling Signals data plane)

| Secret name | Description |
|-------------|-------------|
| `SIGNALS_KRB_USER_KEYTAB` | Path to client keytab for a user/lab principal (e.g. `signals@REALM`) used for Impala/FDW/GSSAPI clients |
| `ENGINE_KRB_KEYTAB` | Path to this project's engine service keytab |
| `ENGINE_KRB_PRINCIPAL` | Optional explicit principal string if not derivable from project name |
| `AGENT_KRB_KEYTAB` | Path to ACP agent service keytab (if not using user delegation) |
| `AGENT_KRB_PRINCIPAL` | Optional agent principal string |

#### Stack service material (only the signals host that runs Impala/Kudu/Atlas)

| Secret name | Description |
|-------------|-------------|
| `POSTGRES_KRB_KEYTAB` | `postgres/host@REALM` keytab path |
| `IMPALA_KRB_KEYTAB` | `impala/host@REALM` keytab path |
| `KUDU_KRB_KEYTAB` | `kudu/host@REALM` keytab path |
| `HTTP_KRB_KEYTAB` | `HTTP/host@REALM` keytab path (Atlas/Ranger SPNEGO) |

Values are **paths** (or provider refs), not keytab bytes in git. Ticket caches
(`KRB5CCNAME`) are **runtime**, never SecretSpec long-lived values.

### 4.3 Minimal `secretspec.toml` for a federated engine (template)

```toml
[project]
name = "aegir"   # or atelier, gaius, …
revision = "1.0"

[profiles.default]
# Non-secret realm/host still live in devenv.nix / env — not here.
SIGNALS_KRB_USER_KEYTAB = { description = "Lab/user client keytab for Impala/FDW GSSAPI", required = false }
ENGINE_KRB_KEYTAB = { description = "Engine service keytab (e.g. aegir-engine@REALM)", required = false }
ENGINE_KRB_PRINCIPAL = { description = "Engine principal if not default {project}-engine@REALM", required = false }

# Optional if this repo hosts ACP workers
AGENT_KRB_KEYTAB = { description = "ACP agent service keytab", required = false }
AGENT_KRB_PRINCIPAL = { description = "ACP agent principal", required = false }
```

Repeat (or import via profile copy) for `[profiles.dev]` / `[profiles.stage]`
with realm-appropriate descriptions.

### 4.4 Shared provider (recommended: Linux Secret Service)

So multiple projects obtain the **same** keytab path material without
`source_up`:

```toml
# ~/.config/secretspec/config.toml  (per developer machine)
[defaults.providers]
shared = "keyring://secretspec/shared/{profile}/{key}"
local  = "keyring://"
```

- Put **shared** client/engine keytab **paths** (or shared blobs if your ops model
  stores them that way) under the `shared` URI.
- Project toml allowlists only what that binary may inject.
- Import from a higher-level dotenv **once** into keyring (intersect with this
  project's declared keys)—do not leave secrets only in parent envrc.

```bash
# Example: set shared client keytab path into keyring
secretspec set SIGNALS_KRB_USER_KEYTAB --provider keyring
# paste path, e.g. /home/…/shared-keytabs/signals.keytab

secretspec run --provider keyring -- \
  bash -c 'kinit -kt "$SIGNALS_KRB_USER_KEYTAB" "signals@$KRB5_REALM" && …'
```

### 4.5 Process start pattern (required for GSSAPI)

**Engine service:**

```bash
secretspec run -- bash -c '
  set -euo pipefail
  princ="${ENGINE_KRB_PRINCIPAL:-${PROJECT}-engine@$KRB5_REALM}"
  kinit -kt "$ENGINE_KRB_KEYTAB" "$princ"
  exec your-engine-binary
'
```

**ACP agent (service identity):**

```bash
secretspec run -- bash -c '
  set -euo pipefail
  princ="${AGENT_KRB_PRINCIPAL:-hermes-acp@$KRB5_REALM}"
  kinit -kt "$AGENT_KRB_KEYTAB" "$princ"
  exec hermes-or-acp …
'
```

**ACP agent (user delegation):** interactive `kinit alice` (or user keytab via
`SIGNALS_KRB_USER_KEYTAB` / user-specific secret); no long-lived agent keytab.

Always dial **`$SIGNALS_KRB_HOST`** for Impala/Kudu/HTTP SPN match.

---

## 5. Ranger authorization participation

1. Principal exists in the Signals (or org) realm.  
2. Principal is known to Ranger (usersync / manual user / group).  
3. Policies grant access to Impala/Hive/Kudu resources and/or **Atlas tags**.  
4. Client uses GSSAPI; Impala Ranger plugin (or equivalent) evaluates the principal.

SecretSpec only enables step 0 (ticket). **Policy authoring** is a Signals/Ranger
ops task when onboarding a new engine principal.

Checklist for a new engine principal:

- [ ] Principal minted  
- [ ] Keytab path stored in shared keyring/sops  
- [ ] Declared in project `secretspec.toml`  
- [ ] Start wrapper runs `kinit` under `secretspec run`  
- [ ] Ranger policy (or group) allows required DBs/tables/tags  
- [ ] Smoke: GSSAPI Impala query or Atlas call as that principal  

---

## 6. Signals core service endpoints (lab defaults)

Non-secret; discovery will eventually advertise these (see protocol roadmap).

| Service | Lab endpoint | Authn |
|---------|--------------|--------|
| Atlas governance | `http://$SIGNALS_KRB_HOST:21010/api/atlas` | App / SPNEGO as configured |
| OpenLineage / Marquez-compat | `http://…:21010/api/v1` | Lab may be open; prod behind ZT |
| Ranger Admin | `http://…:6080` | Admin + SPNEGO as configured |
| Impala HS2 | `$SIGNALS_KRB_HOST:21050` | **Kerberos GSSAPI only** |
| Kudu | `$SIGNALS_KRB_HOST:7051` (masters) | Kerberos |
| Postgres (pglite) | `127.0.0.1:5455` | Password lab / GSSAPI target |
| RustFS (S3) | `127.0.0.1:9010` | S3 keys via SecretSpec when non-lab |

Governance **scale:** Atlas and Ranger stay JDBC clients of Postgres; bulk tables
live on Kudu and are reached via **impala_fdw foreign tables** on that Postgres.
See weathership/signals `docs/current/src/architecture/governance-scale-plane.md`.

---

## 7. What not to do

| Anti-pattern | Instead |
|--------------|---------|
| Project-local KDC for Impala while using Signals Impala | Join Signals/org realm |
| NOSASL / “auth optional” for product data plane | GSSAPI + Ranger |
| `source_up` dumping all parent secrets into every shell | keyring + project allowlist + `secretspec run` |
| `provider: env` for local interactive secrets | `keyring` or sops |
| Storing TGT / ccache in SecretSpec | kinit at process start |
| Keytab **bytes** in git | Paths or encrypted providers only |
| Engine using human admin principal for unattended work | Dedicated `*-engine@` with least privilege |
| Dialing `127.0.0.1` for GSSAPI to Impala | Dial `$SIGNALS_KRB_HOST` |

---

## 8. Onboarding flow (summary)

```text
1. Vendor signals-protocol (submodule) — this document + protos
2. Align non-secret KRB5_* / SIGNALS_KRB_* with Signals lab or org
3. Request engine (+ optional ACP) principals from Signals/org KDC ops
4. Store keytab paths in shared SecretSpec provider (keyring/sops)
5. Add secretspec.toml allowlist + devenv secretspec enable
6. Wrap engine/ACP start: secretspec run → kinit → binary
7. Register principals in Ranger; attach policies/tags
8. Smoke GSSAPI + one Ranger-enforced denied/allowed query
9. Register zndx.engine.v1; optional co-tenancy (see co-tenancy docs when published)
```

---

## 9. Reference implementation pointers

| Topic | Location (weathership/signals) |
|-------|--------------------------------|
| Lab KDC bootstrap | `just bootstrap`, `scripts/signals_kerberos.sh`, `docs/current/src/operations/kerberos.md` |
| SecretSpec layout | `secretspec.toml`, `docs/current/src/operations/secrets.md` |
| Identity layers (ZT + Kerberos + Ranger) | `docs/current/src/architecture/identity-and-access.md` |
| Scale plane (pglite → FDW → Kudu) | `docs/current/src/architecture/governance-scale-plane.md` |
| Engine gRPC face | `specification/protocol/engine_grpc.md` (this repo) |

When in doubt: **same realm, same secret names, SecretSpec allowlist, kinit before GSSAPI, Ranger on the principal.**
