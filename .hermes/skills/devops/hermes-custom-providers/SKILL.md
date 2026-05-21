---
name: hermes-custom-providers
description: "Configure Hermes to use custom OpenAI-compatible LLM endpoints (e.g., DIT.AI, self-hosted vLLM, Ollama, local proxies)."
version: 1.0.0
author: Hermes Agent
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [hermes, configuration, llm, provider, openai-compatible, custom-endpoint]
    related_skills: [hermes-agent]
---

# Hermes Custom Providers

How to wire Hermes to arbitrary OpenAI-compatible endpoints using the `custom_providers:` list in `config.yaml`.

## Trigger

Use this skill when the user wants to:
- Point Hermes at a third-party OpenAI-compatible API (e.g. DIT.AI, Novita, Together, a self-hosted vLLM/Ollama instance, or an internal proxy).
- Switch from a built-in provider to a custom endpoint.
- Troubleshoot why Hermes is still hitting the old provider after adding a `custom_providers` entry.

## Steps

1. **Define the custom provider entry** in `~/.hermes/config.yaml` under the `custom_providers:` list (YAML array — each entry starts with `-`):

```yaml
custom_providers:
  - name: "Api.dit.ai"
    base_url: "https://api.dit.ai"
    api_key: "sk-..."
    model: "claude-opus-4.6"
    api_mode: "chat_completions"
```

| Field | Required | Notes |
|-------|----------|-------|
| `name` | Yes | Display name. Used to build the slug `custom:<normalized-name>`. |
| `base_url` | Yes | Endpoint URL, no trailing `/v1/chat/completions` — just the base. |
| `api_key` | Yes* | Inline key. Prefer `key_env` (see Security below). |
| `model` | No | Default model string when this provider is selected. |
| `api_mode` | No | `"chat_completions"` (default) or `"codex_responses"`. |
| `key_env` | No | Env-var name to read the key from instead of inline `api_key`. |

2. **Point `model` section at the custom provider**:

```yaml
model:
  default: claude-opus-4.6
  provider: custom:api.dit.ai
  api_mode: chat_completions
```

- `provider` must be `custom:<normalized-name>`. Normalization: lowercase, spaces → hyphens.
  - Example: name `"Api.dit.ai"` → slug `custom:api.dit.ai`.
- `default` should match the `model` field from the custom provider entry so `hermes model` and quick commands resolve correctly.

3. **Remove legacy fields from `model:`** — this is the most common pitfall.

   Hermes resolves credentials in a priority order. If the `model:` section still carries a legacy `base_url` or `api_key` from a previous provider, those values **override** the `custom_providers` entry and Hermes keeps talking to the old endpoint.

   **Before:**
   ```yaml
   model:
     default: claude-opus-4.6
     provider: custom:api.dit.ai
     base_url: https://opencode.ai/zen/go/v1   # ← stale — will be used anyway
     api_key: sk-...old                        # ← stale — will be used anyway
   ```

   **After:**
   ```yaml
   model:
     default: claude-opus-4.6
     provider: custom:api.dit.ai
     api_mode: chat_completions
   ```

4. **Verify with doctor**:

```bash
hermes doctor
```

Look under "API Connectivity" for the custom provider name. If it still shows the old provider, the legacy fields were not cleaned.

5. **Test a single-turn query**:

```bash
hermes chat -q "Hello — confirm which provider you are using."
```

## Security: prefer `key_env` over inline `api_key`

Instead of storing the key in `config.yaml`, keep it in `~/.hermes/.env`:

```bash
# ~/.hermes/.env
DIT_AI_API_KEY=sk-...
```

Then reference it in the custom provider:

```yaml
custom_providers:
  - name: "Api.dit.ai"
    base_url: "https://api.dit.ai"
    key_env: "DIT_AI_API_KEY"
    model: "claude-opus-4.6"
```

This keeps secrets out of the YAML file and respects the user's existing `.env` workflow.

## Discovering and declaring available models

By default Hermes tries to fetch the live model list from the endpoint's `/v1/models` route when credentials are available (`discover_models: true`). You can also declare models explicitly in the provider entry so they always appear in the picker even if live discovery is slow or the provider exposes an aggregator catalog.

### Querying a provider's model catalog

For OpenAI-compatible providers, list available models with a direct API call:

```bash
curl -s "https://api.example.com/v1/models" \
  -H "Authorization: Bearer $API_KEY" | jq -r '.data[].id'
```

### Explicit `models:` block format

Add a `models:` key to the provider entry. Hermes expects a **dict keyed by model ID** (values are empty dicts or per-model overrides), not a YAML list:

```yaml
custom_providers:
  - name: "ditai"
    base_url: "https://api.dit.ai/v1/"
    key_env: "DIT_AI_API_KEY"
    model: "gpt-5.5"
    models:
      gpt-5.5: {}
      claude-opus-4-7: {}
      claude-sonnet-4-6: {}
      gemini-3-pro-preview: {}
      kimi-k2.5: {}
      minimax-m2.7: {}
```

- The `models:` dict is merged with any live-discovered models; duplicates are deduplicated.
- Use explicit declaration when you want a curated subset (e.g. a reseller that exposes 100+ models but you only want the cheapest or most capable ones).
- If the provider's `/v1/models` returns an unwanted aggregator catalog, disable live discovery:
  ```yaml
  discover_models: false
  ```

## Pitfalls

- **Dict vs list**: `custom_providers:` must be a YAML list (`- ` prefix per entry). If written as a dict, Hermes logs a warning and ignores it. Run `hermes doctor` to catch this.
- **Bare `custom` provider string**: If `model.provider` is literally the string `"custom"` (corrupt state from an old model-switch bug), Hermes falls back to the first valid `custom_providers` entry. Fix it by setting the explicit slug.
- **Alias shadowing**: If the custom provider `name` happens to match a built-in provider alias (e.g. `"kimi"`), Hermes may route to the built-in instead. Use a distinctive display name (e.g. `"My Kimi Proxy"`).
- **Credential pool exhaustion**: If multiple API keys are pooled for the same base URL, Hermes rotates through them. For single-key custom endpoints this is harmless; for multi-key setups it is a feature.
- **`models:` as a list**: Older hand-edited configs sometimes write `models: [gpt-4, claude-opus-4]` as a YAML list. Hermes accepts this for backward compatibility but normalises it internally to a dict. Prefer the dict form for new configs.

## References

- Hermes provider docs: https://hermes-agent.nousresearch.com/docs/integrations/providers
- Runtime resolution logic: `hermes_cli/runtime_provider.py` (`_get_named_custom_provider`, `_resolve_named_custom_runtime`)
