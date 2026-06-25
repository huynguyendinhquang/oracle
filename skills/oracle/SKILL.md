---
name: oracle
description: Use when the user asks to consult the Oracle, use /oracle, get an external ChatGPT-web / GPT-5.x Pro second opinion, run heavy independent reasoning, validate architecture, review code with file context, or cross-check a design across models. Wraps the @steipete/oracle CLI + oracle-mcp `consult` tool to steer ChatGPT Web (browser, flat ChatGPT-Pro subscription) or call OpenAI/Gemini/Claude APIs, returning a durable answer other agents can read. Reach for this whenever a task benefits from a stronger external reasoner with real file context — even a casual "ask the oracle" / "get a second opinion".
---

# Oracle (@steipete/oracle)

Hand a prompt **plus file context** to a stronger external reasoner. Submission is a **direct tool call** (MCP `consult` or the `oracle` CLI) — no LLM middleman re-writes your prompt, so what you send is what the model sees. Treat the answer as **advisory**: verify against the codebase + tests.

Binaries: `oracle` (CLI), `oracle-mcp` (MCP server, the `consult` tool) — both installed, call them directly (no `npx`). Full flags: `oracle --help --verbose` and [references/cli-reference.md](references/cli-reference.md).

## Reliable lane on THIS machine (WSL2, no API keys set)

The verified, no-cost path is **ChatGPT Web (browser) on the flat Pro subscription**:

```bash
oracle --engine browser -m gpt-5.5-pro --browser-model-strategy current \
  --browser-attachments always \
  -p "<question + expected output shape>" --file "src/**/*.ts"
```

Three rules learned the hard way — follow them:

1. **Model: pick the tier with `-m` + `--browser-thinking-time` (verified matrix below), or pin `gpt-5.5-pro` / use `--browser-model-strategy current`.** Do NOT request `gpt-5.2-instant` or tiers by version-name — the live picker shows short labels (Instant / Medium / High / Extra High / Pro) and version-prefixed ids fail ("Unable to find model option").
2. **Files: always pass `--browser-attachments always`.** The default `auto` *pastes file text inline into the prompt* up to ~60k chars and only uploads above that — so small files are NOT real attachments. `always` forces a real ChatGPT file upload every time. Always include ≥1 `--file`.
3. **Claude/multi-model → API engine, not browser.** Grok = API-only too (browser dropped). Gemini-web works in browser (see matrix). API needs keys (see below).

Session reuse: `--copy-profile <chrome-user-data-dir>` is the supported built-in way to reuse an already-signed-in Chrome session — copies the profile to a throwaway dir, runs against it (real keychain → cookies decrypt), then deletes the copy (mirrors the agent-browser no-Cloudflare approach). Full note in references.

### ChatGPT browser tier → CLI (verified 2026-06-18 on this box)

The current ChatGPT UI is one **compact menu**: Instant / Medium / High / Extra High / Pro. Effort = `--browser-thinking-time`, NOT a version-named model. Map (use `--browser-model-strategy select`, default):

| Tier | Flags |
|---|---|
| Medium | `-m gpt-5.5 --browser-thinking-time medium` |
| High | `-m gpt-5.5 --browser-thinking-time high` |
| Extra High | `-m gpt-5.5 --browser-thinking-time extra-high` |
| Pro (Standard) | `-m gpt-5.5-pro --browser-thinking-time standard` |
| Pro Extended | `-m gpt-5.5-pro --browser-thinking-time extended` |
| **Instant** | ⚠️ Known limitation (as of 2026-06-25): browser-select of Instant silently **falls back to the currently-active tier** (logs `selection unverified … continuing with ChatGPT default`) — ChatGPT's compact menu treats Instant as a top-level tier, not a thinking-effort, so oracle's effort step can't verify it. Fix in progress. Get true Instant via `--engine api -m gpt-5.2-instant`, or accept a higher browser tier; **re-verify after upgrades**. |

So "oracle, run gpt-5.5 extra high" → `oracle --engine browser -m gpt-5.5 --browser-model-strategy select --browser-thinking-time extra-high --browser-attachments always -p "..." --file ...`.

### Gemini browser (works as of 2026-06-18 fix)

`oracle --engine browser -m gemini-3.1-pro -p "..."` (also `gemini-3.5-flash`, `gemini-3.1-flash-lite`, `gemini-3-pro-deep-think`). Header-based model select (RPC, not DOM) — fast (~3s). Cookies auto-import from Linux Chrome; if a cookie-decrypt warning appears, prefix `SWEET_COOKIE_LINUX_KEYRING=basic`. (Evidence now reports the real model, not the old false "unavailable".)

### WSL env (required)

Already persisted in `~/.bashrc` and set in the MCP server env:
- `CHROME_PATH=/usr/bin/google-chrome` — force Linux Chrome, not Windows chrome.exe.
- `ORACLE_BROWSER_REMOTE_DEBUG_HOST=127.0.0.1` — loopback CDP, avoids the Windows-chrome / Tailscale-IP (100.x) ECONNREFUSED.

Cookie auth reads `~/.config/google-chrome/Default` — be logged into ChatGPT there.

## Pick the file set & preview before you spend

Fewer files + a better prompt beats whole-repo dumps. Pick the tightest set that still contains the truth, then preview before sending:

```bash
oracle --dry-run summary --files-report -p "<task>" --file "src/**" --file "!**/*.test.*"
```

- `--dry-run summary|full|json` previews the assembled request without calling any model (no tokens spent). `--files-report` shows the token accounting so you can spot the hogs.
- Budget target: keep total input under ~196k tokens.
- `--file` honors `.gitignore`, skips default-ignored dirs (`node_modules`, `dist`, `.git`, `.next`, `build`, `tmp`, …), does not follow symlinks, filters dotfiles unless you opt in, and rejects files >1 MB. Full semantics in references.

## Prompt template (Oracle has zero project knowledge)

Oracle cannot infer your stack, build tooling, conventions, or "obvious" paths — each run is one-shot with no memory of prior runs. Include:

- **Project briefing** — stack + build/test commands + platform constraints.
- **Where things live** — key dirs, entrypoints, config files, dependency boundaries.
- **Exact question** — what you tried + the error text (verbatim).
- **Constraints** — "don't change X", "keep public API", perf budget.
- **Desired output** — "return a patch plan + tests", "3 options with tradeoffs", "list risky assumptions".

For long investigations, write an "exhaustive prompt" that stands alone later (briefing on top, repro + errors in the middle, all needed context files attached at the bottom) so a fresh model can fully understand it.

## API engine (optional — needs keys, costs money, requires consent)

Reliable, no browser fragility, but costs tokens and sends context to the provider. Only when the user explicitly opts in and the key exists:

```bash
oracle -p "Cross-check this" --models gpt-5.5-pro,gemini-3.1-pro,claude-4.6-sonnet --file "src/**/*.ts"
```

Keys: `OPENAI_API_KEY`, `GEMINI_API_KEY`, `ANTHROPIC_API_KEY`. Check first: `oracle doctor --providers`. `--models` runs in parallel with aggregated cost; `--allow-partial` tolerates failures. For first-party OpenAI (avoid Azure hijack) use `--provider openai` / `--no-azure`. Preflight + 1Password key recipe in references.

## Long Pro runs are detached — recover, don't re-run

GPT-5.x Pro runs detach by default (can take ~10 min to ~1 hour). On timeout do NOT re-run (the duplicate-prompt guard blocks it):

```bash
oracle status            # recent sessions
oracle session <id>      # reattach + capture the final answer
```

Or auto-poll: `--browser-auto-reattach-delay 30s --browser-auto-reattach-interval 2m`. Continue a thread: repeat `--browser-follow-up "<next>"`, or resume later with `--followup <sessionId>`. Use `--slug "<3-5 words>"` for readable session ids.

## Parallel jobs

Genuinely supported (unlike the old single-session setup):
- **API**: `--models a,b,c` runs in parallel in one command.
- **Browser**: `--browser-max-concurrent-tabs N` with a tab-lease registry over one signed-in profile. Still bounded by the single ChatGPT account's rate limits — keep N small (2-3) for heavy Pro runs.

## MCP (preferred for agents)

`oracle-mcp` exposes the `consult` tool (wired into Claude Code / Codex / opencode). In a `consult` call use `preset: "chatgpt-pro-heavy"` for browser GPT-5.5 Pro + Pro Extended thinking; add `dryRun: true` to inspect the resolved run without touching Chrome. Re-generate client config: `oracle bridge claude-config --local-browser` / `oracle bridge codex-config`.

## Safety

- Don't attach secrets by default (`.env`, key files, auth tokens). Redact aggressively; share only what's required.
- Prefer "just enough context": fewer files + a sharper prompt beats a whole-repo dump.

## Reporting back

Report the engine/model used, the session id (so the user can reattach), and the answer. A timed-out Pro run is usually still finishing — recover via `oracle session <id>`, don't treat it as failure.
