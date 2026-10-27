# Verification and troubleshooting

## Build a small, meaningful test matrix

Choose rows that exercise actual risks rather than multiplying every version, mode, OS, and font combination mechanically.

| Dimension | Minimum useful coverage |
|---|---|
| Host | Installed target; minimum/newest if claiming a range |
| Loading | Explicit entrypoint early; installed package before release |
| Behavior | Happy path; meaningful error; cancel when supported |
| State | Persist/reload; branch/switch if session-scoped; write conflict if multi-session writable |
| UI | Actual terminal; narrow/short resize; long/empty input; editor restoration |
| Icons | Nerd, Unicode, ASCII; change/reopen; no stale width cache |
| Modes | Each advertised mode; defined behavior without custom TUI |
| Lifecycle | Restart/reload without duplicate registrations or leaked work |
| Distribution | Linked checkout and clean published package/tag when releasing |

## Automated checks

Use existing project scripts. Common Bun projects expose:

```sh
bun run typecheck
bun test
```

These are examples, not universal script names. Read `package.json`. If a new project has no scripts, add appropriate typecheck/test scripts rather than invoking nonexistent ones. Check required upstream/runtime native dependencies if imports fail before a test executes.

Tests should defend observable contracts:

- Invalid input cannot partially mutate settings/data.
- Cancel/abort prevents further side effects and closes the owned UI/resources.
- Storage errors remain errors and do not reset user data.
- Concurrent writers cannot silently overwrite each other.
- State reconstruction follows the active branch rather than all history.
- Required UI content fits the supplied dimensions across symbol presets.
- A native schema/tool result works with the actual loader and provider contract where relevant.

Avoid tests that merely search source text for method names, echo mocked inputs, pin incidental wording, or assert an object is nonempty. Use a throwaway smoke script for a straightforward new API path; keep a regression test when a plausible future bug would break its assertion.

Mocks can isolate domain behavior, but they cannot prove the host exports an import, mounts a component, emits an event, enforces approval, or reloads a factory. At least one integration check must use the real host for the changed boundary.

## Safe runtime environment

Use a temporary project and separate test profile/agent directory where supported. Inspect the target's profile/environment behavior: changing the agent directory alone may not isolate user-level plugin roots. Prefer explicit extension loading with ambient discovery disabled for early smoke tests, then separately test the real installation route.

Do not edit the user's real settings, sessions, auth, or extension data as test fixtures. Do not make paid model requests or remote mutations unnecessary to verify a local command. When a real provider is required, use an authorized credential and bound the operation.

A harness with terminal process control can launch OMP in a PTY, send keys, inspect output/screenshots, and resize using its available tools. A harness without a PTY should not pretend a pipe or RPC request proves terminal rendering. If the target offers a supported terminal debug facility, inspect its docs before using it and keep it isolated from other sessions.

## Live TUI acceptance procedure

1. Start the target OMP with the intended extension loading route.
2. Confirm there are no load errors and the expected command/tool is registered.
3. Open the extension's actual command, not a standalone fake renderer.
4. Navigate using documented keys, enter/edit/select data, and commit a real harmless action.
5. Observe the result and persisted state where applicable.
6. Reopen and cancel; ensure no unintended mutation and editor text/focus restoration.
7. Trigger a controlled real error, such as an invalid extension input or unwritable temporary data path; ensure the UI reports it without closing the host.
8. Resize to a narrow/short terminal; inspect scrolling, focus, footer, and confirmation controls.
9. Check Nerd Font enabled and disabled via the native symbol preset, including ASCII. Verify the icon/font observation separately from computed width assertions.
10. Restart or use a verified runtime reload. Repeat a distinguishing changed action and check for duplicate callbacks or leaked background work.

For non-UI extensions, replace terminal navigation with actual command/tool/event execution. Print-mode or RPC checks are appropriate when those are the requested surfaces; do not require a cosmetic TUI test for a feature that never renders one.

## Installation acceptance

`omp plugin list --json` should show the expected enabled package and actual manifest entrypoints/source available in that version's output. Resolve the symlink/source if the JSON omits it. Check that project-scope shadowing or a duplicate loose file is not selecting another copy.

Load diagnostics and an actual command/tool invocation matter more than a list entry. An installed package can have a disabled feature, bad import, missing asset, reserved command name, or entrypoint that registers nothing.

Do not run `plugin doctor --fix`, uninstall unrelated plugins, or replace user config automatically to make the test pass. Diagnose first; repair only the intended extension boundary.

## Troubleshooting map

| Symptom | First checks |
|---|---|
| Command absent | Real entrypoint/default factory; manifest; enabled feature; built-in name conflict; logs |
| Old behavior after edit | Which source loaded; duplicate routes; module cache; actual reload scope; restart |
| `ExtensionRuntimeNotInitializedError` | Runtime action during factory/module load |
| Import missing | Target host version; actual package export; deep path; peer/dev mismatch; loader remapping |
| Works linked, fails published | Missing shipped file/asset; runtime dep in devDependencies; cwd-relative path; uncommitted source |
| UI promise never resolves | No `done`; unsupported mode; async failure; abort race |
| TUI corruption or scrolling jitter | stdout writes; cell width; tabs/controls; excessive rows; stale caches |
| Nerd glyph boxes | User preset and actual terminal font; hard-coded private-use glyph; stale theme |
| Wrong fallback icons | Raw config instead of resolved theme; status-line preset confused with symbol preset |
| Tool blocked unexpectedly | Throwing `tool_call` handler; wrong input assumptions; approval tier |
| Tool receives wrong context/signal | Stale positional `execute` example; target type/wrapper mismatch |
| Session crashes later | Raw timer/detached promise rejection; missing resource cleanup |
| Branch shows another branch's state | Whole-history reconstruction or stale mutable closure |
| Reinstall loses settings | Data stored under package code directory; destructive migration; hard-coded profile |

Use `pi.logger` and target runtime logs. The inspected default logs root is `~/.omp/logs`, but active state-root/profile behavior can differ; locate it through the target rather than hard-coding a user's path. Redact credentials and sensitive tool arguments from diagnostic output.

## Completion report

Separate four kinds of evidence:

- Static: exact typecheck command and outcome.
- Behavior: tests/smoke operations and observed state transitions.
- Runtime: host version, loading route, real command/tool invocation.
- Visual: terminal/mode, interaction, dimensions/presets actually observed.

If a prerequisite is unavailable, finish reachable checks and identify the precise missing item, such as “no interactive terminal control; Nerd Font rendering and editor restoration not observed.” Provide exact manual steps. Do not mark the entire integration verified because a mocked component test passed.
