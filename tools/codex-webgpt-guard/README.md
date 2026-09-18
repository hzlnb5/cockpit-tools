# Codex WebGPT Guard

> An ownership-aware route reconciliation layer for Codex desktop integrations on Windows.

[![Build Codex WebGPT Guard](https://github.com/hzlnb5/cockpit-tools/actions/workflows/build-codex-webgpt-guard.yml/badge.svg)](https://github.com/hzlnb5/cockpit-tools/actions/workflows/build-codex-webgpt-guard.yml)

Codex WebGPT Guard started as a compatibility fix, but the underlying problem is broader than any one pair of tools.

Modern Codex setups often have several independent programs touching the same shared control plane:

- account/profile switchers;
- local Responses API bridges;
- custom model routers;
- model catalog injectors;
- desktop wrappers and launchers;
- native Codex itself.

All of them can mutate the same `~/.codex/config.toml`. Without a shared ownership model, the last writer wins. That creates a class of failures that look random from the UI: models disappear, a route silently falls back to native, a router reports an inconsistent journal, a model catalog becomes stale, or Codex starts milliseconds before the intended route is restored.

Codex WebGPT Guard is a small user-space reconciliation layer for that problem.

## The deeper problem

The important issue is **configuration ownership**, not one specific bug.

A typical conflict looks like this:

```text
Account manager                  Route provider
      |                                |
      | writes auth / provider state   | owns local Responses route
      v                                v
             ~/.codex/config.toml
                     |
                     v
                  Codex
```

If the account manager rewrites the file and removes `openai_base_url`, the route provider is still running but Codex no longer points to it. If the route provider blindly writes everything back, it may destroy a legitimate third-party provider. If both sides keep rewriting the file, they can create a write loop or startup race.

The Guard solves this by separating **ownership**, **lifecycle**, and **reconciliation**.

## What the Guard does

### 1. Treats route ownership as explicit state

For `codex-chatgpt-web`, the Guard reads the integration journal written by the route owner itself. It does not guess a port and it does not hard-code `127.0.0.1:17841`.

The journal supplies the expected Codex config path and the route that belongs to Web GPT.

### 2. Uses the route owner's lifecycle instead of fighting it

When switching between Web and native operation, the Guard calls the route provider's own:

```text
route connect
route disconnect
```

That keeps the provider's journal, realtime route, hooks, model cache, and previous-state restoration internally consistent.

### 3. Repairs only fields whose ownership is known

If an account/profile manager changes authentication state but accidentally removes the active Web GPT route, the Guard restores the route owned by Web GPT.

It may also remove a model catalog only when it is a **known Cockpit-managed catalog** that would replace the Web GPT catalog.

Unknown ownership is a hard boundary. The Guard refuses to silently take over:

- an unknown `model_provider`;
- an unknown `model_catalog_json`;
- a different custom `openai_base_url`.

### 4. Handles launch races

Configuration repair alone is not sufficient if Codex has already started and cached the wrong catalog or route.

The Guard watches the config write that immediately precedes a launch. If recovery is required, it only reloads a **freshly-started matching ChatGPT/Codex package family** whose start time correlates with that write.

Older sessions are intentionally left alone.

### 5. Runs outside the tools it coordinates

The Guard does not patch Cockpit, Codex, or codex-chatgpt-web executables.

It installs as a per-user background process under:

```text
%LOCALAPPDATA%\CodexWebGPTGuard\
```

This makes it resilient to ordinary updates of the applications it coordinates.

## What problem does this solve beyond Cockpit + Web GPT?

The current implementation has first-class knowledge of **codex-chatgpt-web as the route owner** and of several **Cockpit-managed mutations**, but the design addresses a more general integration problem:

### Shared mutable configuration

Multiple desktop tools can independently modify one Codex config file without a transaction or ownership protocol.

### Route ownership drift

A local API bridge can still be healthy while another tool silently removes the route that points Codex to it.

### Model catalog interference

One tool can inject a catalog while another expects the native or routed catalog, producing confusing model-picker behavior.

### Lifecycle mismatch

Changing one field manually may leave the route provider's own journal or restore state inconsistent. The Guard prefers lifecycle-aware connect/disconnect operations.

### Startup ordering races

The final config state can be correct while the already-running Codex process still holds stale startup state. The Guard includes a deliberately narrow fresh-process recovery path for this case.

### Unsafe "fix everything" automation

A generic watcher that overwrites every change is dangerous. This project instead follows a **fail-closed ownership model**: known ownership may be reconciled; unknown ownership is left untouched and reported as a conflict.

## Current compatibility model

The architecture is reusable, but this build is intentionally conservative.

| Integration | Current behavior |
| --- | --- |
| `codex-chatgpt-web` route journal | First-class owner integration |
| Cockpit OAuth/profile switching | Supported conflict source |
| Known Cockpit model catalogs | Can be removed when they conflict with Web GPT |
| Native Codex | Supported fallback state |
| Unknown custom router/provider | Preserved; Guard refuses takeover |
| Arbitrary third-party route journals | Not yet implemented |
| macOS / Linux daemon install | Not implemented; Windows is the supported target |

So this is **not** a universal router that claims every Codex configuration. It is an ownership-aware coordinator with one mature route-owner adapter today.

## State model

Default mode is `auto`.

```text
Web GPT live + healthy
        |
        v
ensure Web route is connected
        |
        +-- external account switch removes route --> repair owned route
        |
        +-- Codex launched during repair -----------> reload only fresh matching app family

Web GPT absent for grace period
        |
        v
route disconnect via provider lifecycle
        |
        v
native Codex state
```

Manual modes are also available:

```powershell
codex-webgpt-guard.exe mode auto
codex-webgpt-guard.exe mode web
codex-webgpt-guard.exe mode native
```

## Installation

Download the Windows build artifact, extract it, then run:

```text
INSTALL.cmd
```

or:

```powershell
.\codex-webgpt-guard.exe install
```

No administrator rights are required.

After installation, normal usage remains unchanged:

```text
Want Web models:
open Codex Web GPT -> choose an account/profile -> launch Codex

Want native Codex:
close Codex Web GPT -> choose an account/profile -> launch Codex
```

The Guard handles route reconciliation in the background.

## Status and troubleshooting

```powershell
codex-webgpt-guard.exe status
```

Logs:

```text
%LOCALAPPDATA%\CodexWebGPTGuard\guard.log
```

Uninstall:

```text
UNINSTALL.cmd
```

or:

```powershell
.\codex-webgpt-guard.exe uninstall
```

## Safety properties

The current implementation deliberately includes several guardrails:

- only HTTP loopback `/v1` routes are accepted as Web GPT routes;
- launcher descriptors must match the expected Web GPT launcher kind;
- unknown providers, catalogs, and custom routes are not overwritten;
- config writes are atomic;
- only one Guard instance runs per user session;
- launch-race recovery is bounded in time and limited to freshly-started matching app families;
- older ChatGPT/Codex sessions are not intentionally terminated;
- route connect/disconnect uses codex-chatgpt-web's own lifecycle where available;
- CI checks that the hardening is present before producing a Windows binary.

See [SECURITY.md](SECURITY.md) for the trust model.

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) for the ownership model, state machine, reconciliation rules, and launch-race handling.

See [COMPATIBILITY.md](COMPATIBILITY.md) for supported integrations and expected behavior.

## Build from source

The repository contains the audited source archive plus the hardening patch used by CI. The build workflow applies the hardening, verifies the safety checks, runs tests and vet, then produces the Windows binary.

```text
tools/codex-webgpt-guard/source.zip
tools/codex-webgpt-guard/hardening-v2.patch
.github/workflows/build-codex-webgpt-guard.yml
```

## Project direction

The long-term abstraction is a **Codex route ownership coordinator** rather than a special-case patcher.

A future version could support multiple explicit adapters:

```text
Codex Route Guard
  |- codex-chatgpt-web adapter
  |- account/profile manager adapters
  |- local router adapters
  |- model catalog ownership rules
  `- conflict reporting / policy engine
```

The key rule should remain the same: **reconcile only what you can prove you own; fail closed on everything else.**

## Windows trust notice

CI builds are reproducible from the source in this repository but are not Authenticode-signed. Windows SmartScreen may show an Unknown publisher warning. Build artifacts include `SHA256SUMS.txt` for verification.
