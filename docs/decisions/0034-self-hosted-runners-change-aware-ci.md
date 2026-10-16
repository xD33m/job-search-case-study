# Self-hosted CI runners and change-aware CI

*Adapted from ADR 0034 of the private repository and its amendments 1–5.*

## Context

Waiting for tests was the main slowdown. The repository is private, so GitHub-hosted jobs ran on 2-vCPU runners with paid minutes, and a full suite run took 25–50 minutes there. Each push ran six jobs: lint, types and the suite with coverage; the whole suite again at three fixed clock instants; a multi-arch image build; a container smoke test; and PowerShell checks on Windows. With one runner, those ran one after another, which meant more than 30 minutes per PR even for a docs-only change.

## Options

| Option | Trade-off |
|---|---|
| Keep hosted runners, trim minutes | Still 2 vCPUs and 25–50 min, still paid |
| Make the repo public for free larger runners | Publishes code and fixtures; still 12–25 min |
| Larger hosted runners | Fast, but more expensive per minute |
| **Self-hosted runners on the desktop** | 20 threads, about 4 min per suite run, no minutes. CI only runs while the PC is on |

## Decision

- **Two desktop runners.** A Linux runner container (digest-pinned) with its *own* Docker-in-Docker daemon, reached over a Unix socket. Jobs can build and run containers without seeing the desktop's own containers. A native Windows runner handles the PowerShell job (Windows PowerShell 5.1 and pwsh 7, Pester). Repository variables can switch any job back to hosted runners without a code change.
- **Select jobs, not tests.** Per-test selection was rejected: the 98 % coverage gate is global, and the architecture, clock, template and network guards fail on changes far from their own files. A shell script maps the PR's diff to the jobs it needs:
  - code: the full gate;
  - image inputs: plus the container smoke test;
  - PowerShell or the shared signing contract: plus the Windows job;
  - the Cloudflare Worker: plus its tests and conformance vectors;
  - docs only: the docs validator and the one test that reads docs;
  - any change to CI itself: everything.

  A test keeps the script's docs list complete.
- **Nightly on `main`.** Everything, including the suite at the three clock instants (a weekday peak-price morning, the October DST night, New Year's Eve) and the amd64 + arm64 build. A night whose `main` already passed runs nothing.
- **Every push to `main` runs its own check** (amendment 4). PRs were being merged without waiting for their run, which the merge then cancelled, and one PR reached `main` with four failing tests. Each push run now gets a concurrency group of its own, so a later merge can't cancel it or push it out of the queue.
- **Resilience.** The runner restarts itself after losing DNS. Every check step runs under a wall-clock limit, and a dead pytest-xdist worker fails the run instead of hanging it.

## Consequences

- A typical PR gets feedback in minutes. Docs PRs need no suite run.
- CI depends on the desktop being on. Queued runs start when it comes back, within GitHub's 24-hour window. This dependency later drove a separate runner on the Raspberry Pi for releases ([0044](0044-pi-builds-its-own-image.md)).
- The runner is a service with Docker access on a personal machine. It only serves this private repository.
