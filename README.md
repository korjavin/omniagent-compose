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

## Auth: header mode

`OMNIGENT_AUTH_PROVIDER=header` — omnigent reads the authenticated identity
from `X-Forwarded-Email` and rejects requests without it (401). The Traefik
labels attach `auth-errors@docker,forward-auth@docker`, so oauth2-proxy is
what actually authenticates.

**Security:** the header name must be one the forward-auth middleware lists in
`authResponseHeaders` — Traefik deletes and re-sets exactly those from the
auth response, which is what stops a client from spoofing an identity.
`oauth2-proxy-compose` publishes `X-Forwarded-User,X-Forwarded-Email`, so the
default is safe. Point `OMNIGENT_AUTH_HEADER` at anything *not* in that list
and the server will trust whatever the client sends.

### Known limitation: external runners

Omnigent's server does not execute agents — runners do, and they register from
outside (`omni login <url>` / `omni host <url>`). Those requests come from a
CLI with no oauth2-proxy cookie, so forward-auth will 401 them. The browser UI
works fine; hosting a laptop runner through this hostname will not, until
either:

- a runner-only Traefik router is added on the registration path **without**
  the forward-auth middleware (needs upstream's path list), or
- the runner reaches the container directly over the docker network / a
  Tailscale address, or
- auth is switched to native OIDC against pocket-id (drops forward-auth
  entirely, keeps CLI login working).

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
