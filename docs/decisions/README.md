# Selected decisions

The private repository has 69 architecture decision records (ADR 0001–0069). Each has context, options, evidence (marked as measured or inferred), the decision, its consequences, and amendments as the system changed. The eight below were picked for their relevance to AI engineering and DevOps. They are condensed rewrites, not copies. The file names keep the original ADR numbers.

| ADR | Decision | Area |
|---|---|---|
| [0003](0003-llm-calls-and-validation.md) | Strict structured output, validation-driven retries, deterministic merge, a cost ledger | LLM |
| [0013](0013-llm-extracts-code-scores.md) | The LLM extracts facts; a pure function scores; a harness measures the ranking | LLM, evals |
| [0041](0041-shared-facts-per-user-fit.md) | CV-free facts shared across users; the per-user fit made lazily | LLM, cost |
| [0049](0049-provider-seam-payers-monthly-budget.md) | Provider profiles, a payer on every call, a hard monthly AI budget | LLM, multi-tenant |
| [0034](0034-self-hosted-runners-change-aware-ci.md) | Self-hosted runners; CI selects jobs from the diff | CI |
| [0044](0044-pi-builds-its-own-image.md) | The Raspberry Pi builds, smoke-tests and installs its own arm64 image | CD |
| [0039](0039-published-images-and-self-update.md) | Published multi-arch images; a host updater with backup, health check and rollback | CD, operations |
| [0048](0048-hosted-accounts-and-auth.md) | Accounts, sessions, magic links, OAuth with PKCE, retention | Security, multi-tenant |

Other ADRs in the private repository cover the source framework, SQLite write transactions under WAL, the composition root and process roles, lease-based task running, transit routing on the Pi, Home Assistant, the setup wizard, the shared cache between installs, the one-file-per-user data split and the hosted infrastructure.
