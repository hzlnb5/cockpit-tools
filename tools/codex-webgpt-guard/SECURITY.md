# Security and Trust Model

## Scope

Codex WebGPT Guard is a local Windows utility that coordinates Codex routing state. It is not a network proxy and does not provide authentication for third-party services.

## Trust boundaries

The Guard trusts:

1. the current Windows user account;
2. the `codex-chatgpt-web` integration journal and runtime command under that user's profile;
3. the Codex config path recorded by that integration;
4. loopback-only endpoints that satisfy the Guard's route validation.

It does not treat arbitrary config values as owned state.

## Route validation

A Web GPT route is accepted only when it is:

- HTTP;
- loopback (`127.0.0.1`, `localhost`, or `::1`);
- explicitly bound to a port;
- using the `/v1` path.

## Unknown ownership

The Guard refuses to silently replace:

- a non-OpenAI custom `model_provider`;
- an unknown model catalog;
- a different custom API base URL.

## Process recovery

The launch-race recovery path is intentionally constrained.

It does not issue a broad:

```powershell
Get-Process ChatGPT,Codex | Stop-Process
```

Instead it considers only processes whose start time is tightly correlated with the triggering config write, resolves the app family, and reloads only the fresh matching family. Existing long-running sessions are not intentional recovery targets.

## Privileges

Installation is per-user and does not require administrator rights.

The Guard stores its installed binary and state under:

```text
%LOCALAPPDATA%\CodexWebGPTGuard\
```

Autostart is registered under the current user's Windows Run key.

## Network behavior

The Guard's own health checks are loopback-only. Route lifecycle actions are delegated to the configured `codex-chatgpt-web` runtime command.

## Binary trust

CI builds are not Authenticode-signed. Windows SmartScreen may therefore report an unknown publisher.

Build artifacts include `SHA256SUMS.txt` so the executable can be verified.
