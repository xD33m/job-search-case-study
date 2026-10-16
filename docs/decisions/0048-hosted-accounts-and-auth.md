# Hosted accounts: sessions, magic links, OAuth with PKCE, roles and retention

*Adapted from ADR 0048 of the private repository (hosted edition, step H3), with the login-proxy sign-in from ADR 0066 and the data layout from ADR 0046.*

## Context

The app began as one instance per person, behind a password or a trusted reverse-proxy login. The hosted mode serves many users from one install, first as an invite-only friends stage with Pro granted by an admin. It needed accounts, sessions, passwordless sign-in, social login, an admin role and GDPR-style retention, all without weakening the single-user mode.

## Options

| Question | Chosen | Why |
|---|---|---|
| Tenant isolation | **A shared SQLite file plus one file per user** (not `user_id` columns, not Postgres with RLS) | A missed tenant filter can't leak a CV; export and deletion are file operations |
| OAuth | **Two hand-written flows over httpx** (not an OAuth framework with its own session middleware) | No new dependency, no second session mechanism, every request fakeable in tests |
| Magic-link consumption | **GET shows a confirm page; POST consumes** | An email scanner's prefetch can't use up the link |

## Decision

- **Accounts store.** A separate SQLite database with its own migration chain. The schema lives in the core, and the logic lives in the hosted package.
- **Sessions.** Server-side sessions behind a `__Host-` cookie. Only the token's SHA-256 is stored. Sessions slide over 30 days and are renewed at most hourly. Users can log out of this session or of every session.
- **Magic links.** 32-byte URL-safe tokens, stored as SHA-256, single-use, valid for 15 minutes, consumed by a conditional `UPDATE` that must hit exactly one row. The login form returns the same status, body and headers whether or not the address exists. Invites are single-use, bound to one address and valid for 14 days.
- **Bot protection.** Turnstile on the two anonymous forms, checked before anything else.
- **GitHub and Google sign-in.** Authorization-code flows with PKCE and a short-lived state, linked to accounts by *verified* email only. The number of pending states is bounded.
- **Admin role.** Invites, plan grants, disabling and deleting accounts. Pool-wide decisions and platform actions are admin-only.
- **Entitlements.** Plan features are cached for 30 seconds, overridable per account, and checked at the route *and* right before any AI call.
- **Retention.** Accounts unused for 12 months are deleted, with a warning 30 days before. The account page offers export and deletion.
- **Behind a login proxy (ADR 0066).** When the install sits behind Traefik and a self-hosted OIDC provider, every request must carry the proxy's shared secret and user header. A session counts only for the address the proxy vouches for. New accounts need open signup or an invite naming that address. The anonymous login routes aren't mounted at all in this setup.

## Consequences

- One codebase serves both modes. Personal mode keeps its single-user login, and hosted mode can't enable the parts that only suit a private install.
- An end-to-end browser test caught a referrer-policy setting that made Chromium send `Origin: null`, which the same-origin check refused, so nobody could have signed in. Unit tests alone didn't see it.
- Known limits are written down: a small timing difference between known and unknown addresses in invite mode, and cache windows after a revoke.
- The owner's existing single-file database was migrated into the split layout on a rehearsal copy first, then live following a runbook, with the pre-split file kept for rollback.
