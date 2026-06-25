# Fork Decisions — Pending (huynguyendinhquang/oracle)

_Generated 2026-06-25. Decisions accumulated while maintaining the WSL2 fork of `@steipete/oracle`. PR work is intentionally deferred — priority is resolving local repo + skill issues first._

## Current state (verified, no decision needed)
- `main` @ `f8e958fc` — our 6 WSL/browser fixes are merged in; upstream merged (0.15.1, `--copy-profile`, deps).
- Tests: **1336 passed / 0 failed**. Build current (`dist` 06-22). Global `oracle` → this fork.
- Async-poll regression (our `e6d68fe0`) already re-fixed properly in `448ba21a` (sync IIFE + TS-side retry).
- Skill now canonical at `~/.agents/skills/oracle/` (real dir), exposed via the pre-existing `~/.claude/skills → ~/.agents/skills` parent symlink.

---

## D1 — Skill source-of-truth reconciliation
**Context:** Two oracle skills diverged:
- `~/.agents/skills/oracle/SKILL.md` (canonical install, **our** WSL tier-matrix, ~79 lines)
- repo `skills/oracle/SKILL.md` (**upstream's**, 141 lines, came via the upstream merge)

They diverged in *different directions* (ours adds WSL/tier specifics; upstream rewrote/expanded generically). Agents load the installed (`.agents`) copy.

**Options**
- **A.** Make repo `skills/oracle/SKILL.md` canonical → merge our WSL/tier content into it → re-install to `.agents`. (Ships skill with the fork; one source.)
- **B.** Keep `.agents` (our version) canonical → back-port useful upstream bits → optionally drop repo skill from our fork.
- **C.** Two-layer: keep upstream's repo skill untouched (clean rebases) + a thin machine-local `.agents` overlay with WSL specifics.

**Recommendation:** **A** — one source, shipped with the fork, survives reinstalls. Use `skill-creator` to merge.
**Effort:** ~30 min. **Risk:** low.

**Findings (2026-06-25, opus skill audit):** Our `.agents` copy = canonical base (**8.5/10** vs upstream **6/10**) — better triggering description, real progressive disclosure, only machine-verified copy. On every conflict, upstream is wrong/stale for this box (its `--model "5.5 Pro"` label claim + "browser=GPT+Gemini only" framing would actively mislead). **Merge IN from upstream** (condensed, de-`npx`-ed, WSL-corrected): prompt-template ("Oracle has zero project knowledge"), `--dry-run`/`--files-report` cost preview, ~196k token budget, safety/secrets, `--file` default-ignore semantics, API preflight (`doctor --providers`, `--no-azure`), 1Password key recipe, session-artifact internals. Workflow bits → SKILL.md (keep ~90-110 lines); reference-grade → `references/`. **Cut:** all `npx -y @steipete/oracle` prefixes, macOS `--copy-profile` path, `~/Projects/oracle` debug, remote-serve, upstream's weak description, the superseded label claim. **Sync the merged result to BOTH** `.agents` canonical AND repo `skills/oracle/` (real-dir copy). **→ Decision: A, plan locked.**

---

## D2 — Instant tier (compact-menu tier-mapping)
**Context:** Button-missing is fixed. Remaining: targeting `gpt-5.5 + instant` mis-maps to **Extra High** (the model/effort→single-tier unification isn't implemented; our fork treats model + effort as two steps, but the compact menu is one list).

**Options**
- **A.** Implement real compact-menu tier unification (port pi-oracle's `parseCompactIntelligenceSelection`: Instant/Medium/High/Extra-High → one click). Fixes Instant properly.
- **B.** Formalize the documented fallback: Instant only via `--engine api -m gpt-5.2-instant`; browser Instant unsupported.
- **C.** Wait for upstream's `fix/chatgpt-instant-picker-loop` branch to land, then rebase and re-test.

**Recommendation:** **C then A** — let their branch land (it touches the same area), rebase, then implement only the remaining gap with a TDD test. Instant is the lowest-value tier for an oracle (fastest/cheapest), so low priority.
**Effort:** A = medium (DOM automation). **Risk:** medium (fragile UI).

**Findings (2026-06-25, live agent-browser DOM inspection — corroborates user's puppeteer recording):** Authoritative, STABLE selectors now known:
- Compact tier menu container: `[data-testid="composer-intelligence-picker-content"]` (stable).
- Tiers = `[role="menuitemradio"]` rows inside it, in order **Instant, Medium, High, Extra High, Pro**. **No `data-testid`/`id`/`aria-label` on rows** → select by **text content** (or DOM order; Instant = first). Active row = `aria-checked="true"`. Trailing `<span data-state="checked">` also marks active.
- Pro effort submenu trigger: `[data-testid="composer-intelligence-pro-thinking-effort-trigger"]` (stable; `aria-label="Pro effort options"`; invisible until hover the `group/composer-intelligence-pro-row`) → `Pro Standard` / `Pro Extended` menuitemradios (by text).
- Legacy models submenu: `[data-testid="menu-item-submenu-chevron"]` → GPT-5.5/5.4/5.3/4.5/o3 menuitemradios.
- ⚠️ All `radix-_r_*` ids are **volatile** — never select by them.
- Screenshots: `…/scratchpad/chatgpt-picker-open.png`, `chatgpt-pro-effort-submenu.png`, `chatgpt-legacy-submenu.png`.

**Implied fix:** treat the tier as a **single `menuitemradio` click by text** inside `composer-intelligence-picker-content` (open picker → click "Instant"/"Medium"/… → verify `aria-checked`). Replaces the model+effort two-step that mis-maps Instant→Extra High. For Pro: click Pro row → hover → effort-trigger → Pro Standard/Extended. **Decision still C-then-A** (rebase on upstream branch first), but if upstream's branch stalls, this is now a well-scoped self-contained fix.

---

## D3 — Upstream PR strategy (deferred, but decide the plan)
**Context:** None of our 6 fixes are in upstream/main. Upstream README now says browser "works on Linux and Windows" (the macOS-only premise is **stale**) → WSL fixes are in-scope. Oracle (gpt-5.5 extra-high) advice saved at `/tmp/oracle-pr-advice.md`.

**Per-commit (oracle's verdict):**
| Fix | Verdict |
|---|---|
| Gemini evidence label | **PR** (platform-agnostic correctness) |
| `--browser-thinking-time` in `--help` | **PR** (trivial UX) |
| temp-dir `C:\Users` litter | **PR** (strongest WSL fix, precisely gated) |
| WSL2 GPU/sandbox flags | **PR** (note `--no-sandbox` tradeoff) |
| compact-menu button finder | **Coordinate** (overlaps their active branch) |
| poll-for-button | **Coordinate** (their branch adds related waiting) |

**Decision needed:** Confirm "defer all PRs for now" vs "open the 2 low-friction ones (Gemini + help) immediately." 
**Recommendation:** Per your stated priority — **defer all**; revisit after D1/D2.

---

## D4 — Living alongside upstream (fork maintenance cadence)
**Context:** Upstream moves fast (0.14→0.15 during our session). Long-lived divergence → rebase pain. Our async-poll already collided once.

**Options**
- **A.** Track upstream closely: `git fetch upstream && merge` regularly; upstream the platform-agnostic fixes so the carried delta shrinks.
- **B.** Pin to a known-good upstream tag; update deliberately.

**Recommendation:** **A** — minimize carried delta by upstreaming D3's PR-able fixes once D1/D2 settle. Keep only WSL-specific patches as the standing fork delta.

---

## D5 — `#4/#6` overlap with upstream `fix/chatgpt-instant-picker-loop`
**Context:** Their branch has `bound Instant model selection` + `wait for current Intelligence pill` — same area as our compact-menu + poll fixes.
**Decision:** Comment on their branch with our DOM repro (compact `button[aria-haspopup=menu]` + cold-start late render) and rebase after it lands — **don't** PR ours as-is.
**Recommendation:** Agree; low effort, avoids duplicate/competing work.

---

## D6 — Housekeeping (low stakes)
- **`wsl-support` branch:** delete (merged) or keep as marker? → **Recommend delete** (local + origin).
- **Junk `\\wsl.localhost\...` / `C:\Users...` untracked dirs:** add to `.gitignore` or just `rm`? → **Recommend** `.gitignore` a pattern + one-time clean.
- **Run `skill-creator` on the oracle skill now?** → Tied to D1; **yes** once D1 direction chosen.

---

## Suggested order
1. **D1** (skill reconciliation) — unblocks agents, low risk.
2. **D6** housekeeping (quick).
3. **D2/D5** Instant — wait on upstream branch, then decide.
4. **D3/D4** PRs — after the above, once carried delta is clear.
