# setup prompt — paste this into your coding agent

you are setting up openworkspace — a local MCP server that gives chatgpt (or any MCP client) real access to a dev machine. follow these steps in order. stop and ask the human when noted.

## step 1: detect intent

ask the human:
1. what repo will this workspace manage? (e.g. `owner/repo`, or a local path)
2. local-only, or exposed via cloudflare? (cloudflare = chatgpt desktop app can reach it from the internet)
3. enable memory? (requires a free supabase project + free nvidia API key for semantic search)
4. enable langsmith tracing? (optional observability on every tool call)

## step 2: clone and bootstrap

```bash
git clone https://github.com/consuelohq/openworkspace.git
cd openworkspace
bash setup.sh
```

`setup.sh` handles: venv creation, pip install, `.env` from template, `STEERING.md` from template, and generates helper files under `scripts/generated/`.

## step 3: fill .env

open `.env` and fill in values. **stop and ask the human** for any credentials you don't have.

required:
- `PORT` — default 8850
- `WORKSPACE_DIR` — absolute path to the repo this workspace manages
- `GITHUB_TOKEN` — github personal access token (repo scope)

for memory (recommended):
- `SUPABASE_URL` + `SUPABASE_KEY` — create a free project at https://supabase.com
- `NVIDIA_API_KEY` — free at https://build.nvidia.com/settings/api-keys (powers semantic vector search)

optional:
- `GITHUB_REPO` — owner/repo format, auto-detected from git remote if not set
- `SLACK_WEBHOOK_URL` — for slack notifications
- `LANGCHAIN_API_KEY` — for langsmith tracing (https://smith.langchain.com)
- `OAUTH_CLIENT_ID` — client ID for chatgpt oauth (default: openworkspace)
- `OAUTH_CLIENT_SECRET` — client secret for chatgpt oauth (generate a random string)

## step 4: supabase setup (if memory enabled)

run this SQL in the supabase SQL editor:

```sql
create extension if not exists vector;

create table if not exists memories (
  id uuid primary key default gen_random_uuid(),
  title text,
  content text,
  category text default 'observation',
  source text default 'manual',
  status text default 'active',
  priority text default 'normal',
  agent_id text,
  tags text,
  embedding vector(1024),
  fts tsvector generated always as (to_tsvector('english', coalesce(title,'') || ' ' || coalesce(content,''))) stored,
  created_at timestamptz default now()
);

create index on memories using ivfflat (embedding vector_cosine_ops) with (lists = 100);
create index on memories using gin (fts);
```

## step 5: customize STEERING.md

`STEERING.md` is loaded by `get_steering` at the start of every conversation. edit it to match the human's project — identity, rules, repo structure, workflows.

## step 6: start and verify

```bash
.venv/bin/python3 server.py
```

verify: `curl http://localhost:8850/health` → `{"status":"ok","tools":20}`

## step 7: cloudflare setup (if chosen)

```bash
brew install cloudflared
cloudflared tunnel login
cloudflared tunnel create workspace
cloudflared tunnel route dns workspace your-subdomain.yourdomain.com
```

add to `~/.cloudflared/config.yml`:
```yaml
ingress:
  - hostname: your-subdomain.yourdomain.com
    service: http://localhost:8850
  - service: http_status:404
```

**secure with WAF** — create a custom rule that only allows chatgpt's IPs. openai publishes their ranges at https://openai.com/chatgpt-connectors.json. the rule blocks all traffic except those IPs + your tailscale range. this means only chatgpt and you can reach the endpoint. everyone else gets 403.

## step 8: create chatgpt app

1. go to https://chatgpt.com/apps (enable developer mode if needed)
2. create a new app → add MCP connector
3. set the URL to `https://your-subdomain.yourdomain.com/mcp`
4. authentication: select **Mixed** or **OAuth**
5. client ID: the value of `OAUTH_CLIENT_ID` from your `.env`
6. client secret: the value of `OAUTH_CLIENT_SECRET` from your `.env`
7. token endpoint auth method: `client_secret_post`

the server has built-in oauth endpoints. chatgpt will discover them automatically via `/.well-known/oauth-authorization-server`, redirect through the auth flow, and get a bearer token. every MCP request after that is authenticated.

**important:** your cloudflare WAF rule must exempt `/oauth/` and `/.well-known/` paths so the auth redirect works through your browser. the MCP endpoint stays locked to openai IPs only.

docs: https://platform.openai.com/docs/guides/tools-connectors-mcp

then add custom instructions in chatgpt → settings → personalization:

```
you have an app connected called "workspace"

at the start of EVERY conversation, before responding to any message:
1. call workspace's get_steering tool
2. read the full response — it contains your identity, rules, repo structure, and how to use all your tools
3. follow everything in that document for the rest of the conversation

this is not optional. do not respond to the user until you have called get_steering and internalized the result.
even if the user's first message seems simple, call get_steering first. no exceptions.
```

## step 9: validate

- [ ] `curl http://localhost:8850/health` returns ok with 20 tools
- [ ] `get_steering` returns your customized STEERING.md (not the template)
- [ ] `sandbox_exec("pwd")` returns WORKSPACE_DIR
- [ ] `sandbox_exec("which rg")` returns a path (tools in PATH)
- [ ] `sandbox_exec("rm -rf /")` returns BLOCKED (guardrails working)
- [ ] `brain_search("test")` returns results or empty array (not an error)
- [ ] if cloudflare: external URL returns 403 from your IP (WAF working)
- [ ] if cloudflare: chatgpt can call tools through the connector

## definition of done

1. server responds on /health with 20 tools
2. get_steering returns the human's custom steering file
3. sandbox has the full toolchain in PATH
4. guardrails block destructive commands
5. memory works (if enabled)
6. chatgpt (or chosen client) can connect and call tools
