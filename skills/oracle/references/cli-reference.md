# @steipete/oracle — CLI flag reference (v0.14.x)

Source of truth: `oracle --help` / `oracle --help --verbose`. Re-check after upgrades. Use the installed `oracle` / `oracle-mcp` binaries directly (no `npx`).

## Contents
- [Engine / models](#engine--models)
- [Prompt & files](#prompt--files-both-required)
- [`--file` default behavior](#--file-default-behavior)
- [Browser](#browser)
- [Sessions](#sessions)
- [Output / readiness](#output--readiness)
- [API preflight & routing](#api-preflight--routing)
- [1Password key injection](#1password-key-injection-this-box)
- [MCP / bridge / remote](#mcp--bridge--remote)
- [WSL2 required env](#wsl2-this-box--required-env)

## Engine / models
- `-e, --engine <api|browser>` — browser automates ChatGPT (GPT) / Gemini web; api calls providers. Auto: api if `OPENAI_API_KEY` set, else browser.
- `-m, --model <model>` — default `gpt-5.5-pro`. GPT: `gpt-5.5`, `gpt-5.4-pro/5.4`, `gpt-5.1-pro/5.1`, `gpt-5.1-codex` (API-only), `gpt-5.2`, `gpt-5.2-instant`, `gpt-5.2-pro`. Gemini: `gemini-3.1-pro`, `gemini-3.5-flash`, `gemini-3.1-flash-lite`, `gemini-3-pro-deep-think`. Claude: `claude-4.6-sonnet`, `claude-4.1-opus`. Plus any OpenRouter id.
- `--models a,b,c` — parallel multi-model (API), aggregated cost; `--allow-partial`.
- ⚠️ **Browser model-label drift:** the live ChatGPT picker is one compact menu `Instant / Medium / High / Extra High / Pro`. Pick tier via `-m gpt-5.5 --browser-thinking-time medium|high|extra-high` (Thinking tiers) or `-m gpt-5.5-pro --browser-thinking-time standard|extended` (Pro). `gpt-5.2-instant` by version-name fails — for browser Instant use `-m gpt-5.5 --browser-thinking-time instant` (selects the top-level compact-menu tier; fixed 2026-06-26, DOM-fragile so re-verify after UI changes; `--engine api -m gpt-5.2-instant` is the fallback). Gemini browser works (`-m gemini-3.1-pro`, header/RPC select). Grok = API-only (no browser).

## Prompt & files (both required)
- `-p, --prompt <text>`; `-f, --file <paths...>` (globs, `!` exclude). Files >1MB rejected (`--max-file-size-bytes <bytes>` or `ORACLE_MAX_FILE_SIZE_BYTES` / `config.maxFileSizeBytes`).
- `--browser-attachments <auto|never|always>` — default `auto` inlines ≤~60k chars then uploads; **`always` = real upload every time (use this)**; `never` = inline. `--browser-bundle-files [--browser-bundle-format text|zip]` uploads many files as one bundle (zip preserves original bytes).
- `--files-report` token accounting.

## `--file` default behavior
`--file` accepts literal files, directories, and globs; repeatable; entries can be comma-separated. Exclude with a leading `!` (e.g. `--file "src/**" --file "!src/**/*.test.ts"`). Defaults from the implementation:
- **Default-ignored dirs** (skipped unless passed explicitly): `node_modules`, `dist`, `coverage`, `.git`, `.turbo`, `.next`, `build`, `tmp`.
- **Honors `.gitignore`** when expanding globs.
- **No symlink follow** (glob expansion uses `followSymbolicLinks: false`).
- **Dotfiles filtered** unless the pattern includes a dot-segment (e.g. `--file ".github/**"`).
- **1 MB cap** per file unless raised via `ORACLE_MAX_FILE_SIZE_BYTES` / `--max-file-size-bytes` / `~/.oracle/config.json` `maxFileSizeBytes`.

## Browser
- `--browser-model-strategy <select|current|ignore>` — `current` = keep active model (robust); `select` (default) drives the picker.
- `--browser-thinking-time <level>` — ChatGPT tier/effort: `light|standard|extended|heavy` or UI aliases `instant|medium|high|extra-high`.
- `--copy-profile <dir>` — reuse an already signed-in Chrome session with no manual login: copies the profile to a throwaway dir, launches with the real keychain so cookies decrypt, runs, then **always deletes the copy** (failed/incomplete runs deleted too, so they can't be reattached). macOS/Linux; needs `rsync`. The supported built-in way to reuse the logged-in session (mirrors agent-browser's no-Cloudflare method).
- `--browser-manual-login`, `--browser-keep-browser`, `--browser-input-timeout <ms>`.
- `--browser-attach-running` + `--browser-port` / `--remote-chrome host:port`, `--browser-tab <ref>`, `--chatgpt-url`.
- `--browser-auto-reattach-delay|-interval|-timeout`, `--browser-follow-up <prompt>` (repeatable), `--browser-research deep`, `--browser-archive <never|always>`, `--browser-max-concurrent-tabs N`.

## Sessions
- Detached by default (Pro). Stored under `~/.oracle/sessions` (override `ORACLE_HOME_DIR`). Browser runs save durable files under `~/.oracle/sessions/<id>/artifacts/` — `transcript.md`, Deep Research reports, downloaded ChatGPT images.
- `oracle status [id]` (`--hours N`), `oracle session [id] --render` (`--all`, `--clear --hours N`), `restart <id>`.
- `--followup <sessionId|responseId>`, `--followup-model <model>`, `--wait`, `--force` (override duplicate-prompt guard), `--slug <name>`.
- Guardrails: root run without a prompt exits nonzero; `--dry-run` conflicts with `--render` / `--render-markdown`; Ctrl-C exits foreground API runs with code 130 while browser cleanup/reattach still runs.

## Output / readiness
- `--dry-run <summary|json|full>` previews the request without calling a model (prompt required).
- `--render` / `--render-plain`, `--copy` / `--copy-markdown` (`--copy` is a hidden alias), `--write-output <path>` (+`.oracle.json` manifest for multi-model).
- `--perf-trace` / `--perf-trace-path <path>` (or `ORACLE_PERF_TRACE=1`) — startup & first-output timing; traces redact prompts, tokens, keys, cookies. Inspect `first-output` and `exit` when CLI startup feels slow.
- `oracle doctor [--providers]`, `oracle --preflight`, `oracle bridge doctor` (browser prereqs).

## API preflight & routing
API runs cost money and require **explicit user consent**. Check readiness without printing secrets:
- `oracle doctor --providers --models gpt-5.4,claude-4.6-sonnet,gemini-3-pro`
- `oracle --preflight --models gpt-5.4,gemini-3-pro`
- `oracle --route --model gpt-5.4` (prints the provider route plan and exits).
- First-party OpenAI (prevent exported Azure env/config from hijacking the route): `--provider openai` or `--no-azure` — e.g. `oracle --provider openai --engine api --model gpt-5.5-pro ...`.
- Advisory multi-model panels: `--models a,b,c --allow-partial --write-output /tmp/panel.md` keeps successful model files + `<stem>.oracle.json`.
- `--timeout 10m` is the user-facing API deadline; Oracle derives the HTTP transport timeout unless `--http-timeout` is set.

## 1Password key injection (this box)
If the exported `OPENAI_API_KEY` is invalid and the user wants their personal OpenAI key, inject from 1Password into the single Oracle command only — **never print the key**. Known item: `API Key - OpenAI - Personal`, field `api_key`:
```bash
OPENAI_API_KEY="$(op item get 'API Key - OpenAI - Personal' --account my.1password.com --fields label=api_key --reveal)" \
  oracle --provider openai --engine api --model gpt-5.5-pro -p "<task>" --file "src/**"
```

## MCP / bridge / remote
- `oracle-mcp` stdio server; `consult` tool params: `engine`, `model`, `preset` (`chatgpt-pro-heavy`), `browserFollowUps`, `browserResearchMode`, `generateImage`, `dryRun`.
- `oracle bridge claude-config [--local-browser]` / `codex-config`; `oracle serve --host 0.0.0.0 --port 9473 --token <secret>` + client `--remote-host <host:port> --remote-token <secret>`; `oracle bridge host|client` (Windows↔WSL).

## WSL2 (this box) — required env
- `CHROME_PATH=/usr/bin/google-chrome` (use Linux Chrome, not Windows chrome.exe).
- `ORACLE_BROWSER_REMOTE_DEBUG_HOST=127.0.0.1` (loopback CDP, avoid Tailscale 100.x ECONNREFUSED).
- Both persisted in `~/.bashrc` and set in the MCP server env. Cookie auth reads `~/.config/google-chrome/Default` (be logged into ChatGPT there).
- Gemini cookie-decrypt warning fix: prefix `SWEET_COOKIE_LINUX_KEYRING=basic`.
