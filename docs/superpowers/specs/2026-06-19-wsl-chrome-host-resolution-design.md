# WSL Chrome remote-debug host resolution fix

Date: 2026-06-19
Status: Approved (Approach A)

## Problem

On WSL with a **Linux-local Chrome** (`CHROME_PATH=/usr/bin/google-chrome`) and no
`ORACLE_BROWSER_REMOTE_DEBUG_HOST` / `WSL_HOST_IP` env set, the browser engine fails with:

```
ERROR: connect ECONNREFUSED 10.255.255.254:9222
User error (browser-automation): connect ECONNREFUSED 10.255.255.254:9222
```

### Root cause

`launchChrome()` (`src/browser/chromeLifecycle.ts:20`) calls `resolveRemoteDebugHost()`.
On WSL with no env override, that function reads `/etc/resolv.conf` and returns the
`nameserver` IP (e.g. `10.255.255.254`) as the Chrome remote-debug host
(`chromeLifecycle.ts:676-681`).

Because the returned host `!= "127.0.0.1"`, `launchChrome` assumes a **Windows-host
Chrome**: binds `--remote-debugging-address=0.0.0.0`, takes the patched-launcher path,
and connects to `10.255.255.254:9222` (`chromeLifecycle.ts:21-24`).

But oracle **launched Chrome itself as a local child process** (Linux Chrome inside WSL).
That child listens locally; nothing listens at the WSL DNS gateway. Result: `ECONNREFUSED`.

The heuristic conflates "WSL's DNS nameserver" with "where my Chrome child listens" —
unrelated facts. `10.255.255.254` is the WSL virtual DNS, never a Chrome endpoint. The
inference is only valid for the narrow "drive a Windows `chrome.exe` from WSL" case.

### Trigger vs root cause

- **Trigger:** a background/non-login shell does not source `~/.bashrc`, so the user's
  normally-set `ORACLE_BROWSER_REMOTE_DEBUG_HOST=127.0.0.1` is absent and the broken
  fallback heuristic runs. Setting the env inline is a band-aid that breaks again in any
  non-login shell (background tasks, cron, MCP spawn).
- **Root cause:** the resolv.conf inference itself returns a bad host for Linux-local Chrome.

## Decision — Approach A (gate inference on Windows-Chrome binary)

Most contained blast radius. Only infer the resolv.conf host when oracle actually drives a
Windows `chrome.exe`. For Linux-local Chrome, return `null` so the launch path uses
`127.0.0.1`. Reuses the existing `isWindowsChromeBinary` detection already shipped for the
WSL temp-dir fix (`src/browser/index.ts:3856`).

Rejected:
- **B (drop inference entirely):** widest blast radius — silently breaks anyone relying on
  auto Windows-Chrome inference; changes an upstream-shared contract.
- **C (probe + fallback):** keeps the wrong abstraction, adds per-launch latency, masks intent.

## Changes

1. `src/browser/chromeLifecycle.ts` — in `resolveRemoteDebugHost()`, after the `isWsl()`
   check and before reading `/etc/resolv.conf`, return `null` unless
   `isWindowsChromeBinary(process.env.CHROME_PATH)`.
2. `src/browser/chromeLifecycle.ts` — add a local `isWindowsChromeBinary` helper (the
   existing one in `index.ts` is not exported; importing it into the lower-level
   `chromeLifecycle.ts` risks a circular import — mirror the existing local-`isWsl` pattern).

## Net effect

- WSL + Linux Chrome, no env → `resolveRemoteDebugHost()` returns `null` → local launch +
  connect on `127.0.0.1`. Env-less background shell no longer breaks.
- WSL + Windows `chrome.exe` → unchanged (still infers WSL host IP).
- Non-WSL → unchanged (already returns `null`).
- Explicit `ORACLE_BROWSER_REMOTE_DEBUG_HOST` / `WSL_HOST_IP` override → still honored first.

## Verification

1. `npm run build`.
2. Repro: WSL, `CHROME_PATH=/usr/bin/google-chrome`, unset `ORACLE_BROWSER_REMOTE_DEBUG_HOST`
   → small browser consult → connects `127.0.0.1`, no `ECONNREFUSED 10.255.255.254`.
3. Unit: `resolveRemoteDebugHost()` returns `null` for a Linux path; returns the nameserver
   for an `.exe` / `/mnt/c` path (with `WSL_DISTRO_NAME` set).
