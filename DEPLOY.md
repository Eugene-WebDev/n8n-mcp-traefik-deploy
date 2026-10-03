# Deploy czlonkowski/n8n-mcp on VPS <your-server>

Patched compose adds one service (`n8n-mcp`) on the existing `web` network, behind Traefik with the same `myresolver` cert pattern n8n uses. Endpoint: `https://n8n-mcp.example.com/mcp`. Bearer auth.

## 1. DNS (do first, cert won't issue without it)
At your DNS provider (Cloudflare/Hostinger DNS):

- Type: `A`
- Name: `n8n-mcp`
- Value: `45.93.138.64`
- Proxy: OFF if on Cloudflare (Traefik does HTTP-01 cert challenge directly)
- TTL: 300

Verify:
```bash
dig +short n8n-mcp.example.com   # should return 45.93.138.64
```

## 2. n8n API key
`https://n8n.example.com` → Settings → n8n API → "Create new key" → copy the key.

## 3. SSH to VPS and apply
```bash
ssh <user>@<your-server>
cd ~/n8n-compose

# Backup current compose
cp compose.yaml compose.yaml.bak.$(date +%Y%m%d-%H%M%S)

# Backup current .env
cp .env .env.bak.$(date +%Y%m%d-%H%M%S)

# Append env vars (paste both lines, replacing the placeholders)
echo "MCP_AUTH_TOKEN=$(openssl rand -hex 32)" >> .env
echo "N8N_API_KEY=PASTE_THE_N8N_API_KEY_HERE" >> .env

# Replace compose.yaml with the patched version
# (paste the content of compose.patched.yaml from this folder, OR scp it up)
nano compose.yaml
```

Or in one shot, from your laptop (faster):
```bash
scp ~/Documents/vscode/n8n-mcp-deploy/compose.patched.yaml \
    <user>@<your-server>:~/n8n-compose/compose.yaml.new

ssh <user>@<your-server> "cd ~/n8n-compose && \
  cp compose.yaml compose.yaml.bak.\$(date +%Y%m%d-%H%M%S) && \
  mv compose.yaml.new compose.yaml"
```

## 4. Pull image + bring it up
```bash
docker compose pull n8n-mcp
docker compose up -d n8n-mcp
docker compose logs -f n8n-mcp   # watch for "MCP server listening on :3000"
```

If it crash-loops, check:
```bash
docker compose logs n8n-mcp --tail 50
```
Most common: missing env var, or `N8N_API_URL` unreachable from container.

## 5. Verify cert + endpoint
Wait ~30–60s for Let's Encrypt to issue. Then:
```bash
# Health (no auth needed)
curl -i https://n8n-mcp.example.com/health

# MCP endpoint with bearer (replace TOKEN with value from .env)
TOKEN=$(grep ^MCP_AUTH_TOKEN= .env | cut -d= -f2)
curl -i -H "Authorization: Bearer $TOKEN" \
  https://n8n-mcp.example.com/mcp
```

Expect 200 on `/health`. `/mcp` without proper MCP framing may 400 — that's fine, it confirms auth passed.

## 6. Add to Claude.ai connectors
- claude.ai → Settings → Connectors → Add custom connector
- URL: `https://n8n-mcp.example.com/mcp`
- Auth: `Bearer <MCP_AUTH_TOKEN>` (from `.env`)
- Save → enable in this project

Once enabled, the toolset flips to `n8n_get_workflow`, `n8n_update_partial_workflow`, `patchNodeField`, `updateNode` — surgical editing returns.

## Rollback
```bash
ssh <user>@<your-server>
cd ~/n8n-compose
docker compose stop n8n-mcp
docker compose rm -f n8n-mcp
mv compose.yaml.bak.<timestamp> compose.yaml
mv .env.bak.<timestamp> .env
docker compose up -d
```
