# Authentik on Railway

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/new/template/authentik-sso-mfa?utm_medium=integration&utm_source=button&utm_campaign=authentik)

**Template:** [Authentik (SSO + MFA) on the Railway marketplace](https://railway.com/template/authentik-sso-mfa)

Self-host [Authentik](https://goauthentik.io) — a modern, open-source Identity Provider (OAuth2/OIDC/SAML/LDAP/SCIM) — on Railway with one click.

## Table of contents

- [Overview](#overview)
- [What's included](#whats-included)
- [How it works](#how-it-works)
- [Deploy](#deploy)
- [Post-deploy steps](#post-deploy-steps)
- [Verify the deployment](#verify-the-deployment)
- [Local development (Docker Compose)](#local-development-docker-compose)
- [Estimated cost](#estimated-cost)
- [Optional configuration](#optional-configuration)
- [Custom domain](#custom-domain)
- [Upgrading](#upgrading)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)
- [About hosting Authentik](#about-hosting-authentik)
- [Common use cases](#common-use-cases)
- [Deployment dependencies](#deployment-dependencies)
- [License](#license)

## Overview

Authentik is an open-source Identity Provider that centralises authentication across all your applications. It gives you SSO, MFA, user lifecycle management, and a policy engine for a fraction of the cost of Okta or Auth0. This template deploys the full Authentik stack — **server**, **worker**, and a managed **PostgreSQL** database — pre-wired over Railway's private network with an auto-generated secret key. You configure nothing: just deploy, open the URL, and create your admin account.

## What's included

| Service | Source | Notes |
| --- | --- | --- |
| **Authentik Server** | `ghcr.io/goauthentik/server:2026.5.6` | API, UI, SSO flows. Public HTTPS domain on port 9000. |
| **Authentik Worker** | `ghcr.io/goauthentik/server:2026.5.6` | Background tasks (Dramatiq on Postgres). Private network only. |
| **PostgreSQL** | Railway managed database | Primary datastore. Referenced via `${{Postgres.*}}`, never exposed publicly. |

> The template attaches a persistent Railway volume at `/data` on the server, so uploaded media (icons, logos, avatars) survives redeploys. The readiness endpoint `/-/health/ready/` is served by Authentik itself (200 only when the database connection is up).

> **No Redis.** Authentik 2026.5+ moved its task queue and cache onto PostgreSQL and dropped Redis, matching the upstream official `docker-compose.yml`. This keeps the stack cheaper and smaller.

## How it works

```
                       Internet
                          │  HTTPS
                          ▼
                ┌───────────────────────┐        ┌───────────────────────┐
                │   authentik-server    │        │   authentik-worker    │
                │   (public, :9000)     │        │   (private, no domain)│
                │  UI · API · SSO       │        │  background tasks     │
                └───────────┬───────────┘        └───────────┬───────────┘
                            │      Railway private network   │
                            └───────────┐        ┌───────────┘
                                        ▼        ▼
                                  ┌─────────────────────┐
                                  │  PostgreSQL (managed)│
                                  │  `${{Postgres.*}}`  │
                                  └─────────────────────┘
```

- **Server** runs the web UI, the REST API and the SSO flows on port `9000`. Railway exposes it over a public HTTPS domain; the readiness endpoint `/-/health/ready/` returns 200 only when the database connection is up.
- **Worker** runs background work: blueprints, certificate handling, event processing, scheduled tasks. It is never exposed publicly and only talks to PostgreSQL over the private network.
- **PostgreSQL** is a Railway managed database, referenced by the apps with `${{Postgres.*}}` variables resolved at deploy time.
- **`AUTHENTIK_SECRET_KEY`** is generated once per deployment (`${{secret(64, ...)}}`), shared between server and worker so sessions stay valid across both.
- Both apps run with **app sleeping** enabled (Railway's default), so they only bill while awake.

Deep technical documentation (variables, start commands, storage, networking): [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Deploy

1. Click **Deploy on Railway** above.
2. If prompted, confirm the project region and deploy.
3. Railway provisions PostgreSQL, pulls the Authentik image and starts the server and worker. First boot runs database migrations, so give it a couple of minutes.

## Post-deploy steps

1. Open the **public domain** of the `authentik-server` service (your instance is served over HTTPS at `https://<service>.up.railway.app`).
2. You will be redirected to the **initial setup** wizard. Set a strong password for the `akadmin` superuser.
3. Log in to the **Admin Interface** at `/if/admin/`.
4. Add your first application: **Applications → Create**, then attach a provider (**OAuth2/OIDC**, **SAML**, **LDAP** or **Proxy**). Use `https://<service>.up.railway.app` as the base URL/issuer.
5. Add your first user or connect a source (LDAP, GitHub, etc.) under **Directory**.

That's it — your identity provider is live.

## Verify the deployment

- The healthcheck endpoint `/-/health/ready/` returns **HTTP 200** when the server is ready (503 while it sleeps or boots).
- Open `https://<service>.up.railway.app/if/admin/` and confirm the admin interface loads and you can log in.
- In the admin interface go to **System → Workers**: the worker should be listed as `healthy` while it's awake.

> If the site returns **503**, the service is simply sleeping (serverless). Visiting it wakes it in a few seconds.

## Local development (Docker Compose)

The repo includes a `docker-compose.yml` that mirrors the Railway stack for local testing. You need a `.env` with two values:

```dotenv
AUTHENTIK_SECRET_KEY=<64-char random hex>
PG_PASS=<your database password>
```

Then run:

```sh
docker compose up --build -d
```

- Authentik is served at `http://localhost:9000`.
- `AUTHENTIK_SECRET_KEY` must be identical for server and worker (the compose file reads it from `.env` for both).
- Data lives in `./data` (media) and a Docker volume for PostgreSQL.
- Note: on Docker Compose the start command is `server`/`worker` (plain). Only on Railway the wrapper `/bin/sh -c "exec ak ..."` is required — see [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Estimated cost

Railway bills per second for CPU, RAM and storage (`$20`/vCPU-month, `$10`/GB RAM-month, `$0.15`/GB-month volumes). Both Authentik services deploy with **app sleeping** enabled (Railway's default), so they bill only while awake — an idle instance costs almost nothing.

| Item | Estimate / month (idle-heavy) |
| --- | --- |
| Authentik server (1 GB RAM, sleeps when idle) | ~$0–2 |
| Authentik worker (512 MB RAM, sleeps when idle) | ~$0–2 |
| PostgreSQL (smallest managed instance) | ~$3–8 |
| Volume (1 GB) | ~$0.15 |
| **Total** | **~$5–10** |

The Hobby plan includes a **$5/month usage credit**; usage above that is billed on top. New accounts start with a one-time **$5 trial credit**. Under constant active traffic the server stays awake and costs grow toward ~$10–25/month; you can also scale each service per-your-usage. Check [railway.com/pricing](https://railway.com/pricing) for current rates.

## Optional configuration

The template works with zero configuration. The following variables can be added per-service afterwards for advanced setups:

| Variable | Service | Purpose |
| --- | --- | --- |
| `AUTHENTIK_BOOTSTRAP_EMAIL` / `AUTHENTIK_BOOTSTRAP_PASSWORD` | worker | Pre-create the `akadmin` account (skips the setup wizard). Read on **first boot only**. |
| `AUTHENTIK_EMAIL__HOST` (+ port, username, password, `FROM`) | server + worker | SMTP for password recovery, notifications and email stages. |
| `AUTHENTIK_WEB__WORKERS` | server | Raise from `1` if the instance is busy. |
| `AUTHENTIK_DISABLE_UPDATE_CHECK` | server + worker | Set `true` to disable the update checker. |

See the [Authentik configuration docs](https://docs.goauthentik.io/install-config/configuration/) for the full list.

## Custom domain

1. In Railway, open `authentik-server` → **Settings → Networking → Custom Domain**.
2. Add your domain and point the DNS record as shown. Railway issues and renews the TLS certificate automatically.

## Upgrading

1. In Railway, open `authentik-server` and `authentik-worker` → **Settings → Source**.
2. Change the image tag to the new version (e.g. `2026.5.6` → `2026.6.0`).
3. Redeploy both services. Upgrade sequentially by major version and keep server/worker/outposts on the same version.

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| Setup wizard not reachable / `akadmin` can't be created | The initial-setup flow is only available on first boot. If it expired, set `AUTHENTIK_BOOTSTRAP_EMAIL` and `AUTHENTIK_BOOTSTRAP_PASSWORD` on the worker, then redeploy to a fresh database. |
| Site shows 503 or 502 | The service is **sleeping** (serverless) or still booting. Wait a few seconds and reload; it wakes on request. If it persists, check the deploy logs for errors. |
| Deploy stuck in "Building" | First boot runs DB migrations. Wait a couple of minutes and check `authentik-server` logs for `PostgreSQL connection successful`. |
| Worker logs database errors | The worker may start before migrations finish; it retries. Confirm the `Postgres` service is healthy and the `${{Postgres.*}}` references are resolved (no empty values). |
| Uploaded images disappear after redeploy | The `/data` volume keeps media. If you removed the volume, reattach one at `/data` — existing files are not recoverable. |
| Login fails with "session" errors | `AUTHENTIK_SECRET_KEY` must match between server and worker. Both read the shared variable — don't override it on one service only. |
| Background tasks never run | The worker sleeps when idle (free-tier default). It wakes on private-network traffic; for always-on background processing, disable app sleeping on `authentik-worker`. |
| SSL/proxy errors | Your instance is served over HTTPS by Railway; do not set `AUTHENTIK_DEBUG=true`. |

## FAQ

**Why are there two Authentik services?** Authentik runs a server (HTTP/SSO/UI) and a worker (background tasks: blueprints, certificates, event processing). Both connect to the same PostgreSQL database and share `AUTHENTIK_SECRET_KEY`.

**Does this work with Outposts / proxy auth?** Yes for OAuth2/OIDC/SAML/LDAP providers. The Docker-socket based Proxy Outpost is not supported on Railway (no Docker socket access); use a manual/embedded proxy outpost if you need forward-auth.

**Can I run this on the Railway free/Hobby plan?** Yes. With app sleeping enabled (default), both services sleep when idle, so an idle-heavy stack fits the $5 Hobby credit — see the cost table above.

**Why start with the setup wizard instead of a default admin?** Security by default: the wizard makes you choose your own `akadmin` password. Nothing is pre-created with a known password.

**Do I need to configure anything?** No. The template resolves PostgreSQL connection variables and the secret key for you at deploy time.

## About hosting Authentik

Authentik is a full-featured IAM platform written in Python (2026.5+ ships a Rust core binary), released under the MIT license by Authentik Security Inc. Railway hosts your infrastructure, so you don't have to configure Postgres, TLS, or healthchecks yourself — deploy and manage everything from one dashboard.

## Common use cases

- Centralise authentication for self-hosted apps (Grafana, Nextcloud, Gitea, Jellyfin and more).
- Add SSO to internal tools with OAuth2/OIDC or SAML.
- Enforce MFA (TOTP, WebAuthn/passkeys) across a team with a single policy.
- Replace Okta/Auth0 in SMB environments with a self-hosted IdP.

## Deployment dependencies

- Upstream project: [github.com/goauthentik/authentik](https://github.com/goauthentik/authentik)
- Authentik docs: [docs.goauthentik.io](https://docs.goauthentik.io)
- Image: `ghcr.io/goauthentik/server:2026.5.6`
- Requires PostgreSQL (included, managed by Railway)

## License

This template repository is MIT-licensed. Authentik itself is MIT-licensed by Authentik Security Inc. This template does not redistribute Authentik source; it only references the official Docker image.
