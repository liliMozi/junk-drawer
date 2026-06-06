---
name: oh-plugin-creator-coding-agent
description: Use when a coding agent is asked to design, implement, debug, package, or publish Hana/OH application plugins in a codebase, including SDK-based runtime tools, iframe UI, EventBus integrations, Session/Agent APIs, model/media APIs, local installation, and OH-Plugins marketplace entries.
metadata:
  default-enabled: false
---

# OH Plugin Creator Coding Agent

Use this for professional Hana/OH plugin development by a Coding Agent. For user-facing hand-holding inside Hana, use `hana-plugin-creator`; for Codex `.codex-plugin` bundles, use the system `plugin-creator`.

## Core Rule

Treat a plugin as a separate product boundary. Keep plugin state in plugin data, expose behavior through manifest contributions and SDK APIs, and avoid importing Hana renderer or host internals.

## First Checks

1. Find the Hana repo root. Prefer a workspace containing `PLUGINS.md`, `PLUGIN_SDK.md`, and `packages/plugin-runtime`.
2. Read, in this order:
   - `.docs/PLUGIN-DEVELOPMENT.md`
   - `PLUGINS.md`
   - `PLUGIN_SDK.md`
   - `packages/plugin-sdk/README.md` and `packages/plugin-components/README.md` for iframe UI
   - `packages/plugin-runtime/README.md` for tools, lifecycle, EventBus, tasks, or file output
3. Inspect existing examples before inventing structure: `examples/plugins/sdk-showcase/` and nearby plugins.
4. If the request involves marketplace publication, inspect the `OH-Plugins` repo and run its validation before delivery.

## Pick The Shape

| Shape | Use When | Default Target | Permission |
| --- | --- | --- | --- |
| Tool-only | Agent calls a function, no UI | `tools/*.js` | `restricted` |
| Runtime | lifecycle, EventBus, task, schedule, dynamic tool | `index.js` | `full-access` |
| UI | page, widget, iframe card | `routes/` + built UI assets | `full-access` |
| Parallel session app | plugin-owned Agent/Session, custom chat UI, RAG/world context, Tavern-style flow | runtime helpers + iframe UI | `full-access` |
| Official source | plugin should live in official catalog source | `OH-Plugins/official-plugins/<id>` | depends |
| Marketplace entry | plugin should be discoverable | `OH-Plugins/plugins/<id>.json` | metadata |

If the user's desired capability depends on native renderer components, code sandboxing, fine-grained permission prompts, or remote release auto-install, say that boundary explicitly and choose the nearest supported iframe/runtime shape.

## Host Capability Surface

Prefer typed helpers from `@hana/plugin-runtime`; use raw EventBus only when a helper does not exist.

- Session: `createSession`, `getSession`, `listSessions`, `updateSession`, `sendSessionMessage`, `subscribeSessionEvents`, plus bus `session:abort` and `session:history`. `createSession()` is detached and does not switch the main Hana UI focus.
- Agent: `listAgents`, `getAgentProfile`, `createAgent`, and `updateAgent`. Plugin-only agents should use `visibility: "plugin_private"` and an explicit `ownerPluginId`.
- Per-turn context: `sendSessionMessage(..., { context: { system, beforeUser, afterUser } })` injects hidden context only into the current provider request. Use it for plugin RAG, world lore, mood, character state, or routing hints. Do not mutate visible user text or write JSONL directly.
- Model: `sampleText()` runs non-streaming utility-model calls for query rewriting, summaries, classifiers, routing, and other plugin-side work.
- Media: `listMediaProviders()`, `resolveMediaModel()`, and `generateImage()` use the host provider registry and media task pipeline. Generated files are delivered as `SessionFile` resources.
- Manifest declarations: use ordinary `capabilities` for needs such as `session`, `agent`, `model.sample`, and `media.generate`; use `sensitiveCapabilities` only to record future user-granted permission intent.

## Implementation Workflow

1. Decide ownership:
   - Example/template: `project-hana/examples/plugins/<id>`.
   - Built-in app plugin: `project-hana/plugins/<id>`.
   - Local user-installed plugin: `${HANA_HOME}/plugins/<id>` or the path from `/api/plugins/settings`.
   - Official marketplace source: `OH-Plugins/official-plugins/<id>`.
2. Scaffold with the bundled script when possible. Resolve `scripts/create_hana_plugin.py` relative to this skill directory if the skill is installed somewhere other than the repo-local `.agents/skills` path.

```bash
python3 .agents/skills/oh-plugin-creator-coding-agent/scripts/create_hana_plugin.py "Plugin Name" --path examples/plugins --kind full --audience developer --template professional-react --sdk-mode workspace
```

Use `--kind tool` for restricted tool-only plugins and `--kind ui` for page/widget-only plugins. Prefer `--sdk-mode workspace` inside `project-hana`; use `--sdk-mode bundled` for standalone plugin directories or user-installed plugins that should carry SDK tarballs from this skill.

3. Edit only inside the plugin's ownership boundary unless the host platform needs a deliberate SDK/API change.
4. Keep `manifest.json` explicit: `id`, `name`, `version`, `description`, `minAppVersion`, `trust`, `capabilities`, `sensitiveCapabilities` when relevant, `contributes`, and `ui.hostCapabilities` when iframe host calls are used.
5. For files visible to users, use `stageFile({ sessionPath, filePath, label })` plus `createMediaDetails()`. Never hand-build local `MEDIA:`, `file://`, or local `mediaUrls` output.
6. For iframe UI, use `@hana/plugin-sdk`, `@hana/plugin-components`, `HanaThemeProvider mode="inherit"`, and route shells that safely pass `hana-theme` / `hana-css`.
7. For EventBus handlers, use `defineBusHandler()`, `requestBus()`, and `HANA_BUS_SKIP`. Pass `sessionPath` explicitly for session operations.
8. For plugin-owned chat experiences, create detached plugin-private sessions and agents instead of reusing UI focus pointers. Query by `ownerPluginId` to recover plugin resources after restart.
9. For background work, use `task:*` and `deferred:*` contracts. Re-register handlers in `onload()` for restart recovery.

## Bundled Resources

- `scripts/create_hana_plugin.py`: deterministic plugin scaffold generator.
- `assets/sdk/*.tgz`: SDK package snapshots for standalone scaffolds.
- `agents/openai.yaml`: Codex UI metadata. Keep it aligned with this `SKILL.md` when renaming or changing the skill's purpose.

## Marketplace Rules

- `OH-Plugins` is the catalog repo; `project-hana` is the host/SDK repo.
- Every `plugins/<id>.json` entry needs identity, version, repository, compatibility, trust, permissions, contributions, distribution, and one README source.
- Use inline `readme` or HTTPS `readmeUrl` for URL marketplaces.
- Use `readmePath` only for local file marketplaces.
- `distribution.kind = "source"` is installable only when Hana reads a local marketplace file and the path resolves on disk.
- `distribution.kind = "release"` is the future remote package contract; one-click remote download, sha256 verification, and permission review must exist in the host before calling it install-ready.
- Before pushing `OH-Plugins`, run privacy-push and wait for explicit user confirmation.

## Verification

Run the narrowest checks that prove the touched boundary:

- Scaffold smoke: create a temp plugin with the same scaffold path and delete it after.
- Tool plugin: inspect exports and install locally, or run the focused plugin tests if present.
- UI plugin: run plugin build, then verify assets referenced by routes exist.
- SDK package change: `npm run build:packages`, then focused type/tests.
- Host behavior change: `npm run typecheck` and relevant tests.
- Session/Agent capability change: run focused plugin runtime, EventBus capability, hub capability, session coordinator, and turn-context tests.
- Marketplace change: from `OH-Plugins`, run `npm run check`.

Do not claim a plugin is ready until install/run instructions and verification evidence are clear.

## Delivery

Report:

- plugin path and shape
- permission level and why
- install/test commands
- marketplace entry path if added
- any remaining host-platform gap

If the plugin idea is valuable, describe the concrete workflow it improves rather than giving generic praise.
