# NetBird Docker Setup

Self-hosted [NetBird](https://netbird.io) mesh VPN using the combined `netbirdio/netbird-server` image, PostgreSQL, and nginx-proxy-manager for TLS.

Architecture:

- **netbird** — single container running management, signal, relay, and STUN (combined image, v0.65.0+).
- **db** — PostgreSQL, backs NetBird's management store, activity events, and embedded IdP.
- **nginx-proxy-manager** (external, in `../nginx-proxy-manager`) — terminates TLS, proxies HTTP/gRPC/WebSocket to the NetBird container on `127.0.0.1:8081`.

Peer login flow: log in to the embedded Dex IdP with the initial owner account, then federate to Authentik as an external OIDC provider via the Dashboard UI.

## Prerequisites

- Docker + Docker Compose
- A public domain pointed at this host (e.g. `netbird.mgsfoxhound.net`)
- nginx-proxy-manager (or another reverse proxy) with HTTP/2 support
- Firewall port-forward: **UDP 3478** direct to this host (STUN traffic cannot be proxied)
- Optional but recommended: reachable Authentik instance for SSO

## Quick Start

1. Copy the example files and edit them:

   ```bash
   cp .env.example .env
   cp config.yaml.example config.yaml
   ```

2. Edit `.env` — at minimum set `DOCKER_DATA_PATH` and `POSTGRES_PASSWORD`.

3. Edit `config.yaml` — replace every `REPLACE_...` placeholder. Generate secrets:

   ```bash
   openssl rand -base64 32   # authSecret (relay)
   openssl rand -base64 32   # store.encryptionKey
   ```

   Set `store.dsn` / `activityStore.dsn` / `authStore.dsn` to include the same `POSTGRES_PASSWORD` you put in `.env`. Set the `auth.owner.email` and `auth.owner.password` — this becomes your first admin. `auth.issuer` should be your public NetBird URL.

4. Create the data directories:

   ```bash
   mkdir -p ${DOCKER_DATA_PATH}/{data,postgres}
   ```

5. Start the stack:

   ```bash
   docker compose up -d
   ```

6. Set up the reverse proxy (see [Nginx-Proxy-Manager](#nginx-proxy-manager) below).

7. Open `https://netbird.mgsfoxhound.net/` in a browser. Log in with the owner credentials from `config.yaml`.

8. (Recommended) Federate to Authentik — see [Authentik Federation](#authentik-federation).

## Configuration

Compose-level knobs (paths, image tags, DB credentials, port mappings) live in `.env`. NetBird-specific configuration (public URL, auth, store DSN, secrets) lives in `config.yaml`. NetBird does NOT expand `${VAR}` inside `config.yaml`, so values are literal.

### Volume Paths

- `DOCKER_DATA_PATH` — base directory on the host. Compose appends:
  - `${DOCKER_DATA_PATH}/data` → `/var/lib/netbird` (server data, IdP DB, keys)
  - `${DOCKER_DATA_PATH}/postgres` → `/var/lib/postgresql/data` (database)

### Image Tags

- `NETBIRD_TAG` — defaults to `0.75.0`. Track the latest at [Docker Hub](https://hub.docker.com/r/netbirdio/netbird-server/tags).
- `POSTGRES_TAG` — defaults to `17`.

### Ports

- `NETBIRD_HTTP_PORT` (default `8081`) — bound to `127.0.0.1`, consumed by NPM.
- `NETBIRD_STUN_PORT` (default `3478`) — UDP, **must be publicly reachable**.
- `NETBIRD_METRICS_PORT` (default `9090`) — Prometheus, localhost only.

## Nginx-Proxy-Manager

NetBird speaks HTTP, gRPC, and WebSockets on the same backend port. NPM must proxy all three protocols.

1. In NPM, add a new **Proxy Host**:
   - Domain: `netbird.mgsfoxhound.net`
   - Scheme: `http`
   - Forward hostname/IP: `127.0.0.1` (or the host's docker0 IP if NPM is in a different network)
   - Forward port: `8081` (or whatever you set `NETBIRD_HTTP_PORT` to)
   - Enable **Block Common Exploits**
   - Enable **Websockets Support**

2. In the **SSL** tab: request a Let's Encrypt cert, enable **HTTP/2 Support**, enable **Force SSL**.

3. In the **Advanced** tab, paste the gRPC/WebSocket location blocks:

   ```nginx
   # gRPC — management API + signal exchange + relay authorization
   location /management.ManagementService/ {
     grpc_pass grpc://127.0.0.1:8081;
     grpc_read_timeout 1d;
     grpc_send_timeout 1d;
     grpc_socket_keepalive on;
   }

   location /signalexchange.SignalExchange/ {
     grpc_pass grpc://127.0.0.1:8081;
     grpc_read_timeout 1d;
     grpc_send_timeout 1d;
     grpc_socket_keepalive on;
   }

   location /management.ProxyService/ {
     grpc_pass grpc://127.0.0.1:8081;
     grpc_read_timeout 1d;
     grpc_send_timeout 1d;
     grpc_socket_keepalive on;
   }

   # WebSocket — relay traffic
   location /relay {
     proxy_pass http://127.0.0.1:8081;
     proxy_http_version 1.1;
     proxy_set_header Upgrade $http_upgrade;
     proxy_set_header Connection "upgrade";
     proxy_set_header Host $host;
     proxy_read_timeout 1d;
     proxy_send_timeout 1d;
   }

   # WebSocket — dashboard silent-auth
   location /ws-proxy/ {
     proxy_pass http://127.0.0.1:8081;
     proxy_http_version 1.1;
     proxy_set_header Upgrade $http_upgrade;
     proxy_set_header Connection "upgrade";
     proxy_set_header Host $host;
     proxy_read_timeout 1d;
   }

   client_header_timeout 1d;
   client_body_timeout 1d;
   ```

4. Save. Test by opening `https://netbird.mgsfoxhound.net/` — you should get the login page.

## Firewall

The STUN server must be reachable from the public internet or peers behind symmetric NAT can't connect:

- Port-forward **UDP 3478** → this host on your router/firewall.
- Do NOT try to proxy this through NPM. UDP + STUN protocol don't map to HTTP.

The TCP relay path (WebSocket over HTTPS) goes through 443 via NPM — no extra port-forward needed.

## Authentik Federation

Once you're logged into NetBird as the local owner, add Authentik as a federated identity provider so you can sign in with SSO.

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

Save. The next time you sign out, the login page shows a "Sign in with Authentik" button.

## Usage

### Starting the service

```bash
docker compose up -d
```

### Viewing logs

```bash
docker compose logs -f netbird
```

### Stopping

```bash
docker compose down
```

### Upgrading

Bump `NETBIRD_TAG` in `.env`, then:

```bash
docker compose pull
docker compose up -d
```

Read the [NetBird release notes](https://github.com/netbirdio/netbird/releases) before crossing a minor version.

### Backup

Stop the stack (or use `pg_dump` for a hot Postgres backup) and archive both volumes:

```bash
docker compose down
tar -czf netbird-backup-$(date +%Y%m%d).tar.gz -C ${DOCKER_DATA_PATH} data postgres
docker compose up -d
```

Hot Postgres backup (no downtime):

```bash
docker compose exec -T db pg_dump -U ${POSTGRES_USER} ${POSTGRES_DB} \
  | gzip > netbird-db-$(date +%Y%m%d).sql.gz
```

Also keep a copy of `config.yaml` and `.env` — they contain the encryption keys needed to read the backup.

## Peer Onboarding

Install the NetBird client on each device: <https://docs.netbird.io/how-to/installation>.

Point the client at your self-hosted server:

```bash
netbird up --management-url https://netbird.mgsfoxhound.net
```

The client opens a browser for OIDC login. After signing in (with the local owner or via Authentik), the peer appears in the dashboard.

## Security Notes

- Never commit `.env` or `config.yaml` — both contain secrets.
- The embedded IdP owner is a real admin — set a strong password and enable TOTP after first login.
- Rotate `authSecret`, `encryptionKey`, and `POSTGRES_PASSWORD` on a schedule; the encryption key rotation requires a data migration — plan before changing.
- Put HTTPS in front (NPM) for the dashboard/management/signal traffic. STUN over UDP is unencrypted by design.

## Troubleshooting

### Peers can't connect / "connection refused"

- Check UDP 3478 is open on the firewall and reaches this host.
- Confirm NPM is proxying gRPC (check the Advanced tab — the location blocks above are non-optional).
- `docker compose logs netbird` — look for `starting management server` and `starting signal server`.

### "server.authSecret is required when running local relay"

`authSecret` in `config.yaml` is empty. Set it to a random 32-byte string.

### Container restarts / "failed to connect to postgres"

- Confirm `POSTGRES_PASSWORD` in `.env` matches the password embedded in each DSN inside `config.yaml`.
- `docker compose ps` — is the `db` container healthy?
- `docker compose logs db`

### Dashboard loads but login page never appears

The dashboard's OIDC discovery is hitting the wrong URL. Confirm `server.auth.issuer` and `server.exposedAddress` in `config.yaml` both point to `https://netbird.mgsfoxhound.net` (no trailing slash).

### `context deadline exceeded` on peer connect

The client is trying to reach management/signal on gRPC but the reverse proxy isn't forwarding HTTP/2. In NPM, confirm **HTTP/2 Support** is checked on the SSL tab and the `grpc_pass` blocks are present in Advanced.

## Documentation

- [NetBird Docs](https://docs.netbird.io/)
- [Self-Hosting Quickstart](https://docs.netbird.io/selfhosted/selfhosted-quickstart)
- [Reverse Proxy Setup](https://docs.netbird.io/selfhosted/reverse-proxy)
- [Authentik + NetBird (official)](https://docs.netbird.io/selfhosted/identity-providers/authentik)
- [Combined config.yaml.example](https://github.com/netbirdio/netbird/blob/main/combined/config.yaml.example)

## License

This configuration is provided as-is. NetBird is licensed under the BSD-3-Clause License.
