---
name: develop-omp-extention
description: >-
  Develop, modify, debug, test, package, and release extensions for Oh My Pi (OMP).
  Use whenever the user asks for an OMP extension or plugin, ExtensionAPI tools,
  commands, lifecycle hooks, native terminal UI, extension compatibility after
  an OMP update, local plugin linking, GitHub distribution, or porting an OMP
  extension to pi. Covers both greenfield work and existing repositories.
  Applies regardless of which AI agent harness is doing the development.
  Do not confuse OMP with Oh My Posh, Oh My Zsh, or a generic browser extension.
compatibility: >-
  Harness-independent instructions. Requires repository/file access and a shell;
  official OMP source/docs access and the target OMP runtime are needed to verify
  integration. Interactive terminal access is needed to verify TUI behavior.
---

# Develop OMP extensions

Build the requested behavior against an identified OMP version, using native host facilities. Prove it loads and works in that host; compilation alone is not integration proof.

This skill does not require the developing agent to run inside OMP. Use the file, search, shell, web, process, and terminal tools available in your harness. References below are ordinary files relative to this directory, not harness-specific URIs. Do not assume access to the user's current OMP session, credentials, internal tools, or process state.

## Read the relevant references

| Work | Read |
|---|---|
| Every development task | [API and source research](references/api-and-research.md) |
| Any terminal UI, picker, widget, renderer, or editor integration | [Native TUI development](references/tui.md) |
| Creating a package, dependency changes, installation, updates, releases | [Packaging and release](references/packaging-and-release.md) |
| Any implementation or bug fix | [Verification and troubleshooting](references/verification.md) |
| User explicitly requests pi support or a port | [Porting to pi](references/pi-porting.md) |
| Need current official entry points or examples | [Source map](references/sources.md) |

The references record an inspected snapshot, not a stable SDK guarantee. Recheck version-sensitive claims for the actual target. Do not read every upstream file or copy the entire API catalog into a small project.

## 1. Establish the target and acceptance criteria

Before editing:

1. Identify the extension repository, requested behavior, supported OS/runtime modes, and intended installation route. Read existing project instructions, manifest, lockfile, affected source, tests, and release conventions.
2. Run `omp --version` and inspect `omp --help` and relevant plugin subcommand help. Distinguish the OMP executable version from the extension's development dependency versions and the latest upstream branch.
3. Locate target-version OMP source/types locally or fetch the corresponding upstream tag/commit. Record the version/ref inspected. If the exact sources cannot be established, state that limit; do not label latest-main observations as installed-runtime facts.
4. Write observable acceptance criteria: command/tool behavior, output, cancellation, error behavior, persistence, UI modes, compatibility range, and install/reload success as applicable.
5. For an existing extension, identify the current behavior and smallest affected boundary. Preserve unrelated user changes, interfaces, settings, release practices, and data.

Resolve environment facts from files and commands first. Ask only about product choices with materially different consequences: a pi port versus dual support, destructive migration, public publication, or broad compatibility requirements that are not inferable.

## 2. Research before designing

Read all three for every host feature you will use:

- The official feature guide/specification at the target revision.
- The actual exported type and package export map.
- The loader, runner, or mode-specific implementation that makes it work.

Then inspect one relevant official example and any upstream regression test covering the uncertain contract. An exported method can be a no-op in a particular mode; an example can still use an obsolete signature. Follow discrepancies to implementation and runtime behavior, not whichever source is easiest to copy.

Create a small compatibility note in your working plan:

| Needed capability | Actual symbol/import | Inspected version/ref | Supported mode | Risk/check |
|---|---|---|---|---|
| Example: interactive selection | `ctx.ui.select` | exact target | TUI; bridge-dependent elsewhere | cancellation result and labels |

Prefer high-level `ExtensionAPI`/context features over exported deep internals. Exported wildcard paths are not a promise of semantic stability. If a required capability is absent, choose an explicit higher minimum version, or a documented supported alternative that preserves the requested behavior. Do not hide an unsupported host behind `any`, swallowed import errors, or a fake fallback.

## 3. Choose the smallest architecture

For new work, use a default-exported extension factory. Register commands, tools, events, flags, shortcuts, providers, and renderers there; perform session-dependent actions from handlers after initialization.

Choose the surface by responsibility:

- Slash command: user-triggered action or interactive workflow.
- Tool: model-callable operation with a validated schema and structured result.
- Event handler: lifecycle policy, context transformation, or observation.
- Native dialog/status/widget: small UI addition.
- Custom TUI: only when native dialogs or components cannot express the interaction.
- Provider registration: custom model/API integration through host facilities.
- SDK application: embedding a new agent application, not the default solution for an extension.
- Legacy hook/custom-tool: maintain when already required; prefer unified extensions for new work.

Keep domain logic separate from host side effects when that makes behavior testable. Do not create a framework, service container, adapter hierarchy, or build pipeline for one command. Add files at real responsibility boundaries, not to satisfy a template.

For existing work, reuse its conventions unless they are the cause of the problem. If replacing a contract, update all consumers and remove code made obsolete by the change. Do not opportunistically redesign unrelated UI or storage.

## 4. Establish a real development loop early

For a new installable package, use a minimal ESM TypeScript project with an explicit `omp.extensions` entry. See the packaging reference for dependency and manifest rules.

Before implementing a large feature:

1. Install dependencies using the repository's package manager and supported runtime.
2. Typecheck a minimal registration against the target host package.
3. Load it once through `omp --extension ./src/index.ts` or the package directory.
4. Exercise a real command or tool to verify runtime import resolution.
5. Switch to `omp plugin link .` when persistent local installation is needed.

Use one loading route per development session. Do not leave a loose copy, explicit `--extension`, linked plugin, and published copy all active and then debug duplicate registrations.

A working baseline should perform a real harmless operation. Do not mistake registration-only scaffolding for the requested finished feature.

## 5. Implement with host ownership intact

- Use current host model, tool, theme, editor, keybinding, settings, and session facilities instead of reproducing their state locally.
- Read mutable host state at the time of action. Do not keep a stale model, working directory, active-tools list, or session branch captured during load.
- Validate all inputs before mutation. Preserve user data, use the correct scope/profile, and await persistence before reporting success.
- Respect cancellation and failures. Do not report an aborted or partially written operation as successful.
- Avoid process-wide side effects, raw terminal writes, unhandled background rejections, and leaked listeners/timers.
- Never alter auth files, user model settings, the user's active session, or installation scope just to make a smoke test easier. Isolate test state.
- Do not claim a substring-based shell guard is a security sandbox. Extensions execute in the host process with its privileges.

For TUI work, read the TUI reference before writing rendering/input code. Prefer native surfaces, then native components, then a small custom component.

Use native theme icons where helpful, including Nerd Icons when the user's resolved preset enables them. Respect Unicode/ASCII alternatives, symbol overrides, and live theme changes; never hard-code Nerd glyphs for every user or invent a `nerdIcons` boolean. The TUI reference defines the source-verified `symbolPreset` workflow and checks for enabled/disabled modes.

## 6. Verify the changed behavior

Use the verification reference to select checks, not to generate a bloated test suite.

- Run the project's typecheck and relevant automated tests; before delivery run its required full checks.
- Bug fix: use the user's reported failure as the starting fact; create a focused reproduction where feasible, then prove it no longer fails.
- New behavior: exercise the real operation, including a meaningful error or cancellation path.
- TUI: launch actual OMP in a controlled terminal and check rendering, navigation, action result, close/cancel, resize, and restoration of the editor.
- Installation: inspect `omp plugin list --json` and runtime diagnostics, reload or restart, then invoke the extension.
- Compatibility promise: test the minimum and newest supported versions; compile-only compatibility is not sufficient for mode-sensitive APIs.

If the available harness cannot drive an interactive terminal, complete typechecks, behavior tests, loading checks, and any noninteractive integration checks. Explicitly mark TUI smoke testing as unverified and provide exact manual steps. Do not manufacture a screenshot or claim a live interaction occurred.

## 7. Version, install, and release deliberately

Version the extension independently from OMP. Decide a bump from the extension's user-facing compatibility contract, not the size of the diff. Ordinary unshipped edits do not require repeated version increments. Follow existing release automation and keep all authoritative version fields and relevant lockfile metadata consistent.

Use the packaging reference to distinguish:

- Linked checkout updates: update checkout/dependencies, refresh the link when needed, reload/restart.
- Direct GitHub installs: install/reinstall the explicit repository source or verified ref.
- Marketplace installs: use that marketplace's upgrade mechanism.
- npm distribution: only when a package is actually published there.

Publishing, pushing tags, upgrading the user's host runtime, or replacing an installed source requires the user's authorization. Preparing and verifying release files does not imply permission to publish.

Never delete extension data to complete an upgrade. Package code version, host compatibility range, and persistent-data schema version are three different things.

## 8. Deliver evidence

Report briefly:

1. Implemented behavior and relevant files.
2. OMP version(s), upstream ref(s), and runtime modes actually checked.
3. Checks run and observed outcomes; separate automated checks from live TUI proof.
4. Version decision and local/GitHub installation or update instructions as applicable.
5. Remaining limitations, especially unavailable interactive verification or untested host versions.

A task is not complete because types pass. The intended installed extension must load and the requested behavior must work, or the missing verification prerequisite must be stated precisely.
