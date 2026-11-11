# Buzz on Coolify

Import `compose.yml` as a Docker Compose service on the existing amd64 Coolify
server. The storage images build locally from the official MinIO release
binaries with pinned SHA-256 checksums because the original registry images no
longer pull. The MinIO versions and persistent volume paths are unchanged.

In the Buzz service application, set the URL to `https://buzz.sezaba.de:3000`.
The port tells Coolify which internal container port to route to; the public URL
is `https://buzz.sezaba.de`. Coolify derives the relay, media, and CORS URLs from
this setting. DNS must point to the Coolify server, and ports 80 and 443 must be
reachable for HTTPS certificate issuance.

Keep the existing Coolify environment variables and volumes when updating the
stack. Coolify generates the `SERVICE_USER_*`, `SERVICE_PASSWORD_*`, and
`SERVICE_HEX_64_*` credentials; no credentials belong in this repository.
Set `RELAY_OWNER_PUBKEY` to the community owner's Nostr public key if required by
the deployed Buzz version.

Mark `minio-init` as **Exclude from service status** in Coolify. It is a one-shot
bucket initializer and should exit successfully, while Buzz waits for its zero
exit status before starting.

Validate syntax with:

```sh
docker compose -f deploy/coolify/compose.yml config --no-interpolate --quiet
```

After deploying, check Coolify's startup log for successful checksum verification,
healthy PostgreSQL, Redis, and MinIO, and a successful `minio-init` exit. Check the
Buzz readiness probe and `https://buzz.sezaba.de/health`, then load the public URL
in a browser with normal certificate verification.
