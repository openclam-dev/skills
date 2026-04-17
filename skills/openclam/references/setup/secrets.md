---
name: setup-secrets
description: >
  Initialize the encrypted secret vault. Covers AES-256-GCM file encryption,
  the master key in `~/.openclam/vault-key`, and the `clam secrets` command
  surface for writing, reading, listing, and injecting secrets.
metadata:
  author: openclam
  version: 0.1.0
---

# Secrets and the vault

The vault is a per-workspace encrypted key-value store that holds things
you'd otherwise paste into env vars or commit by accident: API keys,
database passwords, webhook signing secrets, OAuth tokens. Encryption is
AES-256-GCM per value, with a per-workspace derived key. The root key
lives in `~/.openclam/vault-key` (mode 0600) or in the `OPENCLAM_VAULT_KEY`
env var.

## Initialize

Per-workspace, one-time. Generates the root key if missing, then creates
the workspace's secrets store file.

```bash
clam secrets init
```

The root key is reused across workspaces; a fresh `init` on another
workspace uses the same root key and derives a workspace-specific
encryption key from it.

## Write and read

```bash
clam secrets set OPENAI_API_KEY              # prompts securely
clam secrets set OPENAI_API_KEY --value sk-...   # direct (beware shell history)
clam --json secrets set OPENAI_API_KEY --input-json '{"value":"sk-..."}'

clam --json secrets get OPENAI_API_KEY
clam --json secrets list
clam --json secrets list --input-json '{"cursor":"0","limit":50}'
clam --json secrets remove OPENAI_API_KEY
clam --json secrets remove OPENAI_API_KEY --force   # skip confirmation
```

Reset a workspace's vault (destructive — drops all secrets for that
workspace, keeps the root key):

```bash
clam secrets reset --force
```

## Inject secrets into a subprocess

Run a command with every secret in the active workspace exported as an
env var. Useful for local scripts and for calling tools that read
`OPENAI_API_KEY` directly from the environment.

```bash
clam secrets exec -- node scripts/seed.mjs
clam secrets exec -- python run.py
```

## What lives here vs. elsewhere

The vault has two kinds of data:

1. **Secrets store** — what `clam secrets set` writes. Scoped per
   workspace.
2. **Credentials store** — user API keys and sessions that OpenClam
   writes automatically on `workspace new`, `login`, and token mint. You
   normally don't touch these directly; the CLI manages them.

Both live under `~/.openclam/`:

```
~/.openclam/
├── vault-key                 # root key, mode 0600 (or via OPENCLAM_VAULT_KEY)
├── credentials.enc           # user/agent API keys and session tokens
└── workspaces/<slug>/
    └── secrets.enc           # this workspace's secrets store
```

## Reference secrets from manifests and AI config

Use the `$secret:<NAME>` prefix to pull a value from the vault at runtime
instead of from an env var. Example in a workspace manifest:

```json
{
  "ai": {
    "embedding": {
      "provider": "openai",
      "model": "text-embedding-3-small",
      "api_key_env": "$secret:OPENAI_API_KEY"
    }
  },
  "environments": {
    "production": {
      "connection": {
        "dialect": "postgres",
        "url": "$secret:PROD_DB_URL"
      }
    }
  }
}
```

At resolution time, OpenClam sees the `$secret:` prefix and fetches the
current value from the vault — no env export needed.

## Encryption details

- **Algorithm:** AES-256-GCM with a 12-byte random IV per record and the
  workspace ID as AAD.
- **Root key:** 256-bit, stored as 64 hex chars in `~/.openclam/vault-key`
  (mode 0600) or the `OPENCLAM_VAULT_KEY` env var. Env takes precedence.
- **Per-workspace key:** derived from the root key + workspace ID, so
  workspaces can't read each other's secrets even with the same root.
- **`vaultKeyCmd`** — optional; set `vaultKeyCmd` in `~/.openclam/openclam.json`
  to a shell command that prints the key. OpenClam runs it per process
  and never stores the output. Use this if you want the key loaded from
  a password manager CLI at runtime.
- **OS keychain integration** — not a current direct storage target; the
  OS keychain can be used via `vaultKeyCmd` (e.g. `security find-generic-password ...`
  on macOS, `secret-tool lookup ...` on Linux).

## Backup and rotation

- `~/.openclam/vault-key` is the single point of recovery. Back it up
  somewhere safe (1Password item, offline device). Losing it means every
  `.enc` file is unrecoverable.
- **Never commit** `vault-key`, `credentials.enc`, or `secrets.enc` to
  source control.
- `clam backup add --scope full --include-vault-key` bundles the key
  into a backup archive; `--no-include-vault-key` omits it (the resulting
  archive can't be restored on another machine).
- Rotating the root key is not a built-in flow yet — today it means
  `clam secrets reset`, capturing the values first, then `set` them back
  with a new key in place.

## Notes and gotchas

- **Setting a value with `--value` is visible in shell history.** Prefer
  the prompt form or `--input-json` fed from a file.
- **`clam secrets get <name>` prints plaintext** — fine for scripts, dangerous
  in shared terminals.
- **Multiple workspaces share one vault key** — switching workspaces
  changes which `.enc` file is read; the root key is constant.
- **Daemon inherits env at install time.** If you set `OPENCLAM_VAULT_KEY`
  after `clam daemon start`, the running daemon won't see it until
  `clam daemon restart`.

See `../surfaces.md` to translate these operations to MCP, HTTP, or the
typed client.
