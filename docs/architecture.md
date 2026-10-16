# Architecture

This page adds four views to the component diagram in the [README](../README.md): the processing pipeline, one LLM call, the deployment topologies and the CI/release flow. Names match the private codebase. Internal hostnames, addresses and source-specific details are left out.

## 1. Processing pipeline

```mermaid
flowchart TD
  F[Fetch<br/>scheduled source adapters] --> N[Normalise<br/>postings become cards]
  N --> D1[Dedupe stages 1-3<br/>URL, exact keys, SimHash bands]
  D1 --> D4[Dedupe stage 4<br/>5-word shingle overlap]
  D4 -->|"overlap 0.9 or more"| M[Merge]
  D4 -->|"0.3 to 0.9"| P[Possible duplicate<br/>user confirms]
  M --> X
  D4 -->|distinct| X[Facts extraction<br/>small model, strict schema,<br/>no CV in the prompt]
  X --> S["Salary<br/>stated, else extracted, else estimate<br/>clamped to official wage quartiles"]
  S --> C[Commute<br/>MOTIS transit and car times]
  C --> FL[Filters, per user]
  FL --> SC[Deterministic score 0-100<br/>stored per-factor breakdown]
  SC --> FIT[Fit text, per user<br/>top cards only, lazy]
  SC --> NT[Digest and pushes<br/>ntfy, Home Assistant]
  FIT --> UI[Web UI and tracker]
  NT --> UI
  UI -->|on demand| PREP[Draft CV edits and letter<br/>larger model]
  PREP --> G[Grounding check<br/>every claim vs. the CV]
  G --> R[User reviews the diff<br/>then exports a PDF]
```

Each heavy stage queues the next one when it finishes. Queue items run under database leases with heartbeats and a watchdog, so a crashed or hung worker gives its task back instead of holding it forever. The facts call depends only on the posting. Its result can therefore be shared across users, and between separate installs through the *commons*: a content-addressed, first-write-wins cache served by a Cloudflare Worker with D1, or by a self-hosted server. Both implementations pass the same conformance vectors.

## 2. One LLM call

```mermaid
sequenceDiagram
  autonumber
  participant Task as LLM task module
  participant Client as ChatClient (ProviderProfile)
  participant Ledger as Ledger (SQLite)
  participant API as Provider API

  Task->>Client: extract(tool, messages, model, check)
  Client->>Ledger: reserve(estimate, payer, purpose)
  Note over Ledger: BEGIN IMMEDIATE: sum usage + live holds<br/>for the day / month, refuse if over the cap
  Ledger-->>Client: reservation (or BudgetExceeded)
  Client->>API: strict structured request (temperature 0)
  API-->>Client: answer + usage
  Client->>Client: Pydantic validation + check()
  alt invalid (at most 2 more attempts)
    Client->>API: same request + the validation error appended
  end
  Client->>Ledger: settle(real usage), or settle_unknown at the estimate
  Client-->>Task: Result[T] or a typed error
```

No write transaction is held across the HTTP call. The protocol refuses a call made while one is open. If a settle fails, the reservation later expires into an estimated usage row, so spend can be over-counted but never under-counted.

## 3. Deployment topologies

```mermaid
flowchart LR
  subgraph Personal["Personal install (one Pi)"]
    A1[app] --- H1[heavy]
    A1 --- W1[wol sidecar]
    A1 --- DB1[(one SQLite file)]
  end
  subgraph PiHosted["The Pi in hosted mode (invite-only)"]
    T[Traefik] --> OIDC[OIDC login proxy]
    T --> A2[app]
    A2 --- H2[heavy]
    A2 --- DB2[(shared file +<br/>one file per user)]
  end
  subgraph Public["Public host overlay (designed and staged)"]
    CF[Cloudflare] --> CA[Caddy, digest-pinned]
    CA --> A3[app]
    A3 --- H3[heavy]
  end
```

- **Personal:** the base Compose file. A Compose profile limits the Wake-on-LAN sidecar to this mode. It wakes a Windows desktop at night for the full map import, and afterwards a PowerShell power helper on the PC asks the Pi whether to power down. It acts only on an answer signed with its worker token and bound to the nonce it just sent, so a replayed or unsigned answer never powers anything down. The Pi in hosted mode keeps using the desktop worker.
- **Pi, hosted mode:** the same stack plus a Traefik overlay. Sign-in is delegated to a self-hosted OIDC provider. The app checks a shared proxy secret on every request.
- **Public overlay:** Caddy is the only service with published ports. It accepts traffic from Cloudflare's ranges with an Origin CA certificate. Uvicorn trusts forwarded headers only from Caddy's fixed address. This overlay passed a ten-step staging smoke run on the Pi. A real public server is not in use.

## 4. CI and release flow

```mermaid
flowchart TD
  PR[Pull request] --> CH[ci_changes.sh<br/>diff vs. base]
  CH -->|code| CK["check: ruff, pyright strict,<br/>full suite + 98 % coverage, docs"]
  CH -->|docs only| DC[docs: OKF validator + docs-reading test]
  CH -->|image inputs| SM[smoke: build amd64, run the container]
  CH -->|PowerShell| PS[Windows runner: parse + Pester]
  CH -->|commons| CW[Worker tests + conformance vectors]

  MAIN[Push to main] --> CH
  MAIN --> REL[release: desktop runner<br/>amd64 + arm64 via QEMU]
  REL --> SMK[smoke test by digest] --> LAT[move tag latest]
  MAIN --> RPI[release-pi: Pi runner<br/>native arm64 build]
  RPI --> SMK2[pull by digest, smoke] --> PIT[move tag pi] --> INST[host updater installs<br/>busy check, backup, health, rollback]

  NIGHT[Nightly on main] --> ALL["everything + suite at 3 clock instants<br/>+ multi-arch build, skipped if main passed"]
  WEEK[Weekly] --> LIVE[live smoke vs. real sources on the Pi]
```

Installs follow a *channel* tag. The Pi follows `pi`, everyone else `latest`. The status page shows when a newer image exists and what changed, drawn from the release notes baked into the image labels.
