# Context Sync

Context Sync is a proposed cross-platform desktop application that combines:

- [OpenCodex](https://github.com/lidge-jun/opencodex)'s local provider proxy and
  Codex model catalog integration; and
- [CC Switch](https://github.com/farion1231/cc-switch)'s provider, MCP, prompt,
  skill, plugin-adjacent configuration, and WebDAV synchronization workflows.

The project is currently in **requirements discovery**. Implementation will
start after the product boundaries and security model below are confirmed.

## Current understanding

The initial product should:

1. Run on Windows, macOS, and Linux.
2. Let an already-running Codex session use different provider-backed models
   without restarting Codex.
3. Manage API providers and subscription/account metadata from one desktop UI.
4. Manage and synchronize skills, MCP servers, prompts, and the agreed set of
   plugin-like configuration.
5. Synchronize supported data between devices through WebDAV only in the first
   release.
6. Reuse upstream implementations rather than independently rebuilding their
   mature features.

## Proposed integration boundary

The lowest-effort design is to use **CC Switch as the desktop shell and source
of truth** (Tauri, React, Rust, SQLite, configuration adapters, skills/MCP
management, and WebDAV), while running **OpenCodex as a managed local sidecar**
for request routing and Codex model-catalog injection.

This composition avoids merging two unrelated user interfaces and avoids
reimplementing protocol adapters. Context Sync would add a narrow integration
layer that:

- starts, stops, upgrades, and health-checks the OpenCodex sidecar;
- projects the selected CC Switch provider/account configuration into an
  OpenCodex-compatible runtime configuration;
- refreshes OpenCodex routing/catalog state when configuration changes;
- keeps secrets out of logs and conflict payloads; and
- exposes sidecar and synchronization status in the desktop UI.

"No restart" must be specified precisely. A local proxy can change the route
used by the **next request** without restarting the Codex process. It cannot
guarantee that every Codex client will update its visible model picker in the
middle of an existing conversation, nor can it safely move an in-flight request
to another provider. Those cases need explicit acceptance rules.

## Draft data ownership

| Data | Proposed owner | WebDAV default |
| --- | --- | --- |
| Provider definitions and non-secret metadata | CC Switch database | Yes |
| API keys, OAuth refresh tokens, session cookies | OS credential store | No |
| OpenCodex routing/model aliases | Generated projection | Yes, via source records |
| Skills | CC Switch skill manager | Yes |
| MCP server definitions | CC Switch MCP manager | Yes |
| Prompts/instruction files | CC Switch prompt manager | Yes |
| Device paths, autostart, window state | Local device settings | No |
| Usage history and request logs | Local database | No by default |

## Decisions required before implementation

1. **Codex surfaces:** Does "Codex" mean the CLI/TUI only, or must the Codex
   desktop app and IDE extension also support hot switching?
2. **Switch semantics:** Is it sufficient for the next user turn/new request to
   use the newly selected model, or must the native picker and existing
   conversation change immediately as well?
3. **Product scope:** Is the first release Codex-only, or should it retain all
   CC Switch targets (Claude Code/Desktop, Gemini CLI, Grok Build, OpenCode,
   OpenClaw, and Hermes)?
4. **Provider scope:** Which official subscriptions and third-party API
   providers are mandatory for the first release? Are OAuth-based ChatGPT/Claude
   subscriptions in scope, or API keys only?
5. **Secret synchronization:** Should WebDAV include encrypted credentials? If
   yes, users need a separate end-to-end encryption passphrase and defined key
   recovery behavior.
6. **Plugin definition:** Does "plugins" mean MCP servers, Codex plugins,
   Claude Code plugins, extensions/hooks, or arbitrary configuration folders?
7. **Skill behavior:** Should skills be copied, symlinked, or selectable per
   target application? How should locally edited skills conflict with remote
   changes?
8. **WebDAV conflicts:** Should conflicts use last-write-wins, preserve both
   versions for manual resolution, or merge records by identifier?
9. **Packaging:** May the installer bundle the OpenCodex runtime/sidecar, or must
   it detect and control a separately installed OpenCodex?
10. **Upstream strategy:** Should Context Sync maintain forks pinned to audited
    commits, use Git submodules, consume released packages, or contribute the
    integration hooks upstream?
11. **Distribution:** Will releases be free/open source only, commercially sold,
    or distributed through app stores? Signing, notarization, and updater
    requirements depend on this choice.
12. **Compatibility and migration:** Must existing OpenCodex and CC Switch users
    import their current configuration losslessly and be able to roll back?

## Draft acceptance criteria

- A user can switch provider/model, and the next eligible Codex request reaches
  that route without restarting Codex.
- A failed route change is atomic: the prior working route remains active.
- WebDAV synchronization is deterministic, resumable, and never uploads secrets
  unless the user explicitly enables encrypted secret sync.
- Imported OpenCodex and CC Switch data is backed up before migration.
- Installers and smoke tests exist for Windows, macOS, and Linux.
- All reused or modified upstream files preserve required copyright and license
  notices, and distributed builds include third-party notices.

## Licensing note

At the time of this initial review, both upstream repositories declare the MIT
License. MIT generally permits use, modification, merging, redistribution, and
commercial use, provided the copyright and permission notices remain in copies
or substantial portions. Context Sync should therefore keep upstream notices,
add a `THIRD_PARTY_NOTICES` inventory, and audit transitive dependencies and
bundled assets separately before distribution. This is an engineering summary,
not legal advice.
