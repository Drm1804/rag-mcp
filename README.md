# RAG-mcp

Coolify service template for deploying
[Umbrella MetaMCP](https://github.com/Umbrella-IT-Group/metamcp) with PostgreSQL and a
Cloudflare Tunnel connector.

The stack is intentionally based on the official
[MetaMCP template from the Coolify catalog](https://github.com/coollabsio/coolify/blob/main/templates/compose/metamcp.yaml).
The application is built from the maintained Umbrella distribution using its own Dockerfile; the
deployment shape and Coolify variable conventions stay aligned with the catalog template.

## Deploy in Coolify

1. Create a new resource from this Git repository.
2. Select **Docker Compose** as the build pack.
3. Set the Compose location to `/docker-compose.yaml`.
4. Add `METAMCP_PUBLIC_URL` with the public HTTPS origin, for example
   `https://metamcp.example.com`.
5. Add `TUNNEL_TOKEN` as a secret variable.
6. Deploy the resource. Do not assign a Coolify proxy domain to the `app` service.

Coolify generates and persists these credentials from the Compose file:

- `SERVICE_USER_POSTGRES` — PostgreSQL user;
- `SERVICE_PASSWORD_POSTGRES` — PostgreSQL password;
- `SERVICE_PASSWORD_AUTH` — MetaMCP authentication secret.

The following values must be supplied manually:

- `METAMCP_PUBLIC_URL` — the exact public HTTPS origin exposed through the tunnel;
- `TUNNEL_TOKEN` — token of a remotely-managed tunnel; mark it as secret. Keep this standard
  variable name: `cloudflared` redacts it from its environment log.

In the Cloudflare dashboard, configure the tunnel's published application route as follows:

| Setting | Value |
| --- | --- |
| Public hostname | Hostname from `METAMCP_PUBLIC_URL` |
| Service type | HTTP |
| Service URL | `http://app:12008` |

`app` is the Compose service name resolvable from `cloudflared` inside the stack network. No host
port needs to be published.

The PostgreSQL data is stored in the `postgres_data` named volume. Do not rotate generated
passwords or remove the volume unless the corresponding data migration or recovery procedure is
planned.

## Optional variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `POSTGRES_HOST` | `postgres` | PostgreSQL service host |
| `POSTGRES_PORT` | `5432` | PostgreSQL service port |
| `POSTGRES_DB` | `metamcp_db` | PostgreSQL database name |
| `TRANSFORM_LOCALHOST_TO_DOCKER_INTERNAL` | `true` | Rewrites localhost MCP targets for Docker |

MetaMCP application data such as MCP servers, namespaces, endpoints, users, and API keys is not
declared by this deployment template.

## Umbrella source version

The Compose build context is pinned to an exact commit of the public `umbrella` branch. Coolify
builds that commit with Umbrella's own `Dockerfile` for the architecture of its Docker host. This
avoids both the private/inaccessible GHCR package and the published image's `linux/amd64` limit.

To upgrade Umbrella MetaMCP, verify the desired commit belongs to the `umbrella` branch, update the
SHA in `docker-compose.yaml`, and redeploy. Do not replace the SHA with the mutable branch name or
`latest` tag.

## Local validation

Provide placeholder values for Coolify-generated variables and render the Compose model:

```bash
METAMCP_PUBLIC_URL=http://localhost:12008 \
SERVICE_USER_POSTGRES=metamcp \
SERVICE_PASSWORD_POSTGRES=local-only-password \
SERVICE_PASSWORD_AUTH=local-only-auth-secret \
TUNNEL_TOKEN=validation-only-token \
docker compose -f docker-compose.yaml config
```

Architecture decision: [ADR-0001](docs/ADR/ADR-0001-maintain-coolify-template-for-umbrella-metamcp.md).
