# Official source map and freshness record

## Inspected snapshot

Prepared on 2026-09-15. OMP upstream tree inspected at commit `07a10ebf7263bf419f53c84a5d516037fae61911`; `packages/coding-agent/package.json` reports `18.2.0`. The local CLI independently reported `omp/18.2.0`. pi's official main package inspected through the historical repository URL reports `@earendil-works/pi-coding-agent` version `0.85.1`; its package repository field points to `earendil-works/pi`. pi main links below are discovery references, not an immutable tested pi release.

Re-resolve the installed/target version before using these instructions. Prefer its exact tag/commit. Do not infer a tag naming scheme; inspect tags/releases and package metadata. A same-version local build may differ from the current upstream source.

The links below lead to official project material. Examples show patterns, not guaranteed copy-paste compatibility. Read declarations and runtime implementations at the same revision.

## OMP guides and specifications

- [Repository](https://github.com/can1357/oh-my-pi)
- [Extension runtime guide](https://github.com/can1357/oh-my-pi/blob/07a10ebf7263bf419f53c84a5d516037fae61911/docs/extensions.md)
- [Official extension authoring skill](https://github.com/can1357/oh-my-pi/blob/07a10ebf7263bf419f53c84a5d516037fae61911/docs/skills/authoring-extensions.md)
- [Extension discovery/loading](https://github.com/can1357/oh-my-pi/blob/07a10ebf7263bf419f53c84a5d516037fae61911/docs/extension-loading.md)
- [TUI integration contract](https://github.com/can1357/oh-my-pi/blob/07a10ebf7263bf419f53c84a5d516037fae61911/docs/tui.md)
- [TUI core renderer](https://github.com/can1357/oh-my-pi/blob/07a10ebf7263bf419f53c84a5d516037fae61911/docs/tui-core-renderer.md)
- [TUI runtime internals](https://github.com/can1357/oh-my-pi/blob/07a10ebf7263bf419f53c84a5d516037fae61911/docs/tui-runtime-internals.md)
- [Plugin manager/installer plumbing](https://github.com/can1357/oh-my-pi/blob/07a10ebf7263bf419f53c84a5d516037fae61911/docs/plugin-manager-installer-plumbing.md)
- [Marketplace guide](https://github.com/can1357/oh-my-pi/blob/07a10ebf7263bf419f53c84a5d516037fae61911/docs/marketplace.md)
- [SDK guide: use for embedding, not automatically for extensions](https://github.com/can1357/oh-my-pi/blob/07a10ebf7263bf419f53c84a5d516037fae61911/docs/sdk.md)
- [Release history](https://github.com/can1357/oh-my-pi/releases)
- [Coding-agent changelog](https://github.com/can1357/oh-my-pi/blob/07a10ebf7263bf419f53c84a5d516037fae61911/packages/coding-agent/CHANGELOG.md)

## OMP implementation map

Base: [packages/coding-agent/src](https://github.com/can1357/oh-my-pi/tree/07a10ebf7263bf419f53c84a5d516037fae61911/packages/coding-agent/src).

| Question | Source path relative to that base |
|---|---|
| Factory/API/context/event/tool types | `extensibility/extensions/types.ts` |
| Discovery and initialization | `extensibility/extensions/loader.ts`, `runner.ts`, `index.ts` |
| Tool interception and approval | `extensibility/extensions/wrapper.ts` |
| Shared event payloads | `extensibility/shared-events.ts` |
| Host/legacy import mapping | `extensibility/legacy-pi-compat.ts` and sibling shim modules |
| Interactive dialog mounting, no-ops, lifecycle | `modes/controllers/extension-ui-controller.ts` |
| CLI command routing/reload | `modes/controllers/command-controller.ts`, `session/agent-session.ts` |
| Manifest and features | `extensibility/plugins/types.ts`, `manager.ts` |
| Git source/ref syntax | `extensibility/plugins/git-url.ts` |
| Local/remote target classification | `cli/classify-install-target.ts`, `commands/install.ts` |
| Settings keys, scope and semantics | `config/settings-schema.ts`, `config/settings.ts` |
| Resolved theme and live updates | `modes/theme/theme.ts`, `theme-class.ts` |
| Nerd/Unicode/ASCII symbols | `modes/theme/symbols.ts` |
| Theme adapters for native TUI widgets | `modes/theme/tui-adapters.ts` |
| Read-only session traversal and persistence | `session/session-manager.ts`, `session/messages.ts` |

Follow the target tree if paths have moved. Read relevant tests under the coding-agent package rather than relying on this map for a method's semantics.

Other important package paths:

- [Coding-agent package exports and engines](https://github.com/can1357/oh-my-pi/blob/07a10ebf7263bf419f53c84a5d516037fae61911/packages/coding-agent/package.json)
- [Native TUI exports](https://github.com/can1357/oh-my-pi/blob/07a10ebf7263bf419f53c84a5d516037fae61911/packages/tui/src/index.ts)
- [Native TUI source](https://github.com/can1357/oh-my-pi/tree/07a10ebf7263bf419f53c84a5d516037fae61911/packages/tui/src): `tui.ts`, `utils.ts`, `keys.ts`, `keybindings.ts`, `components/`

## Official examples to inspect selectively

[OMP extension examples](https://github.com/can1357/oh-my-pi/tree/07a10ebf7263bf419f53c84a5d516037fae61911/packages/coding-agent/examples/extensions):

- `hello.ts`: minimal factory/tool/schema pattern. At this snapshot its unused execute arguments carry stale names/order; use canonical types instead.
- `question.ts`: native selection rather than a hand-built picker.
- `tools.ts`: native `SettingsList` composition and session branch reconstruction. Recheck command-name collisions and lifecycle coverage instead of copying the entire file.
- `status-line.ts`: native status output and theming.
- `todo.ts`: structured results and custom rendering/state patterns.
- `with-deps/`: extension-local third-party dependencies.
- `confirm-destructive.ts`: pre-action confirmation pattern, not a security sandbox.
- `custom-compaction.ts`: specialized lifecycle integration when actually requested.

The [official hello-extension skill example](https://github.com/can1357/oh-my-pi/tree/07a10ebf7263bf419f53c84a5d516037fae61911/docs/skills/examples/hello-extension) is another entrypoint example. Check current existence and imports at the selected target ref.

Do not copy unrelated auto-commit, shell-guard, or game behavior into a normal extension. Reuse the narrow native integration pattern that solves the task.

## pi official sources for an explicit port

- [Current repository](https://github.com/earendil-works/pi)
- [Historical repository entry point](https://github.com/badlogic/pi-mono)
- [Extension guide](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/extensions.md)
- [Package/install guide](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/packages.md)
- [TUI guide](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/tui.md)
- [Package metadata/exports/engines](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/package.json)
- [Official extension examples](https://github.com/earendil-works/pi/tree/main/packages/coding-agent/examples/extensions)
- [Extension types](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/extensions/types.ts)

Resolve these to the actual pi release/ref before porting. Old pi examples, new pi docs, and OMP compatibility shims are three different sources of contracts.

## Maintenance trigger

Revisit this skill's snapshot-specific facts when:

- A target upgrade changes exports, schemas, tool signatures, events, UI behavior, icon settings, or loader mapping.
- A documented command behaves differently from CLI help/implementation.
- A native method compiles but is unsupported/no-op in the target mode.
- A pi namespace, package manager, or runtime requirement changes.

Update the date/ref and only the affected guidance. Preserve the source-first workflow; do not accumulate fallback layers for every historical API.
