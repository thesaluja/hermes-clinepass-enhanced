# hermes-clinepass-enhanced

ClinePass model-provider plugin for [Hermes Agent](https://github.com/NousResearch/hermes-agent) - enhanced fork of [Adolanium/hermes-clinepass](https://github.com/Adolanium/hermes-clinepass) with a fix for non-streaming responses.

Use [ClinePass](https://cline.bot/cline-pass) models (`cline-pass/*`) from Hermes with a normal API key. Includes Kimi K3, GLM, DeepSeek, MiniMax, MiMo, Qwen, and Muse Spark - with **live plan-catalog discovery** so new models appear without a plugin update.

## Why this fork

Upstream v1.2.x removed the response-envelope unwrap, but the Cline gateway **still double-wraps non-streaming Chat Completions responses** (verified against the live API 2026-10-02):

```json
{"success": true, "data": {"choices": [...], "usage": {...}}}
```

With `streaming: false` in Hermes (`hermes config set streaming.enabled false`, or any code path that makes a non-streaming call - cron fallbacks, auxiliary clients), the OpenAI SDK reports `response.choices is None` and the agent fails with **"response has no 'choices' attribute"** after 3 retries.

This fork restores a **choices-aware unwrap**: the envelope is stripped only when the top level has no `choices`, so both wrapped and unwrapped gateway behaviors work. It also patches at the **class level** (`Completions.create` / `AsyncCompletions.create`) so clients cached before the plugin loads - cron scheduler, auxiliary-client cache - are covered too, and evicts stale cached clients at registration.

Everything else follows upstream v1.2.1: live catalog from `GET /api/v1/ai/cline/recommended-models` (public, no auth; keys `clinePass` + `free`), curated fallback verified live 2026-10-02, dead slugs removed (`glm-5.2`, `kimi-k2.7-code`, `kimi-k2.6`, `deepseek-v4-flash`).

## Install

**Linux / macOS**

```bash
curl -fsSL https://raw.githubusercontent.com/thesaluja/hermes-clinepass-enhanced/main/scripts/install.sh | bash
```

**Windows (PowerShell)**

```powershell
irm https://raw.githubusercontent.com/thesaluja/hermes-clinepass-enhanced/main/scripts/install.ps1 | iex
```

**Or with Hermes directly**

```bash
hermes plugins install thesaluja/hermes-clinepass-enhanced
```

**Manual**: copy `plugin/clinepass/` into `$HERMES_HOME/plugins/model-providers/` (Windows: `%LOCALAPPDATA%\hermes\plugins\model-providers\`), then `hermes plugins enable model-providers/clinepass`.

## API key

Create a key at [app.cline.bot](https://app.cline.bot) (Settings → API Keys). You need an active [ClinePass](https://cline.bot/cline-pass) subscription for `cline-pass/*` models.

```bash
# $HERMES_HOME/.env  (Windows: %LOCALAPPDATA%\hermes\.env)
CLINEPASS_API_KEY=your_key_here     # or CLINE_API_KEY
```

## Chat

```bash
hermes chat -q "Reply with exactly: HELLO" --provider clinepass -m cline-pass/kimi-k3
```

Or in `config.yaml`:

```yaml
model:
  provider: clinepass
  default: cline-pass/kimi-k3
```

Provider aliases: `clinepass`, `cline-pass`, `cline`.

## Models

The picker lists the live plan catalog (14 pass + 4 free models as of 2026-10-02, self-updating). Fallback if the fetch fails:

| Model | ID |
| --- | --- |
| GLM-5.3 Flash | `cline-pass/glm-5.3-flash` |
| GLM-5.3 | `cline-pass/glm-5.3` |
| DeepSeek V4 Pro | `cline-pass/deepseek-v4-pro` |
| DeepSeek V4.1 Flash | `cline-pass/deepseek-v4.1-flash` |
| Qwen3.8 Max | `cline-pass/qwen3.8-max` |
| Kimi K3 | `cline-pass/kimi-k3` |
| MiMo-V2.5-Pro | `cline-pass/mimo-v2.5-pro` |
| MiMo-V2.5 | `cline-pass/mimo-v2.5` |
| MiniMax M3 | `cline-pass/minimax-m3` |
| Muse Spark 1.3 (contributor) | `cline-pass/muse-spark-1.3-contributor` |
| Qwen3.7 Max | `cline-pass/qwen3.7-max` |
| Qwen3.7 Plus | `cline-pass/qwen3.7-plus` |

Default auxiliary model: `cline-pass/deepseek-v4-pro`.

Free models (`cline-free/*`, `stealth/*`) keep their gateway IDs and cost nothing - useful when the account is out of credits.

`muse-spark-1.3-contributor` requires interactive consent for data-training tiers on first use (Hermes-side gate; set `security.allow_data_training_tiers_noninteractive: true` for unattended runs).

## Verify

```bash
hermes chat -q "Reply with exactly: OK" --provider clinepass -m cline-pass/kimi-k3
# → OK
```

## Differences from upstream

| | upstream v1.2.1 | this fork |
| --- | --- | --- |
| Non-streaming responses | broken (choices = None) | ✅ choices-aware unwrap |
| Cached clients (cron/aux) | uncovered | ✅ class-level patch + eviction |
| Live plan catalog | ✅ recommended-models | ✅ same |
| Env vars | `CLINE_API_KEY` | ✅ + `CLINEPASS_API_KEY` |

## Credits

- [Adolanium](https://github.com/Adolanium/hermes-clinepass) - original plugin, live-catalog discovery, tests, and install scripts (MIT)
- Fixes and hardening: Gaurav Saluja ([@thesaluja](https://github.com/thesaluja))

## License

MIT - see [LICENSE](LICENSE).
