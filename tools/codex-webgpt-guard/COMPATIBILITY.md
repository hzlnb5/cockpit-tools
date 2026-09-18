# Compatibility

## Supported platform

- Windows 10/11: supported
- macOS: daemon installation not implemented
- Linux: daemon installation not implemented

## Integration matrix

| Component | Support | Notes |
| --- | --- | --- |
| Native Codex | Yes | Used as the native fallback state |
| `codex-chatgpt-web` | Yes | First-class route-owner integration through its journal and route lifecycle |
| Cockpit Tools OAuth/profile switching | Yes | Guard repairs route loss caused by profile projection |
| Known Cockpit model catalogs | Yes | Removed only when they conflict with Web GPT |
| Microsoft Store Codex / ChatGPT app family | Yes | Fresh-process recovery is package/family aware |
| Custom `model_provider` | Preserved | Unknown provider ownership causes a conflict instead of takeover |
| Custom `openai_base_url` | Preserved | Different custom route is not overwritten |
| Unknown `model_catalog_json` | Preserved | Unknown catalog is not removed |
| Arbitrary third-party route journals | Not yet | Requires an explicit adapter/ownership model |

## Expected behavior by scenario

### Web GPT is running, account manager changes OAuth account

Expected:

1. account/profile state is allowed to change;
2. if the active Web route is removed, Guard restores it;
3. known conflicting Cockpit catalog state is removed;
4. unknown third-party provider/catalog state is not overwritten;
5. if Codex launched during the mutation, only a newly-started matching app family may be reloaded.

### Web GPT is closed

After the absence grace period, Guard asks `codex-chatgpt-web` to disconnect its route through the provider lifecycle and returns Codex to native routing.

### Web GPT starts again

Guard reconnects the route through the provider lifecycle. If Codex was launched milliseconds before the reconnect completed, the fresh-process recovery path can reload that just-started app instance so startup state matches the final route.

### A different router is active

If the config clearly belongs to another provider or custom route, Guard reports a conflict and does not take ownership.

## Compatibility philosophy

Compatibility is not defined as "the Guard always wins."

Compatibility means:

- known ownership is restored deterministically;
- unknown ownership is preserved;
- lifecycle-aware integrations remain internally consistent;
- existing unrelated sessions are not disrupted;
- failures are visible rather than silently rewritten.
