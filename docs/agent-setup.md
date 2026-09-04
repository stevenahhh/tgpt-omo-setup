# TimelyGPT (SCNU) -> OMO Setup Manual (for coding agents)

This document is the procedure a coding agent must follow to connect the TimelyGPT SCNU workspace as a provider in the user's OMO (`omo` CLI) installation. It is written for agents to read and execute, not for humans.

---

## GATE 0: Obtain the API key (mandatory, blocking)

**Do not modify any file before this gate is passed.**

1. Ask the user explicitly:
   > "Please provide your TimelyGPT integration key (starts with `tgpt_sk_`). You can issue one at https://timelygpt.co.kr/scnu/timely-agent under Profile -> Settings -> Integration Key Management."
2. **Wait** until the user supplies the key. Never guess a key, never proceed with a placeholder, and never reuse a key found in earlier sessions or logs.
3. When the key is received, verify the `tgpt_sk_` prefix. If it does not match, ask the user to confirm.
4. Do not echo the key into output or logs. Write it only into the config file below. Never re-print the full key after setup completes.

---

## Step 1: Pre-flight checks

- `omo --version` must run successfully. OMO is distributed as an npm/bun global package (`omo-ai`) and runs on Node or Bun, so any OS with either runtime works.

### Config paths per OS (verified against `omo-ai/bin/lib/agent-dir.js`)

OMO resolves its state directory through one module for all entry points. The resolution order is identical on every OS:

1. If one of the env vars `OMO_CODING_AGENT_DIR`, `SENPI_CODING_AGENT_DIR`, `PI_CODING_AGENT_DIR` is set, that directory wins.
2. Otherwise the default is `<home>/.omo/agent`, where `<home>` is `$HOME` or `%USERPROFILE%` (whichever is set; on Windows `%USERPROFILE%` is the fallback).

There is NO `%APPDATA%` or `~/.config` special-casing. Concretely:

| OS | Agent dir | models.json |
|---|---|---|
| macOS / Linux | `~/.omo/agent` | `~/.omo/agent/models.json` |
| Windows (PowerShell/CMD) | `%USERPROFILE%\.omo\agent` | `%USERPROFILE%\.omo\agent\models.json` |

The second config file lives one level above the agent dir:

| OS | omo.jsonc |
|---|---|
| macOS / Linux | `~/.omo/omo.jsonc` |
| Windows | `%USERPROFILE%\.omo\omo.jsonc` |

To resolve the paths programmatically instead of hardcoding, run:
```sh
# POSIX (macOS/Linux sh, bash, zsh; also Git Bash on Windows)
echo "$HOME/.omo/agent"
```
```powershell
# Windows PowerShell
echo "$env:USERPROFILE\.omo\agent"
```

- Back up both files before editing (pick the variant matching the current shell):
  ```sh
  # POSIX
  cp ~/.omo/agent/models.json /tmp/models.json.bak
  cp ~/.omo/omo.jsonc /tmp/omo.jsonc.bak
  ```
  ```powershell
  # Windows PowerShell
  Copy-Item $env:USERPROFILE\.omo\agent\models.json $env:TEMP\models.json.bak
  Copy-Item $env:USERPROFILE\.omo\omo.jsonc $env:TEMP\omo.jsonc.bak
  ```

## Step 2: Register the provider (`~/.omo/agent/models.json`)

Add `timely-scnu` under `providers`. Set `apiKey` to the key obtained in GATE 0.

```json
{
  "providers": {
    "timely-scnu": {
      "baseUrl": "https://hello.timelygpt.co.kr/api/v2/chat/bridge/openai",
      "api": "openai-completions",
      "apiKey": "tgpt_sk_THE_KEY_THE_USER_PROVIDED",
      "authHeader": true,
      "models": [ ...model list below... ]
    }
  }
}
```

Notes:
- Do NOT append `/v1` to the baseUrl; doing so causes 404.
- Do not touch other existing providers.
- If the user prefers not to store the key in the file, use `"apiKey": "$TIMELYGPT_API_KEY"` instead and instruct the user to set the env var per OS:
  - macOS/Linux: add `export TIMELYGPT_API_KEY=tgpt_sk_...` to `~/.zshrc` or `~/.bashrc`
  - Windows PowerShell (persistent): `[Environment]::SetEnvironmentVariable('TIMELYGPT_API_KEY','tgpt_sk_...','User')`, then restart the terminal
  - Windows CMD (persistent): `setx TIMELYGPT_API_KEY "tgpt_sk_..."`, then restart the terminal

  By default (this project owner's preference) write the key directly into the file instead of using env vars.

### Common model entry format

```json
{
  "id": "google/gemini-3.8-flash",
  "name": "google/gemini-3.8-flash via Timely",
  "contextWindow": 1048576,
  "reasoning": true,
  "input": ["text"],
  "thinkingLevelMap": { "off": null, "minimal": "low", "low": "low", "medium": "medium", "high": "high", "xhigh": "high", "max": "high" },
  "compat": { "supportsReasoningEffort": true }
}
```

**`contextWindow` correctness matters.** OMO uses this number for its model-switch guard and compaction timing, not just display. Set it too high and the agent believes it still has headroom long after the real upstream limit is gone, so every call starts failing with a 400 right when a long session needs the model most. Do not blanket-set 1M for every model — use the verified value below.

### Recommended model list with verified `contextWindow` (as of 2026-09-03)

Register exactly these 14 models — the latest per family, no `:batch` variants (batch requires the async Batch API and cannot be used by interactive agents). The `contextWindow` column is either an empirically confirmed real limit (found by sending an oversized prompt and reading the upstream error message) or, where marked unverified, the vendor's advertised figure copied as a placeholder:

| Family | Model id | `contextWindow` | Verified? |
|---|---|---|---|
| GPT | `openai/gpt-5.6-sol` | `1048576` | Yes — live probe at 900K passed, 1.1M failed with `"maximum context length is 1050000 tokens"`. `1048576 < 1050000`, safe margin. |
| GPT | `openai/gpt-5.6-terra` | `1048576` | Yes, same probe result as gpt-5.6-sol. |
| GPT | `openai/gpt-5.6-luna` | `1048576` | Yes, same probe result as gpt-5.6-sol. |
| Kimi | `kimi-k3` | `262144` | Yes — live probe at ~450K failed with `"maximum context length is 262144 tokens"` (DeepInfra backend). Do NOT set this to 1M; it is a real regression, not a placeholder. |
| Kimi | `moonshotai/kimi-k3` | `262144` | Same as `kimi-k3` (same backend). |
| Claude | `anthropic/claude-sonnet-4.6` | `1048576` | **No** — not empirically probed. Anthropic's own docs advertise 1M for this tier; treat as a placeholder until tested. |
| Claude | `anthropic/claude-opus-4.7` | `1048576` | No, same caveat. |
| Claude | `anthropic/claude-opus-4.8` | `1048576` | No, same caveat. |
| Claude | `anthropic/claude-sonnet-5` | `1048576` | No, same caveat. |
| Claude | `anthropic/claude-opus-5` | `1048576` | No, same caveat. |
| Claude | `anthropic/claude-fable-5` | `1048576` | No, same caveat. |
| Claude | `anthropic/claude-fable-5.1` | `1048576` | No, same caveat. |
| Gemini | `google/gemini-3.8-flash` | `1048576` | No, same caveat. |
| Grok | `x-ai/grok-4.6` | `256000` | No — not probed; set conservatively below the GPT/Claude tier since xAI's public figure for this class is smaller. |

Do NOT register older or mid-tier models (gpt-5.5 and below, claude-4.5 and below, gemini-3.7 and below, mistral, solar, llama, qwen) — they still answer on the bridge, but keeping the catalog to the frontier avoids silent quality regressions when an agent picks a model by name.

Note: the model list itself drifts. `curl -s https://hello.timelygpt.co.kr/api/v2/chat/bridge/info/models` shows the public list, but models missing from it can still work when called with authentication — a successful live call is the real criterion.

#### GATE 1 (optional, cost-blocking): empirically verifying an unverified `contextWindow`

Confirming a real context limit requires sending a prompt large enough to approach or exceed it, which consumes real workspace credit proportional to input tokens (a single ~900K-token probe is not cheap; a full sweep across the unverified Claude/Gemini/Grok rows in the table above costs noticeably more). **Do not run this kind of probe on your own initiative.** If the user asks you to verify one of the "No" rows above:

1. Tell the user up front, before sending anything, roughly how many large-prompt calls the verification needs (one binary-search style sweep per model is typically 2-4 calls).
2. Only proceed after the user confirms.
3. Method: send a prompt sized near the vendor's advertised limit; read the upstream error message on failure (it states the real limit directly, e.g. `"maximum context length is 262144 tokens"`), or confirm success under it. A needle-in-haystack prompt (place a unique fact in the middle of filler text, then ask for it back) additionally confirms the model can actually use the full window, not just accept the byte count.
4. Update the table and `contextWindow` value only for the models actually tested; do not extrapolate one family's result onto another.

## Step 3: Register recommendation chains (`~/.omo/omo.jsonc`)

Insert `timely-scnu/kimi-k3` into the `models` arrays under `categories` and `agents`. Array order is the priority order (1st choice -> fallback). This project owner's current preference:

- **1st choice**: `categories.writing`, `categories.visual-engineering`
- **2nd choice (fallback)**: everything else (`quick`, `unspecified-low`, `unspecified-high`, `architect`, `ultrabrain`, `deep`, `artistry`, `agents.explore`, `agents.librarian`, `agents.metis`, `agents.momus`)

Entry format:
```json
{ "model": "timely-scnu/kimi-k3", "reasoning": "low" }
```

If the user requests a different priority, follow their instruction. Do not delete existing entries.

## Step 4: Verification (mandatory)

1. Catalog recognition:
   ```sh
   omo --list-models timely-scnu
   ```
   The model list must be printed. These commands are identical on macOS, Linux, and Windows because `omo` itself is a cross-platform Node/Bun binary; only the config file paths differ (see Step 1).
2. Live call (use the offline flag to avoid a startup stall):
   ```sh
   omo --offline --provider timely-scnu --model kimi-k3 \
     --no-model-fallback --no-recommended-models --no-session --no-tools \
     -p 'Reply with exactly SETUP_OK'
   ```
   On Windows PowerShell, replace the trailing backslash line-continuations with backticks or write the command on one line. Success = output contains `SETUP_OK` and exit code 0.
3. Per-model reasoning verification (mandatory after the basic live call passes):

   The Timely bridge accepts the OpenAI-style `reasoning_effort` parameter (`low`/`medium`/`high`) and returns the reasoning trace in `message.reasoning` plus `usage.completion_tokens_details.reasoning_tokens`. Verify this per model family — do not assume all registered models support it.

   For each model you registered, run this test (POSIX shell; on Windows use Python or `curl.exe` with double quotes):
   ```sh
   curl -s -X POST https://hello.timelygpt.co.kr/api/v2/chat/bridge/openai/chat/completions \
     -H "Authorization: Bearer $TIMELYGPT_API_KEY" \
     -H 'Content-Type: application/json' \
     -d '{"model":"MODEL_ID_HERE","messages":[{"role":"user","content":"What is 17*19? Think step by step."}],"max_tokens":200,"reasoning_effort":"high"}'
   ```
   Pass criteria per model:
   - HTTP 200 (a 400 `not a valid model ID` means the id is wrong or the model was delisted — remove it from models.json)
   - `usage.completion_tokens_details.reasoning_tokens` > 0
   - `choices[0].message.reasoning` present and non-empty (some backends omit the field while still spending reasoning tokens; a token count > 0 alone is still acceptable support)

   Minimum test matrix (one per family is enough if time-constrained, full matrix preferred):
   | Family | Test model |
   |---|---|
   | GPT | `openai/gpt-5.6-sol` |
   | Claude | `anthropic/claude-sonnet-5` |
   | Gemini | `google/gemini-3.8-flash` |
   | Kimi | `kimi-k3` |

   If a model fails the reasoning check, set `"reasoning": false` and remove its `thinkingLevelMap`/`compat.supportsReasoningEffort` in models.json for that model only — do not disable reasoning globally. Then confirm OMO-level thinking works end to end:
   ```sh
   omo --offline --provider timely-scnu --model kimi-k3 --thinking high \
     --no-model-fallback --no-recommended-models --no-session --no-tools \
     -p 'What is 23*29? Show your reasoning.'
   ```
4. Failure handling by cause:
   - `not a valid model ID` -> model id typo; compare against the Step 2 list.
   - 401 / `Missing Authorization header` -> key missing or wrong; return to GATE 0.
   - `cannot switch: target context window ...` -> the model's `contextWindow` is set smaller than actual. Check the table in Step 2 first: if the model has a verified real limit smaller than what the guard needs, that model genuinely cannot serve this session (compact the session instead of raising the number). Only raise `contextWindow` when the table marks the model unverified and you have separately confirmed the real limit is actually higher (see GATE 1).
   - Multi-minute stall at startup -> add the `--offline` flag (it is a startup-phase issue, not a model call issue).
   - `reasoning_effort` rejected with 400 -> that model/backend does not accept the parameter; set `"reasoning": false` for that model (see Step 4 item 3).

## Security notes

- Never commit a real key to this manual or to any file tracked in git. The key must exist only in the user's local `~/.omo/agent/models.json`.
- If a key is exposed, rotate it in the Timely settings; rotating immediately invalidates the old key.
- The official `timely-gpt-sdk` (native SDK) uses the `sdk_live_` key scheme and is NOT compatible with `tgpt_sk_` keys. This setup bypasses the SDK and uses the OpenAI-compatible bridge directly.
