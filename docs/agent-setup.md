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

Gemini/Claude families:
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

For GPT-family models use `contextWindow` 128000 (or omit to use the default), and 262144 for Kimi. Models with the `:batch` suffix must have `"reasoning": false`.

### Verified model list (as of 2026-09-03, confirmed by live bridge calls)

- GPT: `openai/gpt-5.5`, `openai/gpt-5.5:batch`, `openai/gpt-5.5-pro`, `openai/gpt-5.6-sol`, `openai/gpt-5.6-terra`, `openai/gpt-5.6-luna`
- Claude: `anthropic/claude-haiku-4.5` (+`:batch`), `anthropic/claude-sonnet-4.5`, `anthropic/claude-sonnet-4.6` (+`:batch`), `anthropic/claude-opus-4.7` (+`:batch`), `anthropic/claude-opus-4.8`, `anthropic/claude-sonnet-5`, `anthropic/claude-opus-5`, `anthropic/claude-fable-5`, `anthropic/claude-fable-5.1`
- Gemini: `google/gemini-3.5-flash`, `google/gemini-3.5-flash-lite`, `google/gemini-3.6-flash`, `google/gemini-3.8-flash`
- Kimi: `kimi-k3`, `moonshotai/kimi-k3` (absent from the public model list but confirmed working via authenticated calls)

Note: the list drifts. `curl -s https://hello.timelygpt.co.kr/api/v2/chat/bridge/info/models` shows the public list, but models missing from it can still work when called with authentication — a successful live call is the real criterion.

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
3. Failure handling by cause:
   - `not a valid model ID` -> model id typo; compare against the Step 2 list.
   - 401 / `Missing Authorization header` -> key missing or wrong; return to GATE 0.
   - `cannot switch: target context window ...` -> the model's `contextWindow` is set smaller than actual; raise Gemini/Claude entries to 1048576.
   - Multi-minute stall at startup -> add the `--offline` flag (it is a startup-phase issue, not a model call issue).

## Security notes

- Never commit a real key to this manual or to any file tracked in git. The key must exist only in the user's local `~/.omo/agent/models.json`.
- If a key is exposed, rotate it in the Timely settings; rotating immediately invalidates the old key.
- The official `timely-gpt-sdk` (native SDK) uses the `sdk_live_` key scheme and is NOT compatible with `tgpt_sk_` keys. This setup bypasses the SDK and uses the OpenAI-compatible bridge directly.
