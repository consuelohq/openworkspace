# openworkspace

openworkspace is designed to be used as a chatgpt app, but works with any cloud MCP client. it gives access to a local workspace: filesystem, terminal, git, github, memory, handoffs/compaction, and agent spawning.

a personal AI workspace that connects chatgpt to your own machine. no extra hardware, no complex setup — just your computer, your tools, your files, secured by cloudflare.

## why openworkspace

- **built for long-running tasks** — work for long periods across multiple tool calls and compaction guidance.
- **separate 5-hour rate-limit** — chatgpt apps don't count against the codex 5-hour rate-limited message cap.
- **delightful interface** — runs inside the chatgpt desktop app. integrates with your existing chatgpt setup, memories, and conversation history.
- **memory system with semantic search** — supabase-backed memory that integrates with local coding agents via vector search.
- **real host-machine sandbox** — not a container mock. full terminal, filesystem, git, and your actual dev tools.
- **command guardrails** — destructive commands are blocked. `rm` gets rewritten to `trash`. every command is audit-logged.
- **progressive skill loading** — steering file + compact skill index loaded upfront. full skill docs loaded on demand. minimal tokens wasted.
- **optional langsmith tracing** — every tool call traced for observability.
- **cloudflare tunnel + WAF** — safe exposure to chatgpt connectors. only openai's published IP ranges can reach your endpoint.

## quick start

paste this into your coding agent:

```
read https://raw.githubusercontent.com/consuelohq/openworkspace/main/prompts/setup.md and follow it to set up openworkspace on my machine.
```

or manually:

```bash
git clone https://github.com/consuelohq/openworkspace.git
cd openworkspace
bash setup.sh
```

`setup.sh` creates a virtualenv, installs dependencies, copies `.env.example` → `.env`, copies `STEERING.example.md` → `STEERING.md`, and generates launchd/cloudflare helper files under `scripts/generated/`.

## tools

| tool | what it does |
|------|-------------|
| `get_steering` | loads your steering file + skill index |
| `brain_search` | keyword search over memories |
| `brain_vector_search` | semantic search (nvidia embeddings) |
| `brain_remember` | save a memory |
| `brain_get_memory` | fetch a memory by id |
| `brain_list_skills` | list available skills |
| `brain_get_skill` | load full skill doc by name |
| `sandbox_exec` | run bash on your machine (with guardrails) |
| `sandbox_read_file` | read a file |
| `sandbox_write_file` | write a file |
| `sandbox_list_files` | list directory contents |
| `github_get_file` | read a file from a github branch |
| `github_get_pr` | get a PR by number |
| `github_list_prs` | list open PRs |
| `github_push_files` | push files to a branch (blob→tree→commit→ref) |
| `invoke_opencode` | spawn an opencode agent in tmux |
| `invoke_kiro` | spawn a kiro agent in tmux |
| `slack_post` | post to a slack channel |
| `handoff_save` | save conversation context |
| `handoff_load` | load previous context |

## guardrails

the sandbox blocks destructive commands automatically:

- `rm -rf /`, `rm -rf ~`, `rm -rf *` → **blocked**
- `git push --force` → **blocked**
- `npm publish`, `shutdown`, `reboot`, `mkfs` → **blocked**
- `rm file.txt` → **rewritten to `trash file.txt`**
- writes to `/etc/`, `/System/`, `~/.ssh/` → **blocked**

every command is logged to `/tmp/sandbox-audit.jsonl`.

## security

- **oauth authentication** — chatgpt authenticates via oauth2 with client credentials. only your chatgpt app has the client secret. no one else can connect, even from openai's IP range.
- **cloudflare WAF** — only openai's published IP ranges can reach the MCP endpoint. oauth and well-known paths are exempted so the auth flow works. ([chatgpt-connectors.json](https://openai.com/chatgpt-connectors.json))
- **cloudflare tunnel** — server never exposed directly to the internet
- **bearer token validation** — every MCP request requires a valid bearer token, issued through the oauth flow
- **guardrails** — destructive commands blocked at the sandbox layer
- **audit log** — every command logged with timestamp and result

## chatgpt setup guide

### 1. create the app

go to [chatgpt.com/apps](https://chatgpt.com/apps) (turn on developer mode if you haven't). create a new app → add MCP connector → point it at your openworkspace URL.

chatgpt app docs: [platform.openai.com/docs/guides/tools-connectors-mcp](https://platform.openai.com/docs/guides/tools-connectors-mcp)

### 2. add custom instructions

go to chatgpt → settings → personalization → custom instructions. add this:

```
you have an app connected called "workspace"

at the start of EVERY conversation, before responding to any message:
1. call workspace's get_steering tool
2. read the full response — it contains your identity, rules, repo structure, and how to use all your tools
3. follow everything in that document for the rest of the conversation

this is not optional. do not respond to the user until you have called get_steering and internalized the result.
even if the user's first message seems simple, call get_steering first. no exceptions.
```

### 3. nvidia API key (free, recommended)

get a free API key at [build.nvidia.com/settings/api-keys](https://build.nvidia.com/settings/api-keys). this powers `brain_vector_search` — semantic search over your memories and chat history. without it, you still have keyword search (`brain_search`), but vector search finds conceptually related content even when the exact words don't match.

we recommend securing your nvidia key the same way you secure the workspace endpoint: restrict API access to known IPs via cloudflare or tailscale. the key itself is only used server-side (never sent to chatgpt), but defense in depth is the right posture.

## license

Apache 2.0 — see [LICENSE](LICENSE)

