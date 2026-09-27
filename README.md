# NetBird Docker Setup

Self-hosted [NetBird](https://netbird.io) mesh VPN using the combined
`netbirdio/netbird-server` image, the `netbirdio/dashboard` SPA, and the shared
`global-postgres` instance for the management store.

Architecture:

- **netbird-server** — single container running management, signal, relay, and
  STUN (combined image, v0.65.0+). Backend only — does NOT serve the dashboard UI.
- **dashboard** — separate `netbirdio/dashboard` container that serves the web
  SPA at `/`. Talks to `netbird-server` via the public URL, not the docker network.
- **global-postgres** (external, in `../global-postgres`) — shared PostgreSQL
  instance. Provides the `netbird`, `netbird_activity`, and `netbird_idp`
  databases, all owned by the `netbird_user` role.
- **nginx-proxy-manager** (external, in `../nginx-proxy-manager`) — terminates
  TLS. Routes API / gRPC / WebSocket paths to `netbird-server` on
  `NETBIRD_HTTP_PORT` (default `8081`) and everything else (`/`) to `dashboard`
  on `NETBIRD_DASHBOARD_PORT` (default `8080`).

Peer login flow: log in to the embedded Dex IdP with the initial owner account,
then federate to Authentik as an external OIDC provider via the Dashboard UI.

## Prerequisites

- Docker + Docker Compose
- The shared `global-postgres` stack running on this host (see
  [Provisioning the database](#provisioning-the-database))
- A public domain pointed at this host (e.g. `netbird.mgsfoxhound.net`)
- nginx-proxy-manager (or another reverse proxy) with HTTP/2 support
- Firewall port-forward: **UDP 3478** direct to this host (STUN traffic cannot
  be proxied)
- Optional but recommended: reachable Authentik instance for SSO

## Quick Start

1. Copy the example files:

   ```bash
   cp .env.example .env
   cp config.yaml.example config.yaml
   ```

2. Edit `.env` — at minimum set:

   - `DOCKER_DATA_PATH`
   - `NETBIRD_DOMAIN`, `NETBIRD_PUBLIC_URL`
   - `NETBIRD_DATASTORE_ENC_KEY` (`openssl rand -base64 32`)
   - `NETBIRD_DB_PASSWORD` (must match what you'll provision in the DB below)

3. Edit `config.yaml` — replace every `REPLACE_...` placeholder and every
   `netbird.example.com`:

   ```bash
   openssl rand -base64 32   # server.authSecret
   ```

   `server.exposedAddress` / `server.auth.issuer` / `dashboardRedirectURIs`
   must all point at the same public URL as `NETBIRD_PUBLIC_URL` in `.env`.

4. Provision the NetBird database in `global-postgres` — see
   [below](#provisioning-the-database).

5. Create the data directory:

   ```bash
   mkdir -p ${DOCKER_DATA_PATH}/data
   ```

6. Start the stack:

   ```bash
   docker compose up -d
   ```

7. Set up the reverse proxy (see [Nginx-Proxy-Manager](#nginx-proxy-manager)).

8. Open `https://netbird.mgsfoxhound.net/` in a browser. Log in with the owner
   credentials from `config.yaml`.

9. (Recommended) Federate to Authentik — see
   [Authentik Federation](#authentik-federation).

## Configuration

Compose-level knobs (paths, image tags, DB credentials, port mappings, and any
secret the netbird-server image accepts via env var) live in `.env`. NetBird's
public URL, OIDC issuer, redirect URIs, embedded IdP owner, and reverse-proxy
trust live in `config.yaml`. NetBird does NOT expand `${VAR}` inside
`config.yaml`, so values are literal.

### What overrides what

The netbird-server image applies configuration in this order (later beats
earlier): defaults → `config.yaml` → env vars → CLI flags. NetBird's own docs
are inconsistent about whether `NETBIRD_STORE_ENGINE_POSTGRES_DSN` and
`NETBIRD_DATASTORE_ENC_KEY` env vars are read directly by the running
combined-image binary, or only by the `configure.sh` setup script. To avoid
relying on that ambiguity, this stack does **not** pass store config via env
vars at all — `config.yaml`'s `store`, `activityStore`, and `authStore`
blocks are the single source of truth for Postgres connection details and
the datastore encryption key.

`authSecret` has no env-var equivalent for the combined image either way, so
it must be set in `config.yaml`.

`.env`'s `NETBIRD_DB_*` variables are informational only (documenting what
must match `config.yaml` and the `global-postgres` `.env`) — they are not
read by `netbird-server`.

### Volume Paths

- `DOCKER_DATA_PATH` — base directory on the host. Compose appends:
  - `${DOCKER_DATA_PATH}/data` → `/var/lib/netbird` (server data, keys —
    the embedded IdP DB and activity events also live here only if you
    fall back to the default SQLite engines for those stores)
  - `${DOCKER_DATA_PATH}/config.yaml` → `/etc/netbird/config.yaml`
    (read-only mount; this file must exist at that exact path on the host —
    copy/sync it there from this repo before starting the stack)

PostgreSQL data lives in the separate `global-postgres` stack — not here.

### Image Tags

- `NETBIRD_TAG` — track the latest at
  [Docker Hub](https://hub.docker.com/r/netbirdio/netbird-server/tags). Pin to
  a full version like `0.75.0` for zero-surprise updates.
- `NETBIRD_DASHBOARD_TAG` — [Dashboard tags](https://hub.docker.com/r/netbirdio/dashboard/tags).

### Ports

Ports are bound on `0.0.0.0` — any interface on this host can reach them.

- `NETBIRD_DASHBOARD_PORT` (default `8080`) — dashboard SPA. NPM's catch-all
  `/` route points here.
- `NETBIRD_HTTP_PORT` (default `8081`) — HTTP/gRPC backend. NPM's location
  blocks below re-route API / gRPC / WebSocket paths here.
- `NETBIRD_STUN_PORT` (default `3478`) — UDP, **must be publicly reachable**.

### Database

The stack uses the shared `global-postgres` instance for its management store.
Both containers join the `POSTGRES_NETWORK_NAME` external network so
netbird-server can reach `global-postgres` by service name.

## Provisioning the database

This stack uses three separate Postgres databases, all owned by the same
`netbird_user` role:

- `netbird` — main management store (`server.store` in `config.yaml`)
- `netbird_activity` — activity/audit log events (`server.activityStore`)
- `netbird_idp` — embedded Dex IdP store (`server.authStore`)

The `global-postgres` init script provisions databases on first boot from
env vars. If your `global-postgres` is already running (the common case),
you'll need to create the databases and user by hand:

```bash
docker exec -it global-postgres psql -U postgres <<SQL
CREATE USER netbird_user WITH PASSWORD 'CHANGE_ME_MATCH_NETBIRD_DB_PASSWORD';
CREATE DATABASE netbird OWNER netbird_user;
GRANT ALL PRIVILEGES ON DATABASE netbird TO netbird_user;
CREATE DATABASE netbird_activity OWNER netbird_user;
GRANT ALL PRIVILEGES ON DATABASE netbird_activity TO netbird_user;
CREATE DATABASE netbird_idp OWNER netbird_user;
GRANT ALL PRIVILEGES ON DATABASE netbird_idp TO netbird_user;
SQL
```

Use the same password for the `dsn` values in `config.yaml`'s `store`,
`activityStore`, and `authStore` blocks (and for `NETBIRD_DB_PASSWORD` in
this stack's `.env`, which is documentation-only — see
[What overrides what](#what-overrides-what)) as for the psql command above.

For future first-boot deployments of `global-postgres`, the NetBird block in
the global-postgres `.env` provisions all three automatically via its
comma-separated `_DB` list convention:

```env
NETBIRD_DB=netbird,netbird_activity,netbird_idp
NETBIRD_DB_USER=netbird_user
NETBIRD_DB_PASSWORD=changeme
```

## Migrating from the getting-started.sh install

If you're upgrading from the NetBird `getting-started.sh` output (SQLite-backed,
single `netbird_data` volume):

1. Optional — export a list of peers, groups, and policies from the current
   dashboard for reference. Peer credentials cannot be preserved across the
   rebuild.

2. Stop the current stack and remove the SQLite data:

   ```bash
   docker compose down
   rm -rf ${DOCKER_DATA_PATH}/data
   ```

3. Follow [Quick Start](#quick-start) with the new files.

4. Re-invite peers. On each device:

   ```bash
   netbird up --management-url https://netbird.mgsfoxhound.net
   ```

The old `authSecret` and `store.encryptionKey` are no longer valid — a fresh
set of secrets was generated in step 2 of Quick Start.

## Nginx-Proxy-Manager

NPM proxies two upstreams for the same domain: the dashboard SPA at `/`
(catch-all) and the backend for `/api/*`, `/oauth2/*`, `/relay`,
`/ws-proxy/*`, and every `/*.gRPC*/` path.

1. In NPM, add a new **Proxy Host**:
   - Domain: `netbird.mgsfoxhound.net`
   - Scheme: `http`
   - Forward hostname/IP: this NetBird host's LAN IP (e.g. `192.168.1.142`).
     Since ports are bound on `0.0.0.0`, NPM at any LAN address can reach them.
   - Forward port: `8080` (dashboard — the catch-all `/` upstream). Backend
     paths are re-routed to `8081` via the Advanced tab below.
   - Enable **Block Common Exploits**
   - Enable **Websockets Support**

2. In the **SSL** tab: request a Let's Encrypt cert, enable **HTTP/2 Support**,
   enable **Force SSL**.

3. In the **Advanced** tab, paste the blocks from
   [`npm-advanced-config.txt`](./npm-advanced-config.txt). Replace
   `192.168.1.142:8081` with your NetBird host's LAN IP and
   `NETBIRD_HTTP_PORT` if you changed either.

4. Save. Test by opening `https://netbird.mgsfoxhound.net/` — you should get
   the login page.

## Firewall

The STUN server must be reachable from the public internet, or peers behind
symmetric NAT can't connect:

- Port-forward **UDP 3478** → this host on your router/firewall.
- Do NOT try to proxy this through NPM. UDP + STUN protocol don't map to HTTP.

The TCP relay path (WebSocket over HTTPS) goes through 443 via NPM — no extra
port-forward needed.

## Authentik Federation

Once you're logged into NetBird as the local owner, add Authentik as a
federated identity provider so you can sign in with SSO.

### 1. Create the OAuth2 provider in Authentik

In Authentik → **Applications → Providers → Create**:

- Type: **OAuth2/OpenID Provider**
- Name: `NetBird`
- Client type: **Public** (NetBird uses PKCE)
- Authorization flow: `default-provider-authorization-explicit-consent`
- Signing key: any existing cert, e.g. `authentik Self-signed Certificate`
- Access code validity: `minutes=10`
- Subject mode: `Based on the User's ID`
- Redirect URIs (Strict mode):
  - `https://netbird.mgsfoxhound.net/nb-auth`
  - `https://netbird.mgsfoxhound.net/nb-silent-auth`
- Include claims in id_token: **enabled**

### 2. Create the application

In Authentik → **Applications → Applications → Create**:

- Name: `NetBird`
- Slug: `netbird`
- Provider: the `NetBird` provider you just created

### 3. Add the federation in NetBird

In the NetBird dashboard → **Settings → Identity Providers → Add**:

- Type: **Generic OIDC**
- Name: `Authentik`
- Client ID: (copy from Authentik provider)
- Issuer URL: `https://authentik.mgsfoxhound.net/application/o/netbird/`
- Client Secret: leave blank for public/PKCE clients

Save. The next time you sign out, the login page shows a "Sign in with
Authentik" button.

## Usage

### Starting

```bash
docker compose up -d
```

### Viewing logs

```bash
docker compose logs -f netbird-server
docker compose logs -f dashboard
```

### Stopping

```bash
docker compose down
```

### Upgrading

Bump `NETBIRD_TAG` and/or `NETBIRD_DASHBOARD_TAG` in `.env`, then:

```bash
docker compose pull
docker compose up -d
```

Read the [NetBird release notes](https://github.com/netbirdio/netbird/releases)
before crossing a minor version.

### Backup

The NetBird server keeps peer keys under `${DOCKER_DATA_PATH}/data`. All
three stores (management, activity, embedded IdP) live in the
`global-postgres` stack — back them up there.

```bash
# NetBird server data (peer keys, config)
docker compose down
tar -czf netbird-data-$(date +%Y%m%d).tar.gz -C ${DOCKER_DATA_PATH} data
docker compose up -d

# Hot Postgres backup (no downtime) — all three databases
for db in netbird netbird_activity netbird_idp; do
  docker exec -T global-postgres pg_dump -U netbird_user "$db" \
    | gzip > "netbird-db-${db}-$(date +%Y%m%d).sql.gz"
done
```

Also keep a copy of `.env` and `config.yaml` — they contain the encryption
keys needed to read the backup.

## Peer Onboarding

Install the NetBird client on each device:
<https://docs.netbird.io/how-to/installation>.

Point the client at your self-hosted server:

```bash
netbird up --management-url https://netbird.mgsfoxhound.net
```

The client opens a browser for OIDC login. After signing in (with the local
owner or via Authentik), the peer appears in the dashboard.

## Deploying with Komodo

This stack is set up for [Komodo](https://komo.do) deployments:

- The `dashboard` service uses `env_file: .env`, so Komodo can either mount
  `.env` from disk or inject each variable via its Variables UI.
- The `netbird-server` service does **not** use `.env` for anything at
  runtime — all of its secrets (store DSNs, encryption key, authSecret) live
  in `config.yaml`, which must be present on disk and mounted read-only into
  the container regardless of how Komodo manages `.env`.
- `${VAR:?...}` (e.g. `POSTGRES_NETWORK_NAME`) fails fast if a required value
  isn't set — Komodo surfaces the error immediately instead of starting a
  broken stack.
- `DOCKER_DATA_PATH` is the single knob for where NetBird's data lives on the
  host; point it at the host path Komodo manages for this stack, and ensure
  `config.yaml` is synced to `${DOCKER_DATA_PATH}/config.yaml` before Komodo
  starts the stack (Komodo's pre-deploy hook, or a file resource, works well
  here).
- Set `restart: unless-stopped` (already the default in `compose.yaml`) and
  make sure Docker itself is enabled to start on boot on the host
  (`systemctl enable docker`) — Komodo relies on the Docker daemon coming up
  automatically after a reboot in order to bring the stack back with it.

Import as a **Stack** in Komodo, wire up `.env` (or add each variable in the
Variables tab), make sure `config.yaml` lands on disk before deploy, and
deploy.

## Security Notes

- Never commit `.env` or `config.yaml` — both contain secrets. Both are
  `.gitignore`d by default.
- The embedded IdP owner is a real admin — set a strong password and enable
  TOTP after first login.
- Rotate `authSecret`, `NETBIRD_DATASTORE_ENC_KEY`, and `NETBIRD_DB_PASSWORD`
  on a schedule. Rotating `NETBIRD_DATASTORE_ENC_KEY` requires a data
  migration — plan before changing.
- Put HTTPS in front (NPM) for the dashboard/management/signal traffic. STUN
  over UDP is unencrypted by design.

## Troubleshooting

### Peers can't connect / "connection refused"

- Check UDP 3478 is open on the firewall and reaches this host.
- Confirm NPM is proxying gRPC (check the Advanced tab — the location blocks
  from `npm-advanced-config.txt` are non-optional).
- `docker compose logs netbird-server` — look for `starting management server`
  and `starting signal server`.

### `server.authSecret is required when running local relay`

`authSecret` in `config.yaml` is empty. Set it to a random 32-byte string
(`openssl rand -base64 32`).

### Container restarts / "failed to connect to postgres"

- Confirm `global-postgres` is running: `docker ps | grep global-postgres`.
- Confirm both stacks share the same network: `docker network ls | grep
  global-postgres-network` and `POSTGRES_NETWORK_NAME` in this `.env`.
- Confirm the `dsn` host in `config.yaml`'s `store`/`activityStore`/
  `authStore` blocks is the Postgres **container name** (`global-postgres`),
  not the network name (`global-postgres-network`) — these are easy to
  confuse and only the container name resolves over the Docker network.
- Confirm the password in each `dsn` matches the password on the
  `netbird_user` role for all three databases:
  `docker exec -it global-postgres psql -U netbird_user -d netbird -c '\q'`
  (repeat with `-d netbird_activity` and `-d netbird_idp`).
- `docker compose logs netbird-server` should show
  `using Postgres store engine` after a successful connection.

### Dashboard loads but login page never appears

The dashboard's OIDC discovery is hitting the wrong URL. Confirm
`NETBIRD_PUBLIC_URL` in `.env` and `server.auth.issuer` / `server.exposedAddress`
in `config.yaml` all point to the same URL (no trailing slash).

### `context deadline exceeded` on peer connect

The client is trying to reach management/signal on gRPC but the reverse proxy
isn't forwarding HTTP/2. In NPM, confirm **HTTP/2 Support** is checked on the
SSL tab and the `grpc_pass` blocks are present in Advanced.

### Migration: peers show "Registration Failed"

`authSecret` or `NETBIRD_DATASTORE_ENC_KEY` was rotated but the old peers are
still trying with cached credentials. Re-run
`netbird up --management-url https://netbird.mgsfoxhound.net` on each device.

## Documentation

- [NetBird Docs](https://docs.netbird.io/)
- [Self-Hosting Quickstart](https://docs.netbird.io/selfhosted/selfhosted-quickstart)
- [Environment Variables](https://docs.netbird.io/selfhosted/environment-variables)
- [PostgreSQL Store](https://docs.netbird.io/selfhosted/postgres-store)
- [Reverse Proxy Setup](https://docs.netbird.io/selfhosted/reverse-proxy)
- [Authentik + NetBird (official)](https://docs.netbird.io/selfhosted/identity-providers/authentik)
- [Combined config.yaml.example](https://github.com/netbirdio/netbird/blob/main/combined/config.yaml.example)

## License

This configuration is provided as-is. NetBird is licensed under the
BSD-3-Clause License.
