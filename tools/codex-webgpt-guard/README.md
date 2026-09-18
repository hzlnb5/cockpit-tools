# Codex WebGPT Guard

Windows compatibility guard for **Cockpit Tools + codex-chatgpt-web**.

It does **not** patch Cockpit, Codex, or codex-chatgpt-web binaries. It is designed to keep working across ordinary Cockpit updates.

## What it does

Default mode is `auto`:

- If the Web GPT launcher/runtime is live, it keeps the codex-chatgpt-web route connected.
- If Cockpit switches a normal OAuth account and rewrites `config.toml`, it restores only the Web GPT loopback route that is recorded in codex-chatgpt-web's own integration journal.
- If Web GPT is closed for a short grace period, it uses codex-chatgpt-web's **official `route disconnect` lifecycle** to restore native Codex routing and keep the Web GPT journal internally consistent.
- When Web GPT comes back, it uses the official **`route connect` lifecycle** to reconnect it.
- It removes only known Cockpit-managed model catalog references when Web mode is active and those references would replace the Web GPT model catalog.
- It refuses to overwrite an unknown custom provider, unknown model catalog, or another custom `openai_base_url`.
- Its launch-race fallback only reloads a freshly-started matching ChatGPT/Codex package family; it never intentionally terminates an older session.

So your normal workflow is simply:

- Want Web GPT: open Codex Web GPT, then choose an account in Cockpit and press the launch button.
- Want native Codex: close Codex Web GPT, then choose an account in Cockpit and press the launch button.

No Cockpit source patch is required for the Guard itself.

## Install

Easiest: double-click `INSTALL.cmd`.

Or run once from PowerShell:

```powershell
.\codex-webgpt-guard.exe install
```

It copies itself to:

```text
%LOCALAPPDATA%\CodexWebGPTGuard\codex-webgpt-guard.exe
```

and registers a **current-user** startup entry. No administrator rights are required.

## Status / modes

Double-click `STATUS.cmd`, or use:

```powershell
codex-webgpt-guard.exe status
codex-webgpt-guard.exe mode auto
codex-webgpt-guard.exe mode web
codex-webgpt-guard.exe mode native
```

`auto` is recommended.

- `auto`: live Web GPT => connected; Web GPT closed => native route.
- `web`: keep Web GPT route connected even if its health endpoint is temporarily unavailable.
- `native`: safely disconnect the Web GPT route using codex-chatgpt-web's own route lifecycle.

## Uninstall

Double-click `UNINSTALL.cmd`, or:

```powershell
codex-webgpt-guard.exe uninstall
```

This removes autostart and asks the background Guard to exit. It intentionally leaves the small install directory in place so it never needs administrator privileges or self-delete tricks.

## Safety / ownership rules

The Guard only accepts a Web GPT route from the codex-chatgpt-web journal when it is an HTTP loopback `/v1` endpoint (`127.0.0.1`, `localhost`, or `::1`).

It will not silently take over:

- `model_provider` owned by another provider;
- an unknown `model_catalog_json`;
- a different custom `openai_base_url`.

Logs are written to:

```text
%LOCALAPPDATA%\CodexWebGPTGuard\guard.log
```


## Windows trust notice

The CI-built executable is reproducible from the source in this repository but is **not Authenticode-signed**. Windows SmartScreen may therefore show an Unknown publisher warning. The package includes `SHA256SUMS.txt` so the binary can be verified before installation.
