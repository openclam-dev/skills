# Infrastructure Reference

CLI commands for workspace management, environments, database operations, daemon, secrets, configuration, and authentication. These are always available regardless of which extensions are enabled.

---

## Workspace

Workspaces are isolated data contexts. Each has one or more environments with its own DB connection and enabled extensions.

### Filesystem layout

```
~/.openclam/
├── openclam.json                   global config (active workspace/user, serve settings)
├── credentials.enc                 encrypted credential store (AES-256-GCM)
├── vault-key                       master key (mode 0600) or OPENCLAM_VAULT_KEY env
├── daemon.sock                     daemon control plane (Unix socket)
├── daemon.pid                      daemon PID file
├── models/                         local AI models (node-llama-cpp downloads)
└── workspaces/<slug>/
    ├── manifest.json               environments, extensions, connections, AI config
    └── envs/<env>/
        ├── data.db                 SQLite database (when dialect=sqlite, mode=local)
        ├── secrets.enc             runtime secret store
        └── buckets/<bucket-id>/    local file storage (when storage=file://)
```

### Setup

Before creating workspaces, initialize OpenClam:

```
clam setup                                 # First-time interactive setup (creates config + first workspace)
```

`clam setup` creates `~/.openclam/openclam.json` and walks through first workspace creation. If already set up, it shows current state.

### CLI Commands

```
clam --json workspace new --input-json '{"name":"...","slug":"...","dialect":"sqlite","extensions":["contacts","deals"],"userName":"...","userIdentifier":"...","userType":"human"}'
clam --json workspace list                 # List all workspaces (paginated)
clam --json workspace list --input-json '{"cursor":"0","limit":20}'
clam --json workspace use --input-json '{"slug":"<slug>"}'   # Switch active workspace
clam --json workspace current              # Show active workspace
clam --json workspace rename --input-json '{"old":"<old-slug>","new":"<new-slug>"}'
clam --json workspace delete <slug>        # Delete workspace (requires OTP confirmation)
clam --json workspace fork --input-json '{"source":"<src>","target":"<dst>"}'  # Fork with optional data
clam workspace extensions                  # Interactive: manage extensions for active environment
clam --json workspace extensions --input-json '{"enable":["tasks"],"disable":["deals"],"drop_disabled_tables":false}'
```

---

## Environment

Environments live within a workspace. Each has its own database, connection, and set of enabled extensions.

### CLI Commands

```
clam --json env list                       # List environments in active workspace (paginated)
clam --json env list --input-json '{"cursor":"0","limit":20}'
clam env add                               # Interactive: add new environment
clam --json env use --input-json '{"slug":"<slug>"}'   # Switch active environment
clam --json env info <slug>                # Show environment details
clam --json env delete <slug>              # Delete environment (requires OTP, cannot delete last)
clam --json env clone --input-json '{"source":"<src>","target":"<dst>"}'  # Clone env (optionally with data)
clam --json env diff --input-json '{"env1":"<a>","env2":"<b>"}'  # Compare two environments
```

---

## Database

### CLI Commands

```
clam --json db status                      # Show database status
clam --json db migrate                     # Run pending migrations for all enabled extensions
clam --json db verify                      # Verify schema and integrity
clam --json db repair                      # Attempt to fix missing columns
clam --json db backup                      # Create a database backup
```

### Supported Databases

| Dialect | Mode | Provider | Description |
|---------|------|----------|-------------|
| SQLite | local | — | File-based via LibSQL (`~/.openclam/workspaces/<slug>/envs/<env>/data.db`) |
| SQLite | remote | Turso | Hosted SQLite edge database |
| Postgres | local | — | Local PostgreSQL instance |
| Postgres | remote | Neon | Serverless Postgres |

---

## Status

```
clam --json status                         # Workspace, environment, connection, user, extensions, AI, storage
clam --json status --records               # Same, plus record counts per table
```

---

## Authentication (login/logout/passwd)

Two user types: **human** and **agent**. Both get API keys as the universal internal credential. Humans additionally have passwords for interactive login.

### CLI Commands

```
clam login                                 # Interactive picker — shows only human users, prompts for password
clam login <identifier>                    # Direct login — prompts for password (if set)
clam login <identifier> --code <code>      # Activate account with setup code — prompts to set password
clam --json logout                         # Clear active user session
clam --json whoami                         # Show current user details with ID from database
clam passwd                                # Set or change login password (human users only)
```

### Login rules

- **Agents CANNOT use `clam login`** — blocked if `OPENCLAM_AGENT` is set or user type is agent.
- **Login picker only shows human users** (agents are filtered out).
- **Humans with a password set** must enter their password on login.
- **Setup code flow**: when a human user is created, admin gets a 6-digit setup code (15 min expiry). The human activates via `clam login <identifier> --code <code>`, which prompts them to set a password.

### Auth resolution (in priority order)

1. `--token <api-key>` flag — explicit, per-command
2. `OPENCLAM_TOKEN` env var — persistent for a session/process
3. `OPENCLAM_AGENT` env var — agent credential lookup from encrypted vault
4. Active user session — human login session (via `clam login`)
5. Fail — no credential found

### Security

- OS keychain preferred for credential storage (with encrypted vault file fallback)
- Sessions are long-lived, revocable server-side
- All auth events logged to `log` table with `source='auth'`
- Agents cannot impersonate humans (login blocked, password required for humans)

---

## Daemon

The daemon is a long-running process that provides an HTTP API, cron scheduler, file watcher, and AI provider manager.

### CLI Commands

```
clam daemon start                          # Start in background
clam daemon run                            # Start in foreground (logs to terminal)
clam daemon stop                           # Graceful shutdown
clam --json daemon status                  # Show running state + enabled workspaces
clam daemon enable <workspace>             # Enable workspace in the daemon
clam daemon disable <workspace>            # Disable workspace
```

### What the daemon provides

- **HTTP API** on a configurable port (default 3400)
- **Control plane** on a Unix socket (`~/.openclam/daemon.sock`) for CLI ↔ daemon communication
- **Cron scheduler** that ticks registered jobs per environment
- **File watcher** that monitors storage bucket directories
- **AI provider manager** with reference counting and idle disposal
- **Workspace pool** — multiple workspaces, each with multiple environments, each with its own database connection, Hono app, cron jobs, and optional AI provider

### HTTP API Authentication

```
Authorization: Bearer <api-key>
```

When running through the daemon, requests include `X-Clam-Workspace` and `X-Clam-Env` headers to target a specific workspace and environment.

---

## Secrets (Vault)

Encrypted secret store for connection credentials and API keys. Secrets are resolved in two steps: environment variable (checked first), then encrypted vault fallback.

### CLI Commands

```
clam secrets init                          # Generate vault key + empty store
clam secrets set <name>                    # Prompt securely for value
clam secrets set <name> --value <value>    # Set directly
clam secrets get <name>                    # Decrypt and print
clam --json secrets list                   # List stored secret names (paginated)
clam --json secrets list --input-json '{"cursor":"0","limit":20}'
clam secrets delete <name>                 # Delete a secret
clam secrets reset                         # Destroy vault and re-initialize (requires OTP)
clam secrets exec -- <cmd>                 # Run command with secrets injected as env vars
```

### Usage in manifests

Reference secrets with the `$secret:` prefix in workspace manifests:

```json
{
  "environments": {
    "production": {
      "connection": {
        "dialect": "postgres",
        "provider": "neon",
        "mode": "remote",
        "url": "$secret:PROD_DB_URL"
      }
    }
  },
  "ai": {
    "embedding": {
      "provider": "openai",
      "api_key_env": "$secret:OPENAI_API_KEY"
    }
  }
}
```

Encryption: AES-256-GCM with per-value random salt and IV, scrypt key derivation. The vault key is stored at `~/.openclam/vault-key` (mode 0600) or provided via the `OPENCLAM_VAULT_KEY` environment variable.

---

## AI

Configure AI providers for embedding, completion, and reranker services. Config is stored in the workspace manifest (not the database). Environment-level config overrides workspace-level config per service.

### CLI Commands

```
# Interactive (humans)
clam config ai                                    # Full wizard — all three services
clam config ai embedding                          # Wizard for just embedding
clam config ai completion                         # Wizard for just completion
clam config ai reranker                           # Wizard for just reranker

# Non-interactive (agents — use --json + --input-json)
clam --json config ai set embedding --input-json '{"provider":"openai","model":"text-embedding-3-small","api_key_env":"OPENAI_API_KEY"}'
clam --json config ai set completion --input-json '{"provider":"anthropic","model":"claude-sonnet-4-5-20250514","api_key_env":"ANTHROPIC_API_KEY"}'
clam --json config ai set reranker --input-json '{"provider":"openai","model":"gpt-4.1-nano","api_key_env":"OPENAI_API_KEY"}'

# Non-interactive bulk (set multiple services at once)
clam --json config ai set --input-json '{"embedding":{"provider":"openai","model":"text-embedding-3-small","api_key_env":"OPENAI_API_KEY"},"completion":{"provider":"anthropic","model":"claude-sonnet-4-5-20250514","api_key_env":"ANTHROPIC_API_KEY"}}'

# Show config
clam --json config ai get                         # Show resolved config (env overrides workspace) for all services
clam --json config ai get embedding               # Show just embedding config
clam --json config ai get --input-json '{"scope":"workspace"}'   # Show workspace-level only
clam --json config ai get --input-json '{"scope":"env"}'         # Show env-level only

# Reset config
clam --json config ai reset                       # Clear all AI config
clam --json config ai reset embedding             # Clear just embedding
clam --json config ai reset --input-json '{"scope":"env"}'       # Clear env-level AI config
```

### Providers

| Provider | Embedding default | Completion default | Needs API key | Needs base_url |
|----------|-------------------|-------------------|---------------|----------------|
| openai | text-embedding-3-small | gpt-4.1-nano | Yes (OPENAI_API_KEY) | No |
| anthropic | _(no embedding)_ | claude-sonnet-4-5-20250514 | Yes (ANTHROPIC_API_KEY) | No |
| ollama | nomic-embed-text | llama3.2 | No | Optional (default: http://localhost:11434) |
| openrouter | openai/text-embedding-3-small | openai/gpt-4.1-nano | Yes (OPENROUTER_API_KEY) | No |
| local | embeddinggemma-300m | _(no completion)_ | No | No |

### Scope

AI config can be set at two levels:
- **Workspace** (default) — applies to all environments
- **Environment** — overrides workspace config for that specific environment

Use `--scope env` to target the active environment. When reading config, the resolved view merges both levels (env wins).

---

## Config

### CLI Commands

```
clam --json config get --global            # Get global CLI configuration
clam --json config set --global <key> <value>  # Set global config value
clam --json config reset --global          # Reset global configuration to defaults
clam --json config get                     # Get all workspace config (defaults + overrides)
clam --json config get <key>               # Get workspace config value
clam --json config set <key> <value>       # Set workspace config value
clam --json config reset <key>             # Reset workspace config key to default
clam --json config keys                    # List all valid config keys with descriptions
```

---

## Reset

```
clam reset                                 # Delete ALL workspaces, users, and configuration
```

Requires typing a random 6-digit OTP to confirm. This is irreversible.

---

## Container

Manage Docker/Podman containers for isolated workspace deployment.

```
clam container start                       # Build and start container (default port 3401)
clam container start --port 3500           # Start on custom port
clam container stop                        # Stop the container
clam --json container status               # Show container status
clam container logs                        # Follow container logs
clam container logs --tail 100             # Last 100 lines + follow
```

After starting, create a workspace in container mode:
```
clam workspace new                         # Select "Container" deployment mode when prompted
```

---

## Token

Manage long-lived JWT tokens for external integrations (e.g. MCP service).

```
clam --json token mint                     # Issue a JWT for the active workspace/environment
clam --json token mint --ttl 30d           # Custom lifetime (s, m, h, d)
clam --json token mint --workspace <slug> --env <env>  # Target specific workspace/environment
```

Requires a running daemon. Exchanges your API key for a signed JWT.

---

## Docs

```
clam docs                                  # Open https://openclam.dev in browser
```

---

## Global Flags

| Flag | Description |
|------|-------------|
| `--json` | Produce compact JSON output (no pretty printing, no colors). Required for agents. |
| `--markdown` | Output as markdown tables (lists) or key-value blocks (single objects). Pairs well with `--fields`. Lower token count than JSON. |
| `--token <api-key>` | API key for authentication. Alternative to `OPENCLAM_TOKEN` env var |
| `--env <env>` | Override the active environment for this command |
| `--input-json '<json>'` | Command-level option on many subcommands (not a root global flag). Pass structured input as a single JSON object. |
