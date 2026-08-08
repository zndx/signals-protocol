# Operations specifications

Cross-project **operational** contracts for the signals federation (identity,
secrets, co-tenancy). Distinct from wire protocols under
[`../protocol/`](../protocol/).

| Document | Status |
|----------|--------|
| [Kerberos + SecretSpec](./kerberos_and_secretspec.md) | **Binding** — join Signals core services with GSSAPI + Ranger |
| [MiNiFi sentinels](./minifi_sentinels.md) | **Binding design** — MiNiFi C++ sentinels (C2 + OTel); **Knative** scale-to-zero + **YuniKorn** admission on RKE2 |
| Co-tenancy (GPU leases) | Planned (promote Gaius draft; composes with sentinels) |

Procedures land here first; each project implements via its own devenv and
`secretspec.toml`, then bumps this submodule.

**Sentinel substrate submodule (Signals):** `components/minifi-cpp` →
`git@github.com:weathership/oss-minifi-cpp.git`.
