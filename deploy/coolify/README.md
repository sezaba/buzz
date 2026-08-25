# Buzz on Hetzner Cloud via Coolify

This is the Coolify equivalent of the [Railway template](https://railway.com/deploy/buzz-relay-block):
Postgres and Redis are each provisioned as their **own Coolify resource**,
the S3-compatible bucket is a managed **Hetzner Object Storage** bucket
(Coolify's MinIO one-click service was [removed from its
catalog](https://coolify.io/docs/services/minio) — no longer maintained by
its original author), and only the relay is exposed publicly. This mirrors
Railway's own architecture even more closely than self-hosted MinIO would
have: Railway's "Bucket" is also a managed add-on, not a container you run
yourself. This setup is intentionally different from `deploy/compose/`,
which bundles everything (including Caddy and a self-hosted MinIO) into one
docker-compose stack for a plain single-node VPS with no orchestrator — use
that bundle instead if you don't want Coolify or Hetzner Object Storage in
the picture.

## 1. Server + Coolify

1. Create a Hetzner Cloud server (Ubuntu 24.04; a CX22/CX32 is enough to
   start — the relay is the only thing that needs meaningful CPU/RAM here).
2. SSH in and install Coolify:
   ```bash
   curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
   ```
3. Point `BUZZ_DOMAIN`'s DNS A record at the server's public IP.
4. In the Coolify UI, create a Project (e.g. "buzz") and a "production"
   environment inside it.

## 2. Postgres, Redis (Coolify resources) + Hetzner Object Storage (external)

Add Postgres and Redis under the "buzz" project / "production" environment.
Do **not** put them in the same docker-compose file as the relay — that's
the part this setup deliberately avoids.

- **+ New Resource -> Databases -> PostgreSQL.** Leave it on its default
  version (17.x). Once it's running, open its "Connection" tab and copy the
  internal connection string — this becomes `DATABASE_URL`.
- **+ New Resource -> Databases -> Redis.** Set a password. Copy the internal
  connection string (with password) — this becomes `REDIS_URL`.

The bucket is provisioned separately, in the **Hetzner Cloud Console** (not
Coolify — there's no Coolify resource for it):

1. Hetzner Cloud Console -> Object Storage -> Create Bucket. Pick a region
   (Falkenstein/`fsn1`, Nuremberg/`nbg1`, or Helsinki/`hel1`) and name it
   `buzz-media`.
2. Create an S3 access key/secret for that bucket in the same console.
3. Note the region endpoint, e.g. `https://fsn1.your-objectstorage.com` —
   this becomes `BUZZ_S3_ENDPOINT`. Hetzner Object Storage is
   virtual-hosted-style (`https://<bucket>.<region>.your-objectstorage.com`),
   so `BUZZ_S3_ADDRESSING_STYLE` must be `virtual`, not `path`.

There's no `minio-init` equivalent here — the bucket already exists in a
private state by default; just confirm it isn't set to public in the console.

## 3. Shared Variables

Coolify doesn't auto-wire env vars between independent resources the way
Railway's private network does. The closest equivalent is **environment-level
Shared Variables**: Project -> production environment -> Variables. Add the
values you collected in step 2, plus the relay's own secrets, once:

- `DATABASE_URL`, `REDIS_URL`
- `BUZZ_S3_ENDPOINT`, `BUZZ_S3_ACCESS_KEY`, `BUZZ_S3_SECRET_KEY`
- `RELAY_OWNER_PUBKEY`, `BUZZ_RELAY_PRIVATE_KEY`, `BUZZ_GIT_HOOK_HMAC_SECRET`
- `BUZZ_DOMAIN`, `RELAY_URL`, `BUZZ_MEDIA_BASE_URL`, `BUZZ_MEDIA_SERVER_DOMAIN`, `BUZZ_CORS_ORIGINS`

Then reference them from the relay app's own env var list instead of
retyping values — one edit updates every resource that references it.

## 4. Deploy the relay

1. **+ New Resource -> Application -> Docker Compose** (public/private repo
   source: `block/buzz`, branch: whatever you're deploying, base directory:
   `deploy/coolify`, compose file: `docker-compose.yml`).
   This repo's `docker-compose.yml` only defines the `relay` service using
   the prebuilt `ghcr.io/block/buzz` image — Coolify does not build anything
   for this resource.
2. Set the environment variables listed in `.env.example`, pulling the
   cross-resource ones from the Shared Variables added in step 3. Postgres
   and Redis must sit on the **same Coolify Destination** (Docker network)
   as this app, and `DATABASE_URL` / `REDIS_URL` must be the **internal**
   connection strings from those resources — the public ones are unreachable
   from the relay container.
3. The compose file already mounts `buzz-git-data` at `/data/git`. Do **not**
   add a second `/data/git` volume in the Coolify UI or the mounts collide.
4. **Do not configure a Coolify UI health check on port 8080.** The image
   listens on three ports; only 3000 is public:
   | Port | Role | Coolify |
   |------|------|---------|
   | **3000** | App (WebSocket, HTTP, `/_liveness`, `/_readiness`) | Domain + proxy. This is the only port Coolify should know about. |
   | **8080** | Health-only listener (K8s/Istio probes) | Internal. The compose `healthcheck` already probes it *inside* the container. |
   | **9102** | Prometheus `/metrics` | Internal. Do not put this on a domain. |
5. Assign the domain as `https://$BUZZ_DOMAIN:3000` — the `:3000` suffix is
   the **container** port so Traefik/Caddy targets the app listener, not
   8080 or 9102. Coolify still serves public HTTPS on 443 and issues the
   Let's Encrypt cert. You don't need Caddy in this compose file.
6. Deploy.

## Troubleshooting an `exited` container

Coolify's "Deployment successful" only means `docker compose up` returned
zero. If the relay then crashes (bad `DATABASE_URL`, Redis unreachable, S3
conformance probe, invalid key), Docker leaves the container `exited` and
Coolify's **Runtime logs** tab stays empty because it only tails running
containers. The logs are still on the server:

```bash
# On the Coolify / Hetzner host
docker ps -a --filter name=relay --format 'table {{.ID}}\t{{.Status}}\t{{.Names}}'
docker logs --tail 200 <container-id>
```

The relay logs JSON to stdout. Typical fatal lines:

- `Failed to connect to Postgres` — wrong Destination network, or you pasted
  the public URL instead of the internal one. Append `?sslmode=disable` if
  Coolify Postgres is not offering TLS and the client is trying SSL.
- `PubSub init failed` / `Redis pool creation failed` — same network issue,
  or the Redis password in `REDIS_URL` is missing/unescaped.
- `git conformance probe failed` — Hetzner Object Storage endpoint, keys,
  bucket name, or `BUZZ_S3_ADDRESSING_STYLE=virtual` is wrong. This probe
  runs at boot and **exits the process** on failure (`BUZZ_GIT_CONFORMANCE_PROBE`
  defaults to on).
- `Invalid configuration` — a required env var failed validation (owner
  pubkey, relay private key, `RELAY_URL`, …).
- `Database migration failed` — with `BUZZ_AUTO_MIGRATE=true`, a schema
  error is fatal.

`restart: unless-stopped` will loop a crashing process. If Coolify shows
`exit` but `docker ps` shows `Up`, the UI is not seeing a published port
([Coolify #6591](https://github.com/coollabsio/coolify/issues/6591)); fix
the domain to `https://$BUZZ_DOMAIN:3000` and redeploy so the proxy labels
include the load-balancer port.

## 5. Post-deploy

- If `BUZZ_AUTO_MIGRATE=false`, run migrations once:
  ```bash
  docker exec -it <relay-container> /usr/local/bin/buzz-admin migrate
  ```
- Verify through the public proxy (port 3000, not 8080):
  ```bash
  curl -fsS "https://$BUZZ_DOMAIN/_liveness"
  curl -fsS "https://$BUZZ_DOMAIN/_readiness"
  ```
  Those paths exist on the app listener. Hitting 8080/9102 from the public
  hostname will fail — Coolify is not proxying them, by design.
- Manage relay membership the same way as the compose bundle, via `docker exec`:
  ```bash
  docker exec -it <relay-container> /usr/local/bin/buzz-admin add-member --pubkey <npub-or-hex>
  docker exec -it <relay-container> /usr/local/bin/buzz-admin list-members
  ```

## Backups

Back these up on a regular schedule, same as `deploy/compose/`:

- The Shared Variables/secrets — especially `BUZZ_RELAY_PRIVATE_KEY`,
  `BUZZ_GIT_HOOK_HMAC_SECRET`, and the DB/Redis/S3 credentials.
- The owner private key if one was generated for `RELAY_OWNER_PUBKEY`.
- The Coolify PostgreSQL resource's data (Coolify's built-in backup feature,
  or `pg_dump`).
- The Hetzner Object Storage bucket contents (media + git CAS objects) —
  Hetzner durably stores this already, but back up separately if you need
  point-in-time recovery, since Hetzner doesn't version objects for you.
- The `buzz-git-data` volume attached to the relay resource
  (`BUZZ_GIT_REPO_PATH=/data/git`).

Keep Postgres, the bucket, and the git volume snapshots from the same
maintenance window — they reference each other.
