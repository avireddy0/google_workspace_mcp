# Fork Notes

This repository is a fork of [`taylorwilsdon/google_workspace_mcp`](https://github.com/taylorwilsdon/google_workspace_mcp), maintained by Envision Construction for deployment behind Open WebUI on Google Cloud Run.

## Why this fork exists

Upstream `google_workspace_mcp` requires an authenticated MCP handshake. Open WebUI's MCP client cannot complete that handshake against a remote streamable-HTTP server because it has no way to inject a bearer token before the initial `initialize` call. The fork applies a single patch that lets the handshake proceed unauthenticated while still enforcing auth at the tool-call boundary.

Auth enforcement after the patch:
- `/mcp` handshake: unauthenticated (no `RequireAuthMiddleware`).
- OAuth routes (`/.well-known/*`, `/authorize`, `/token`, `/oauth2/register`, `/register`): mounted manually.
- Tool calls: gated by `AuthInfoMiddleware` — every `tools/call` still requires a valid bearer token.
- Sessions: `MCPSessionMiddleware` binds Starlette sessions normally.

## Fork-only commit

| SHA | Subject |
|-----|---------|
| `39b8772` | `fix: allow unauthenticated MCP handshake for Open WebUI compatibility` |

This commit overrides `FastMCP.http_app()` to call `create_streamable_http_app(auth=None, ...)` and manually attaches the OAuth routes plus `BearerAuth` / `AuthInfo` middleware. It also adds a `/register` alias proxying to `/oauth2/register` for clients that expect DCR at the root path.

All other changes track upstream `main`. Periodically rebase `fork/main` onto `origin/main` and reapply `39b8772` if conflicts arise.

## Deployment target

GCP project: `flow-os-1769675656`
Region: `us-central1`
Service: `workspace-mcp` (Cloud Run)
Image: `us-central1-docker.pkg.dev/flow-os-1769675656/flow-os/workspace-mcp:latest`
Service account: `open-webui-sa@flow-os-1769675656.iam.gserviceaccount.com`

Tools subset enabled via `TOOLS=gmail calendar drive` (other Workspace surfaces are intentionally disabled to keep the tool list small for Open WebUI). Runs in `MCP_SINGLE_USER_MODE=1` because the only consumer is the Open WebUI service account.

Credentials are mounted via GCS Fuse from bucket `workspace-mcp-creds-flow-os` at `/app/store_creds`.

## Deploy

The canonical service spec is checked in at [`cloudrun-service.yaml`](./cloudrun-service.yaml). Apply with:

```bash
gcloud run services replace cloudrun-service.yaml \
  --project=flow-os-1769675656 \
  --region=us-central1
```

Or, equivalently, redeploy with explicit flags after building a new image:

```bash
gcloud run deploy workspace-mcp \
  --project=flow-os-1769675656 \
  --region=us-central1 \
  --image=us-central1-docker.pkg.dev/flow-os-1769675656/flow-os/workspace-mcp:latest \
  --service-account=open-webui-sa@flow-os-1769675656.iam.gserviceaccount.com \
  --update-secrets=GOOGLE_OAUTH_CLIENT_ID=google-oauth-client-id:latest,GOOGLE_OAUTH_CLIENT_SECRET=google-oauth-client-secret:latest \
  --set-env-vars=TRANSPORT=streamable-http,TOOLS="gmail calendar drive",WORKSPACE_MCP_CREDENTIALS_DIR=/app/store_creds,MCP_SINGLE_USER_MODE=1,WORKSPACE_EXTERNAL_URL=https://workspace-mcp-210087613384.us-central1.run.app
```

External URL: `https://workspace-mcp-210087613384.us-central1.run.app`
