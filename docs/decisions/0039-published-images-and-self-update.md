# Published images and a self-update with backup, health check and rollback

*Adapted from ADRs 0039 (published images and image updates) and 0036 (self-update) of the private repository.*

## Context

The owner shares the app with friends who self-host it. Until then, every install was a git checkout that built its own image. On a Pi that was slow, and it needed an x86 machine for one step: the upstream MOTIS arm64 binary crashes on the Pi 4's Cortex-A72 (illegal instruction), so a compatible build had to be cross-compiled. Friends shouldn't need the repository, a compiler or a terminal session to update. Off-the-shelf auto-updaters such as Watchtower were ruled out because they offer no backup, no migration rollback, no busy check and no UI.

## Decision

- **One multi-arch tag.** A manual workflow cross-compiles MOTIS for ARMv8.0, so it runs on every arm64 CPU, and combines it with the official amd64 build into one image. The release workflow builds `linux/amd64,linux/arm64` on every merge and tags it `sha-<hash>` and `<date>.<run>`. It moves `latest`, the tag installs follow, **only after the amd64 image passed a smoke test by digest**.
- **Images describe themselves.** Version, revision, build time and the newest 50 commits are baked in as OCI labels and a JSON file. The status page shows what an update would bring.
- **The image is the installer.** `job-search deploy bundle` writes the Compose files, the env template, the setup script and the host updater as a tar stream. A new install is a registry login, one `docker run … | tar -x` and `sh setup.sh`. Updates replace only managed files, never `.env` or a local override.
- **A host updater with a narrow interface.** It's a standalone, standard-library Python program on the host, started by systemd path and timer units. The app never gets the Docker socket. It only writes a request file into a shared directory, and the updater writes state and a log back. Each update:
  1. runs a preflight: disk space and the app's own busy check, so no running fetch or heavy task gets cut off;
  2. takes a backup and verifies it before anything changes;
  3. keeps the running image as `previous`, then pulls the channel tag;
  4. restarts with the Compose files the stack was *started with*, and waits until every service is healthy;
  5. on failure, rolls back: retags `previous`, restores the database backup if the schema revision changed, and starts again.

  A run interrupted by a power cut is detected on the next run, and the log points to the manual steps.
- **Channels.** An install follows `latest` by default. The Pi follows `pi` ([0044](0044-pi-builds-its-own-image.md)). An opt-in automatic update runs at a set morning time.

## Consequences

- Friends update with one button, or automatically. A broken release rolls back on its own instead of leaving a dead stack.
- This held up in practice. The first update after the Pi moved to the hosted layout stopped at the Compose config check, because a new overlay needed a variable that install didn't set. Nothing restarted, and the app kept running on the previous image. The fix (keep an install's own Compose files when it runs behind a login proxy) shipped in the next PR.
- The updater can only replace itself after an update succeeds. A bug in the updater itself therefore needs one manual copy.
