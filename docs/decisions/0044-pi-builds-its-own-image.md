# The Raspberry Pi builds and installs its own arm64 image

*Adapted from ADR 0044 of the private repository.*

## Context

The Pi 4 updates only from published images ([0039](0039-published-images-and-self-update.md)), and the release workflow ran on the desktop runner ([0034](0034-self-hosted-runners-change-aware-ci.md)). The desktop is often off, or on without Docker running, so a merge sat in the queue and the Pi kept the old version. When the desktop did run it, a release took about 18 minutes, mostly the arm64 half emulated under QEMU. The Pi itself is always on.

## Options

| Option | Trade-off |
|---|---|
| **A runner on the Pi with its own `pi` tag** | Works with the desktop off, native arm64 with no QEMU, triggered by the merge. Cost: one more service on the home server |
| A build script over SSH | Needs the machine holding the key, which is the desktop |
| Wake the desktop for every merge | Still 18 minutes, and the PC boots for each merge |
| The Pi as a fallback builder for `latest` | `latest` would lose amd64 whenever the Pi built it |

## Evidence (measured on the Pi 4)

- Cold arm64 build in a BuildKit builder capped at cores 2–3 and 2 GB: **272 s**. Peak load average 8.4, below the thresholds of the Pi's own load governor and watchdog.
- The same build after a change in `src/`: **16 s**. Only the last two layers change.
- Loading into the local daemon instead of pushing took 369 s, because it exports the whole 1.7 GB image every time. So the workflow pushes, and pulls back only the changed layers.
- The idle runner service takes about 50 MB in its cgroup and no measurable CPU.

## Decision

- **The runner** is GitHub's runner, checksum-verified, running as a systemd service under its own system user with no sudo. It runs with `MemoryMax=768M` and `Nice=10`. Its Docker login is kept apart from the one the updater uses.
- **The workflow** runs on every push to `main`. It builds `linux/arm64` in a persistent builder (layer cache kept between runs and capped in size), pushes `pi-sha-<hash>`, pulls back by digest, runs the smoke-test script, moves the `pi` tag and then **installs it**. A newer push cancels an older run.
- **Installation goes through the same path as the status page's "Update" button.** The app's busy check runs first, then the host updater's backup, health check and rollback. A busy app is asked again every 2 minutes for up to 30 minutes. After that, the job only warns, and the status page still offers the update. A failed or rolled-back update fails the job.
- The desktop's release keeps publishing multi-arch `latest` for every other install. The weekly live smoke test against the real sources moved to the Pi runner too. It now fails loudly instead of hiding results in an artifact.

## Consequences

- A merge is live on the Pi a few minutes later, with the desktop off, unless the app is busy.
- Each merge costs the Pi a short burst of work bounded to half its cores. The first cold run, including the first push of the whole image, took 11 minutes.
- Docker access is effectively root, so the runner is limited to this private repository, and only two workflows target its label.
- The `pi` tags are arm64-only and meant for this one install.
