# Adapting OMP extensions to pi

Only add pi support when requested. OMP and pi share ancestry, not a stable interchangeable SDK. OMP loading a legacy pi extension does not prove pi can load an OMP extension.

## Identify both targets first

Run the available target CLIs' version/help commands and inspect their installed manifests, types, loaders, and UI implementations independently. Record an OMP version/ref and a pi version/ref. Do not compare latest pi documentation with an old installed pi package and call the result tested.

At the 2026-09-15 inspection:

- OMP uses `@oh-my-pi/*`; inspected coding-agent version was `18.2.0`.
- The historical official repository `badlogic/pi-mono` leads to the current `earendil-works/pi` project. Its inspected main coding-agent package identifies itself as `@earendil-works/pi-coding-agent`, version `0.85.1`.
- Older pi versions and examples use `@mariozechner/*`. Package names are version-sensitive: inspect the target manifest rather than mechanically replacing every import with a name memorized from old tutorials.
- OMP's compatibility loader remaps several legacy/canonical namespaces into host copies/shims. That is an OMP implementation feature, not a portable dependency contract.

## Choose the scope

**Single-target port:** Prefer when the user wants a pi version instead of OMP. Replace the integration layer and manifests/imports, remove obsolete OMP-only paths, and test pi. Do not keep unused compatibility scaffolding.

**Dual-target support:** Use only when both hosts are required. Share domain logic and data formats where semantics match. Keep small typed per-host entrypoints/adapters for registration, schema building, UI, model/settings access, and persistence only where they differ. Separate package wrappers can avoid installing both hosts' peers; a dual-manifest package may be appropriate if both loaders and dependency installation have been verified.

Do not create one large dynamic `try import OMP; catch import pi` module, broad `any` casts, or no-op fallbacks. They conceal which behavior is actually supported. Required unavailable functionality is a product decision: implement a real equivalent, document a deliberate target-specific limitation agreed with the user, or require the needed host version.

## Compare actual contracts

| Area | What to check when porting |
|---|---|
| Package imports | Current pi namespace and export map; OMP wildcard deep imports generally cannot be copied |
| Schema builders | OMP injected omptype-backed `pi.zod`/`arktype`/legacy `typebox` versus target pi's TypeBox/other supported builders |
| Tool registration | Exact `ToolDefinition`, helpers such as `defineTool` if present, result shape, approval/discovery fields |
| Execution | Positional order, cancellation, partial update callback, context type |
| Events | Event names, payloads, return values, middleware ordering, whether emitted in each mode |
| Session control | Command context versus ordinary context, branch reconstruction, reload semantics |
| UI | Native components/constructors, `custom` options, overlays, return contracts, mode behavior |
| Theme/icons | OMP semantic symbols and `symbolPreset` must not be assumed to exist in pi |
| Model/settings | OMP roles, `ctx.models`, live settings exposure and model browser versus pi equivalents |
| Background work | Managed timer availability, shutdown/reload disposal, errors |
| Native/runtime APIs | Bun-specific operations, native modules, Node compatibility and supported engines |
| Installation | `omp` manifest/plugin manager versus `pi` manifest/package manager |

At the inspected snapshots, both canonical tool execute contracts use `(toolCallId, params, signal, onUpdate, ctx)`. Similar signatures do not guarantee identical context fields or result semantics. Reverify the actual target versions.

Do not assume OMP-specific features exist on pi: examples include injected `pi.pi` exports, `ctx.models`, managed timers, same-tool `ctx.invokeTool`, rich ask-dialog options, OMP approval/discovery fields, model roles, composer shapes, file-write fallbacks, and particular MCP/Python lifecycle events. Conversely, pi may implement UI facilities that are no-ops in an OMP mode.

The inspected pi guide uses `Type` from `typebox` for tool schemas; current examples may also expose helpers through pi-ai. Inspect current exports and serialized schemas instead of textual conversion from `z.object` to an assumed API. Test optional/default/enum behavior after a schema port.

## TUI porting

Reuse pi's own current `ctx.ui` methods and `@earendil-works/pi-tui` components (or the namespace of the target version). Do not bundle OMP's TUI into pi to preserve a screenshot.

Check:

- `custom` callback order and option types, overlay lifecycle and disposal.
- Keyboard actions and user remapping behavior.
- ANSI-aware cell width and immutable render rows.
- Widget/header/footer/editor support in the actual runtime mode.
- Standard dialogs versus unsupported custom components in RPC/headless modes.
- Theme symbols and Nerd Font preference. If pi has no equivalent global icon setting, use native pi rendering or plain text by default. Introduce an explicit extension-owned icon preference only when needed and agreed; never read OMP's config as pi's setting.

A port is not visually verified because an OMP screenshot looked correct. Exercise the pi command in pi.

## Package and local workflow

A pi package declares its entrypoint under `pi` and can include the `pi-package` keyword:

```json
{
  "name": "my-pi-extension",
  "version": "0.1.0",
  "type": "module",
  "keywords": ["pi-package"],
  "pi": { "extensions": ["./src/index.ts"] }
}
```

Add actual dependency and compatibility declarations. The inspected pi package guide recommends host packages as peer dependencies with `"*"` and no bundling. That convention is not a promise that every pi version works: document and test the actual support matrix, install exact dev versions, and use the project's validated compatibility policy.

Inspected pi local workflow:

```sh
npm install
pi -e ./src/index.ts
pi install ./path-to-package
pi list
```

Choose the real project's package manager. Local pi paths are registered without copying; this is not OMP's `plugin link`. Native pi discovery roots are normally `~/.pi/agent/extensions/` and `.pi/extensions/`, with project trust requirements. Follow the target's profile/config resolution instead of hard-coding defaults.

For a published GitHub package/tag:

```sh
pi install git:github.com/OWNER/REPOSITORY@v0.2.0
```

For npm only if actually published:

```sh
pi install npm:@SCOPE/PACKAGE@0.2.0
```

Do not reuse OMP's `github:owner/repo#ref` syntax or `plugin upgrade` command in pi instructions.

At the inspected pi revision:

- `pi update --extensions` updates packages/reconciles configured pinned Git refs without updating pi itself.
- Bare `pi update` updates the pi CLI itself; do not use it as a harmless package-only command.
- Changing a pinned Git version requires installing the new ref, such as `pi install git:github.com/OWNER/REPOSITORY@v0.2.1`.
- `pi install ... -l` targets project settings; verify target help before applying.
- `/reload` is documented for auto-discovered extension locations. Recheck explicit-path/package reload behavior and restart if uncertain.

Runtime third-party dependencies must survive production installs; do not depend on dev-only packages. Git-managed clones can be reset/cleaned during reconciliation, so do not use an installed clone as a development worktree.

## Port acceptance checklist

1. Compile against the actual pi packages without OMP-only imports leaking into the pi graph.
2. Exercise domain behavior and schema differences, including cancellation and errors.
3. Load through pi's real loader, not only Node/Bun directly.
4. Verify each advertised UI/runtime mode and native icon behavior independently.
5. Confirm persistence, branch/switch/reload behavior, and migration policy.
6. Install the published artifact/tag through pi's package manager.
7. For dual support, repeat the same required contracts in OMP; passing one host is not evidence for the other.
8. Document target-specific differences honestly. Do not advertise full parity when a required operation is missing.

Official links are collected in [sources.md](sources.md).
