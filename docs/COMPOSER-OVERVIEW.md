# Deploy and Host Authentik (SSO + MFA) on Railway

Authentik is a self-hosted, open-source Identity Provider (IdP) that centralises authentication across your applications. It provides SSO, MFA (TOTP, WebAuthn/passkeys), user lifecycle management and a policy engine, replacing services like Okta or Auth0 with a stack you fully control.

## About Hosting Authentik (SSO + MFA)

This one-click template deploys the full Authentik stack on Railway: a **server** (web UI, REST API and SSO flows), a **worker** (background tasks such as blueprints, certificates and event processing) and a managed **PostgreSQL** database. Server and worker are pre-wired over Railway's private network and share an auto-generated `AUTHENTIK_SECRET_KEY`, so sessions stay valid across both. Media is persisted on a 1 GB volume mounted at `/data`, and the server is healthchecked on `/-/health/ready/`. First boot runs database migrations automatically — no manual environment variables, no configuration, no Redis (2026.5+ runs the task queue and cache on PostgreSQL).

## Common Use Cases

- Centralise authentication for self-hosted apps (Grafana, Nextcloud, Gitea, Jellyfin and more).
- Add SSO to internal tools with OAuth2/OIDC or SAML.
- Enforce MFA (TOTP and WebAuthn/passkeys) across a team with a single policy.
- Replace Okta/Auth0 in SMB environments with a self-hosted, MIT-licensed IdP.

## Dependencies for Authentik (SSO + MFA) Hosting

- **Authentik server** — `ghcr.io/goauthentik/server:2026.5.6` (pinned, never `:latest`). Public HTTPS domain, healthcheck on `/-/health/ready/`, volume at `/data`.
- **Authentik worker** — same pinned image, private network only, shares `AUTHENTIK_SECRET_KEY` with the server.
- **PostgreSQL** — Railway managed database, referenced with `${{Postgres.*}}` variables, never exposed publicly.

### Deployment Dependencies

- Upstream project: [github.com/goauthentik/authentik](https://github.com/goauthentik/authentik)
- Authentik documentation: [docs.goauthentik.io](https://docs.goauthentik.io)
- Docker image: [ghcr.io/goauthentik/server:2026.5.6](https://ghcr.io/goauthentik/server)
- Template repository: [github.com/BURNI80/authentik-railway-template](https://github.com/BURNI80/authentik-railway-template)

### Implementation Details

- **Start commands** use a `sh -c "exec ..."` wrapper because Railway applies the start command as an exec-form ENTRYPOINT override: `/bin/sh -c "exec ak server"` (server) and `/bin/sh -c "exec ak worker"` (worker).
- **No Redis:** Authentik 2026.5+ moved the task queue (Dramatiq) and cache onto PostgreSQL, matching the upstream `docker-compose.yml`.
- **Zero-config setup:** the initial-setup wizard at `/if/flow/initial-setup/` asks you to set the `akadmin` password on first boot (security by default, no bootstrap credentials shipped).
- **Free-tier friendly:** both apps deploy with app sleeping enabled, so they only bill while awake.

## Why Deploy Authentik (SSO + MFA) on Railway?

Railway is a singular platform to deploy your infrastructure stack. Railway will host your infrastructure so you don't have to deal with configuration, while allowing you to vertically and horizontally scale it.

By deploying Authentik (SSO + MFA) on Railway, you are one step closer to supporting a complete full-stack application with minimal burden. Host your servers, databases, AI agents, and more on Railway.
