---
name: setup-ai-providers
description: >
  Configure AI providers for embeddings, completions, and reranking. Covers
  OpenAI, Anthropic, Ollama, OpenRouter, and on-device local models, the
  default models, and the non-interactive `clam ai set` form for agents.
metadata:
  author: openclam
  version: 0.1.0
---

# AI providers

OpenClam uses AI for three distinct services: **embedding** (semantic search,
hybrid search, decision traces), **completion** (the `clam ask` command,
workflow actions, agent helpers), and **reranker** (optional — improves
search relevance when enabled). Each is configured independently and stored
in the workspace manifest under `ai.<service>`. Config is workspace-level;
different workspaces can use different providers.

## Interactive

```bash
clam ai                 # Configure all three services
clam ai embedding       # Just embedding
clam ai completion      # Just completion
clam ai reranker        # Just reranker
```

The wizard walks you through provider selection, model selection (from a
catalog per provider — Ollama queries your local install live), and the
API key environment variable name. It then verifies: downloads local
models, pings Ollama, or warns if the API-key env var is unset.

## Non-interactive (agents)

Set one service:

```bash
clam --json ai set embedding --input-json '{
  "provider": "openai",
  "model": "text-embedding-3-small",
  "api_key_env": "OPENAI_API_KEY"
}'

clam --json ai set completion --input-json '{
  "provider": "anthropic",
  "model": "claude-sonnet-4-5-20250514",
  "api_key_env": "ANTHROPIC_API_KEY"
}'
```

Set multiple at once (omit the service arg):

```bash
clam --json ai set --input-json '{
  "embedding": {"provider": "openai", "model": "text-embedding-3-small", "api_key_env": "OPENAI_API_KEY"},
  "completion": {"provider": "anthropic", "model": "claude-sonnet-4-5-20250514", "api_key_env": "ANTHROPIC_API_KEY"},
  "reranker":   {"provider": "openai", "model": "gpt-4.1-nano", "api_key_env": "OPENAI_API_KEY"}
}'
```

Inspect and clear:

```bash
clam --json ai get
clam --json ai get completion
clam --json ai reset embedding
clam --json ai reset                # clear everything
```

## Provider matrix

| Provider | Embedding default | Completion default | API key env | Base URL |
|---|---|---|---|---|
| `openai` | `text-embedding-3-small` | `gpt-4.1-nano` | `OPENAI_API_KEY` | – |
| `anthropic` | _(no embedding)_ | `claude-sonnet-4-5-20250514` | `ANTHROPIC_API_KEY` | – |
| `ollama` | `nomic-embed-text` | `llama3.2` | – | default `http://localhost:11434`, override with `OLLAMA_HOST` |
| `openrouter` | `openai/text-embedding-3-small` | `openai/gpt-4.1-nano` | `OPENROUTER_API_KEY` | – |
| `local` | `embeddinggemma-300m` | _(no completion)_ | – | – (llama.cpp on-device) |

## Picking a provider

- **OpenAI** — best default for most workspaces. Small embedding model is
  cheap, nano completion is fast. Needs `OPENAI_API_KEY`.
- **Anthropic** — completion only; pair with OpenAI (or Ollama) for
  embedding.
- **Ollama** — fully local. The wizard pings your Ollama instance and
  lists installed models, filtering by kind (embedding families like
  `bert`, `nomic-bert`, `xlm-roberta` for the embedding service, all
  others for completion). Pull models ahead of time with
  `ollama pull <model>`.
- **OpenRouter** — one API key routes to many upstream models. Good for
  comparing models.
- **Local** — embedding only, via `node-llama-cpp`. Downloads and caches
  models under `~/.openclam/models/`. Use this when you want embeddings
  without any network dependency.

## Env var convention

`api_key_env` stores **the name** of the env var — not the key itself.
The daemon and CLI read `process.env[api_key_env]` at request time. That
way keys never land in the manifest file, and secret rotation is
`export OPENAI_API_KEY=new-key && clam daemon restart`.

If you'd rather store the secret in the encrypted vault than a shell
env, reference it with `$secret:` in the manifest (see
[secrets.md](./secrets.md)).

## Per-environment overrides

Currently AI config is workspace-level only. (Earlier docs mentioned
environment-level overrides — that functionality is not in the current
`clam ai` command.) If your use case needs per-environment models, file
an issue with the details.

## Verifying

- For OpenAI/Anthropic/OpenRouter: `clam ask "health check"` will 401 if
  the key env var isn't set.
- For Ollama: the wizard confirms connectivity live. After config, a
  spinner shows "Ollama connected (<model>)".
- For local: the wizard downloads the model with a progress bar. Once
  done, `clam embed` jobs will run fully offline.

## Notes

- Changing an AI provider doesn't re-embed existing rows — the embedding
  queue marks them stale and re-embeds asynchronously. Expect a catch-up
  period proportional to row count.
- Anthropic + embedding is not supported; the wizard warns and refuses.
- The `local` provider does not offer completion; pair it with OpenAI or
  Anthropic.

See `../surfaces.md` to translate these operations to MCP, HTTP, or the
typed client.
