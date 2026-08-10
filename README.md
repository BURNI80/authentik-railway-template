# Authentik on Railway

<!-- Replace TEMPLATE_ID with the real template id after publishing. -->
[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/new/template/TEMPLATE_ID?utm_medium=integration&utm_source=button&utm_campaign=authentik)

Self-host [Authentik](https://goauthentik.io) — a modern, open-source Identity Provider (OAuth2/OIDC/SAML/LDAP/SCIM) — on Railway with one click.

## Deploy and Host Authentik with Railway

Authentik is an open-source Identity Provider that centralises authentication across all your applications. It gives you SSO, MFA, user lifecycle management, and a policy engine for a fraction of the cost of Okta or Auth0. This template deploys the full Authentik stack — **server**, **worker**, and a managed **PostgreSQL** database — pre-wired over Railway's private network with an auto-generated secret key. You configure nothing: just deploy, open the URL, and create your admin account.

## What's included

| Service | Source | Notes |
| --- | --- | --- |
| **Authentik Server** | `ghcr.io/goauthentik/server:2026.5.6` | API, UI, SSO flows. Public domain, healthchecked on `/-/health/ready/`, volume at `/data`. |
| **Authentik Worker** | `ghcr.io/goauthentik/server:2026.5.6` | Background tasks (Dramatiq on Postgres). Private network only. |
| **PostgreSQL** | Railway managed database | Primary datastore. Referenced via `${{Postgres.*}}`, never exposed publicly. |
| **Volume** | Railway volume, 1 GB min | Persists media, uploaded icons and certificates at `/data`. |

> **No Redis.** Authentik 2026.5+ moved its task queue and cache onto PostgreSQL and dropped Redis, matching the upstream official `docker-compose.yml`. This keeps the stack cheaper and smaller.

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

## Estimated cost

Railway bills per second for CPU, RAM and storage (`$20`/vCPU-month, `$10`/GB RAM-month, `$0.15`/GB-month volumes). This stack, kept at minimal sizes and always-on:

| Item | Estimate / month |
| --- | --- |
| Authentik server (1 GB RAM) | ~$8–15 |
| Authentik worker (512 MB RAM) | ~$4–8 |
| PostgreSQL (smallest managed instance) | ~$3–8 |
| Volume (1 GB) | ~$0.15 |
| **Total** | **~$10–25** |

The Hobby plan includes a **$5/month usage credit**; usage above that is billed on top. New accounts start with a one-time **$5 trial credit**. For a tighter budget you can sleep or scale services down when idle. Check [railway.com/pricing](https://railway.com/pricing) for current rates.

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
| Deploy stuck in "Building"/healthcheck timeout | First boot runs DB migrations. Wait up to 10 minutes (healthcheck timeout is set to 600s). Check `authentik-server` logs for `PostgreSQL connection successful`. |
| Worker logs database errors | The worker may start before migrations finish; it retries. Confirm the `Postgres` service is healthy and the `${{Postgres.*}}` references are resolved (no empty values). |
| Uploaded images disappear after redeploy | The `/data` volume keeps media. If you removed the volume, reattach one at `/data` — existing files are not recoverable. |
| Login fails with "session" errors | `AUTHENTIK_SECRET_KEY` must match between server and worker. Both read the shared variable — don't override it on one service only. |
| SSL/proxy errors | Your instance is served over HTTPS by Railway; do not set `AUTHENTIK_DEBUG=true`. |

## FAQ

**Why are there two Authentik services?** Authentik runs a server (HTTP/SSO/UI) and a worker (background tasks: blueprints, certificates, event processing). Both connect to the same PostgreSQL database and share `AUTHENTIK_SECRET_KEY`.

**Does this work with Outposts / proxy auth?** Yes for OAuth2/OIDC/SAML/LDAP providers. The Docker-socket based Proxy Outpost is not supported on Railway (no Docker socket access); use a manual/embedded proxy outpost if you need forward-auth.

**Can I run this on the Railway free/Hobby plan?** Yes, at minimal sizes. Be aware the stack exceeds the $5 Hobby credit — see the cost table above.

## About hosting Authentik

Authentik is a full-featured IAM platform written in Python, released under the MIT license by Authentik Security Inc. Railway hosts your infrastructure, so you don't have to configure Postgres, TLS, or healthchecks yourself — deploy and manage everything from one dashboard.

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
