# Buzz on Hetzner Cloud via Coolify

This is the Coolify equivalent of the [Railway template](https://railway.com/deploy/buzz-relay-block):
Postgres, Redis, and the S3-compatible bucket are each provisioned as their
**own Coolify resource** on the private network, and only the relay is
exposed publicly. This is intentionally different from `deploy/compose/`,
which bundles everything (including Caddy) into one docker-compose stack for
a plain single-node VPS with no orchestrator — use that bundle instead if you
don't want Coolify in the picture.

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

## 2. Postgres, Redis, MinIO — separate resources

Add each of these under the "buzz" project / "production" environment. Do
**not** put them in the same docker-compose file as the relay — that's the
part this setup deliberately avoids.

- **+ New Resource -> Databases -> PostgreSQL.** Leave it on its default
  version (17.x). Once it's running, open its "Connection" tab and copy the
  internal connection string — this becomes `DATABASE_URL`.
- **+ New Resource -> Databases -> Redis.** Set a password. Copy the internal
  connection string (with password) — this becomes `REDIS_URL`.
- **+ New Resource -> Services -> MinIO.** Set a root user/password. After
  it's running, open the MinIO console (or use `mc`) and:
  1. Create a bucket named `buzz-media`.
  2. Set its access policy to private (`mc anonymous set none local/buzz-media`).

  There's no `minio-init` sidecar in this setup (unlike `deploy/compose/`),
  so this bucket creation is a one-time manual step.

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
   cross-resource ones from the Shared Variables added in step 3.
3. Add a persistent volume mount: container path `/data/git` (backs
   `BUZZ_GIT_REPO_PATH`).
4. Health check: path `/_readiness`, port `8080`.
5. Ports Exposes: `3000`. Assign the `BUZZ_DOMAIN` domain — Coolify issues a
   Let's Encrypt cert and wires its proxy automatically; you don't need Caddy
   here.
6. Deploy.

## 5. Post-deploy

- If `BUZZ_AUTO_MIGRATE=false`, run migrations once:
  ```bash
  docker exec -it <relay-container> /usr/local/bin/buzz-admin migrate
  ```
- Verify:
  ```bash
  curl -fsS "https://$BUZZ_DOMAIN/_liveness"
  curl -fsS "https://$BUZZ_DOMAIN/_readiness"
  ```
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
- The MinIO resource's bucket contents (media + git CAS objects).
- The `buzz-git-data` volume attached to the relay resource
  (`BUZZ_GIT_REPO_PATH=/data/git`).

Keep Postgres, MinIO, and the git volume snapshots from the same maintenance
window — they reference each other.
