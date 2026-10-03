# n8n-mcp self-hosted deploy (Docker Compose + Traefik)

Patched compose and deploy guide for running [czlonkowski/n8n-mcp](https://github.com/czlonkowski/n8n-mcp)
next to a self-hosted n8n behind Traefik, so an MCP client (Claude Code, Claude Desktop) can manage
workflows over the n8n REST API.

- `compose.patched.yaml` — n8n-mcp service wired into an existing n8n + Traefik stack (TLS, routing)
- `env.additions` — the two variables to add to your `.env` (placeholders only)
- `DEPLOY.md` — step-by-step deploy, verification and key rotation

Note the two different secrets: `MCP_AUTH_TOKEN` authenticates the MCP client to n8n-mcp;
`N8N_API_KEY` authenticates n8n-mcp to n8n. Mixing them up is the most common setup failure.

The n8n-mcp server itself is authored by czlonkowski; this repo is the deployment configuration.
