# omniagent-compose

[Omnigent](https://github.com/omnigent-ai/omnigent) server + Postgres, behind
Traefik + oauth2-proxy, deployed to Portainer via GitOps.

Adapted from upstream [`deploy/docker/`](https://github.com/omnigent-ai/omnigent/tree/main/deploy/docker).

## How it works

```
edit master ──► deploy.yml ──────────► deploy branch (compose pinned) ──► Portainer webhook
bump tag   ──► vendor-images.yml ──► ghcr.io/korjavin/omnigent-server-vendor:d-<digest12>
                                  └► deploy branch + webhook
```

- **Portainer watches the `deploy` branch, not `master`.** `master` carries
  `:latest` as a placeholder; the deploy branch rewrites it to the vendored
  digest tag and records it in `image-tags.env`.
- **Images are vendored, not built.** Upstream's `deploy/docker/Dockerfile`
  cannot be built outside their repo — the web UI bundle is gitignored, which
  is why upstream ships `Dockerfile.prebuilt` that just does
  `FROM ghcr.io/omnigent-ai/omnigent-server`. So `vendor-images.yml` mirrors
  the published image into our GHCR namespace and tags it by upstream digest.
  Bump `UPSTREAM_TAG` in that workflow (or dispatch it with an
  `upstream_tag` input) to roll a new version.
- Postgres comes straight from Docker Hub (`postgres:16-alpine`). Vendor it
  too if Hub rate limits ever bite.

## One-time setup

1. **GitHub secret** — `PORTAINER_REDEPLOY_HOOK` = the stack's webhook URL
   (Portainer → your stack → Webhooks).
   `https://github.com/korjavin/omniagent-compose/settings/secrets/actions`
2. **Run the vendor workflow once** so the GHCR package exists, then make the
   package public (or add a Portainer registry credential for GHCR).
3. **Create the Portainer stack** from this repo, branch **`deploy`**,
   compose path `docker-compose.yml`, and set the env vars below.

## Portainer environment variables

Required:

| Variable | Example | Notes |
|---|---|---|
| `OMNIGENT_HOST` | `omnigent.example.com` | Traefik `Host()` rule; also `OMNIGENT_DOMAIN` in-container |
| `POSTGRES_PASSWORD` | `openssl rand -hex 16` | Stack fails to start without it |

Optional (defaults in parentheses): `POSTGRES_USER`/`POSTGRES_DB`
(`omnigent`), `OMNIGENT_AUTH_HEADER` (`X-Forwarded-Email`),
`TRAEFIK_NETWORK_NAME` (`traefik_default`), `TRAEFIK_CERTRESOLVER`
(`myresolver`), `OMNIGENT_CONFIG` (unset), plus the container/volume/network
name overrides. Full list in [`.env.example`](.env.example).

## Auth: OIDC (default)

`OMNIGENT_AUTH_PROVIDER=oidc` — omnigent runs its own login flow against your
IdP. No forward-auth middleware: `OMNIGENT_MIDDLEWARES` is empty by default,
so Traefik passes requests straight through and omnigent authenticates them.

That is deliberate, not an oversight. Omnigent's server doesn't execute
agents — runners do, and they register from outside via `omnigent login <url>`
→ `omnigent host <url>`. That flow hits `/auth/cli-login`, which **only exists
in oidc mode**, and it arrives with no browser cookie, so forward-auth in
front of it would 401 every attempt.

Set up in your IdP a client with redirect URI exactly:

```
https://<OMNIGENT_HOST>/auth/callback
```

The server derives that from `OMNIGENT_HOST`; there is no
`OMNIGENT_OIDC_REDIRECT_URI` knob. Then set `OMNIGENT_OIDC_ISSUER`,
`_CLIENT_ID`, `_CLIENT_SECRET` and `_COOKIE_SECRET` (`openssl rand -hex 32`)
in the Portainer stack env.

### Alternative: header mode behind oauth2-proxy

Works for the browser UI only — no CLI, no external runners. Needs three
things to line up, and getting any one wrong yields a silent 401 on every
`/v1/*` call while the UI shell still loads:

```
OMNIGENT_AUTH_PROVIDER=header
OMNIGENT_MIDDLEWARES=auth-errors@docker,forward-auth@docker
OMNIGENT_AUTH_HEADER=<a name in the middleware's authResponseHeaders>
```

**Security:** Traefik del+sets *only* the headers named in the middleware's
`authResponseHeaders`, and that is what blocks a client from spoofing an
identity. Point `OMNIGENT_AUTH_HEADER` at a name outside that list and the
server trusts whatever the client sends. Note oauth2-proxy with
`SET_XAUTHREQUEST=true` emits `X-Auth-Request-Email`, *not*
`X-Forwarded-Email` — if the middleware only forwards the latter, no identity
reaches omnigent at all.

## Local test

```bash
cp .env.example .env   # set POSTGRES_PASSWORD + OMNIGENT_HOST
docker compose up -d
docker compose logs -f omnigent
```

Header mode 401s every request without a proxy in front — for a bare local
smoke test set `OMNIGENT_AUTH_ENABLED=0` in the compose environment
temporarily (single-user local mode; never on a shared deploy).

## Upstream docs

- [Deploy overview](https://omnigent.ai/docs/deploy/overview)
- [deploy/docker README](https://github.com/omnigent-ai/omnigent/blob/main/deploy/docker/README.md)
- [config.yaml.example](https://github.com/omnigent-ai/omnigent/blob/main/deploy/docker/config.yaml.example)
