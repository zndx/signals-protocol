# Operations specifications

Cross-project **operational** contracts for the signals federation (identity,
secrets, co-tenancy). Distinct from wire protocols under
[`../protocol/`](../protocol/).

| Document | Status |
|----------|--------|
| [Kerberos + SecretSpec](./kerberos_and_secretspec.md) | **Binding** — how adopters join Signals core services with GSSAPI + Ranger |
| Co-tenancy (GPU leases) | Planned (promote Gaius draft) |

Procedures land here first; each project implements via its own devenv and
`secretspec.toml`, then bumps this submodule.
