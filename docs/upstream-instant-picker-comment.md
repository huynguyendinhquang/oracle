# Draft comment for steipete/oracle branch `fix/chatgpt-instant-picker-loop`

> Maintaining a WSL2/Linux fork — sharing a reproducible failing case your branch's
> Instant work doesn't yet cover.
>
> `71b0f2f5 (bound Instant model selection)` and `cc9f4ccd (wait for current Intelligence pill)`
> handle the **legacy versioned-Instant via submenu** path (e.g. "5.3 Instant" behind
> `menu-item-submenu-chevron`) plus the picker re-open loop guard. But the **top-level
> "Instant" tier** is a different shape:
>
> In the current compact picker, the tiers are direct siblings:
> `[data-testid="composer-intelligence-picker-content"] > [role="menuitemradio"]` with text
> **Instant / Medium / High / Extra High / Pro** (no `data-testid`/`id`/`aria-label` on the
> rows; active = `aria-checked="true"`; radix ids are volatile). "Instant" is the **first**
> menuitemradio — a top-level tier, NOT a thinking-effort sub-setting and NOT behind a submenu.
>
> Repro: `oracle --engine browser -m gpt-5.5 --browser-model-strategy select --browser-thinking-time instant`
> → logs `Thinking time: selection unverified for thinking (requested Light); continuing with
> ChatGPT default` and silently keeps the active tier. Root cause: the effort step models Instant
> as `thinking + light`, but selecting the "Instant" row changes the model-kind away from
> "thinking", so the thinking-effort verification can't confirm it and falls back.
>
> Our fork's fix (happy to PR a slimmed version with a test if useful): when the requested level
> is light/instant and the open menu is `composer-intelligence-picker-content`, click the
> "Instant" `menuitemradio` by text and verify success via that row's `aria-checked="true"`
> (or the composer pill label flipping to "Instant"), rather than via thinking-effort state.
