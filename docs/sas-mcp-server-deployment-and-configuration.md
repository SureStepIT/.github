# SAS MCP Server Deployment and Configuration

## Purpose

This article documents how to deploy and configure the SAS MCP Server for a shared team environment. It is intentionally scoped to the MCP server only. It does not cover base SAS Viya deployment.

The SAS MCP Server is a self-hosted MCP server from `sassoftware/sas-mcp-server` that connects MCP clients to a SAS Viya environment. Users authenticate through SAS Logon with OAuth2 PKCE, and tool calls run under each user's own Viya permissions.

Reference: `https://github.com/sassoftware/sas-mcp-server`

## Recommended Deployment Model

Deploy one SAS MCP Server instance per SAS Viya environment.

Use one shared HTTPS MCP endpoint for all users of the same Viya environment. Do not share one MCP server across unrelated client Viya environments, because each MCP server is configured with one Viya endpoint, one OAuth client registration, one callback URL, one signing key, and one tool-tier policy.

Recommended shape:

```text
Client A dev Viya  -> Client A dev MCP server
Client A prod Viya -> Client A prod MCP server
Client B dev Viya  -> Client B dev MCP server
Client B prod Viya -> Client B prod MCP server
```

## Required Inputs

Before deploying the MCP server, collect:

- Target Viya URL, for example `https://client-a-prod-viya.company.com`
- Public MCP base URL, for example `https://client-a-prod-mcp.company.com`
- Viya admin credentials for registering the OAuth client, or a SAS admin who can register it manually
- TLS certificate path for the public MCP endpoint
- Docker-capable host for the MCP container
- Chosen MCP tool tiers, usually `0-3` to start

## Shared Team HTTPS Setup Order

1. Confirm the target Viya environment is reachable and users can log in.
2. Choose the public MCP hostname for this Viya environment.
3. Create DNS for the MCP hostname.
4. Put TLS in front of the MCP hostname with a reverse proxy, load balancer, or ingress.
5. Register a Viya OAuth client for the MCP server using the public HTTPS callback.
6. Create the MCP server environment file on the Docker host.
7. Start the SAS MCP Server container.
8. Configure team MCP clients with the public MCP URL.
9. Trigger a real MCP tool call and complete the browser login.
10. Verify tool access, usually with `list_compute_contexts`.

For shared team usage, users should call:

```text
https://<mcp-host>/mcp
```

Users should not need SSH tunnels for the real deployment model.

## Viya OAuth Client Registration

Each target Viya environment needs an OAuth client registration for the MCP server.

Register this client in SAS Logon:

```text
client_id: sas-mcp
redirect_uri: https://<mcp-host>/auth/callback
grant types: authorization_code, refresh_token
scope: openid
allowpublic: true
autoapprove: true
```

The redirect URI must exactly match the MCP server's public callback URL. If the MCP hostname changes, register or update the OAuth client with the new callback.

The registration endpoint is:

```text
POST https://<viya-host>/SASLogon/oauth/clients
```

If password-grant token retrieval is enabled for admin use, the OAuth client can be registered with an admin bearer token. If password grant is disabled, use the SAS-provided helper or have a SAS administrator register the client through the supported administrative path.

## MCP Server Environment File

Create an env file on the Docker host, for example:

```text
/home/ec2-user/sas-mcp-server.env
```

Recommended shared HTTPS shape:

```env
VIYA_ENDPOINT=https://<viya-host>
CLIENT_ID=sas-mcp
HOST_PORT=8134
MCP_BASE_URL=https://<mcp-host>
MCP_TIERS=0-3
SSL_VERIFY=true
MCP_SIGNING_KEY=<generated-long-random-secret>
```

Important notes:

- `VIYA_ENDPOINT` is the target SAS Viya URL.
- `MCP_BASE_URL` is the public HTTPS URL for the MCP server, without `/mcp`.
- `CLIENT_ID` must match the Viya OAuth client registration.
- `MCP_SIGNING_KEY` must be unique per MCP server instance and treated as a secret.
- `SSL_VERIFY=true` should be used for real deployments with valid certificates.
- Use `SSL_VERIFY=false` only for temporary testing with self-signed or temporary Viya certificates.

Do not commit the real env file or signing key.

## Start the Container

Pull and start the MCP server container:

```bash
docker pull ghcr.io/sassoftware/sas-mcp-server:latest

docker rm -f sas-mcp-server || true

docker run -d \
  --name sas-mcp-server \
  --restart unless-stopped \
  --env-file /home/ec2-user/sas-mcp-server.env \
  -p 127.0.0.1:8134:8134 \
  ghcr.io/sassoftware/sas-mcp-server:latest
```

This binds the container to local port `8134` on the Docker host. Put a reverse proxy, load balancer, or ingress in front of it so team users reach it at:

```text
https://<mcp-host>/mcp
```

Prefer TLS termination in front of the container rather than exposing plain HTTP to users.

If the MCP host must receive traffic directly and there is no local reverse proxy, adjust the binding deliberately and restrict access with security groups or firewall rules:

```bash
-p 8134:8134
```

## Prompt-Driven Deployment Helper

A small wrapper script can automate the same deployment flow by prompting for:

- Target Viya URL
- Public MCP base URL
- OAuth client ID
- MCP tool tiers
- TLS verification behavior
- Container port and bind address
- Container env-file path

The helper should write a Docker env file, generate a unique `MCP_SIGNING_KEY`, optionally attempt OAuth client registration, and start or replace the `sas-mcp-server` container. Keep the generated answers and env file on the deployment host only; they contain environment-specific values and secrets.

## MCP Client Configuration

Configure HTTP-capable MCP clients with the public URL:

```json
{
  "servers": {
    "sas-viya-mcp": {
      "url": "https://<mcp-host>/mcp",
      "type": "http"
    }
  }
}
```

The first real tool call should start the browser-based SAS Logon flow. Sign in with a Viya user that has permissions for the requested Viya tools.

## Health Checks

Check status on the Docker host:

```bash
docker ps --filter name=sas-mcp-server
docker logs --tail 80 sas-mcp-server
curl -i http://127.0.0.1:8134/mcp
```

The unauthenticated local `/mcp` check should return `401 Unauthorized` with a `www-authenticate` header. That is expected and means the MCP endpoint is alive and requiring authentication.

From a workstation, verify the public endpoint:

```bash
curl -i https://<mcp-host>/mcp
```

That should also return an authentication challenge before login.

## Tool Tier Guidance

Start with:

```env
MCP_TIERS=0-3
```

This keeps the exposed tool list focused on common Viya workflows such as compute, data discovery, data operations/files, and reports. Expanding to all tiers is possible, but the target Viya environment may not have every backing service or product enabled.

## Operational Notes

- Use one unique `MCP_SIGNING_KEY` per MCP server instance.
- Keep the MCP server behind HTTPS for shared/team deployments.
- Keep `SSL_VERIFY=true` once Viya has a valid certificate chain.
- If the public MCP hostname changes, update both `MCP_BASE_URL` and the Viya OAuth callback registration.
- If OAuth registration already exists, confirm its redirect URI matches the current public callback.
- Users authenticate individually; do not run the MCP server with a shared Viya user password.

## Minimal Troubleshooting

If users cannot authenticate:

1. Confirm the OAuth redirect URI is exactly `https://<mcp-host>/auth/callback`.
2. Confirm `MCP_BASE_URL=https://<mcp-host>` in the env file.
3. Confirm the browser can reach `https://<mcp-host>/auth/callback` through the public endpoint.
4. Check `docker logs --tail 80 sas-mcp-server`.
5. Confirm the Viya endpoint certificate is valid, or temporarily test with `SSL_VERIFY=false` only in non-production.

If the MCP client cannot list tools:

1. Confirm the client URL is `https://<mcp-host>/mcp`.
2. Confirm the public endpoint returns an auth challenge before login.
3. Confirm the MCP client preserves the session headers required by streamable HTTP.
4. Confirm the signed-in Viya user has permissions for the requested tools.
