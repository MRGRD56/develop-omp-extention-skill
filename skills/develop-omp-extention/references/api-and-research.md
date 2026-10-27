# API research and implementation contracts

## Evidence and freshness

This reference was prepared on 2026-09-15 using OMP upstream commit `07a10ebf7263bf419f53c84a5d516037fae61911` (package version `18.2.0`). The local CLI also reported `18.2.0`; this does not prove its build is byte-identical to that upstream commit. Every future task must establish its own target version.

Use the [source map](sources.md) to find official guides, specifications, implementations, and examples. Prefer an immutable target ref over `main` for implementation decisions. Read changelogs/releases between the old and new target when upgrading. If a path moved, inspect the repository tree; do not assume a 404 means the API disappeared.

### Efficient source investigation

1. Inspect the installed package manifest, `exports`, declarations/source, lockfile, and executable version.
2. Find the exact symbol via the language server when available; otherwise use scoped source search.
3. Read the containing interface/function and its relevant call sites, not only a matching line.
4. Follow the runtime path: discovery → module import → factory registration → runner initialization → mode controller/tool wrapper.
5. Inspect the relevant upstream test if error ordering, cancellation, middleware precedence, or reload behavior is unclear.
6. Record only the contracts this extension consumes. Do not map the entire OMP repository.

If docs, declarations, and implementation conflict, use implementation to understand behavior and the installed types to compile. Then run the affected path. Do not work around the discrepancy by disabling type checking.

Observed examples of why this matters:

- At the inspected revision, the example catalog and `examples/extensions/hello.ts` name `execute` arguments in an older order. The canonical `ToolDefinition` is `execute(toolCallId, params, signal, onUpdate, ctx)`. Unused arguments can conceal this error until a tool starts using cancellation or UI.
- `docs/tui.md` documents a smaller `custom` options object than the actual `ExtensionCustomOptions`, which also supports `overlayOptions`, `onHandle`, and `signal`.
- `setFooter` and `setHeader` exist in types but are no-ops in the inspected interactive controller.
- `resources_discover` exists in the extension types/runner, but the inspected guide reports no AgentSession emission call sites. Registration alone does not prove an event fires.

These are snapshot observations, not permanent restrictions.

## Factory and lifecycle

```ts
import type { ExtensionAPI } from "@oh-my-pi/pi-coding-agent";

export default function extension(pi: ExtensionAPI): void {
  pi.registerCommand("acme-check", {
    description: "Check whether the extension command is available",
    handler: async (_args, ctx) => {
      if (!ctx.hasUI) return;
      ctx.ui.notify("Extension command is available", "info");
    },
  });
}
```

This is a small interactive loading probe, not a headless reporting API or a full product scaffold. Verify the chosen command name is not reserved.

The default factory may be synchronous or asynchronous. During loading, register handlers/tools/commands/providers/renderers. Runtime actions such as `sendMessage` are not initialized yet and may throw `ExtensionRuntimeNotInitializedError`. Do not start an unbounded background task at module evaluation time.

General handlers receive `ExtensionContext`; slash-command handlers receive `ExtensionCommandContext`. Session-control actions such as `waitForIdle`, `newSession`, `switchSession`, `branch`, `navigateTree`, and `reload` belong to command context. Do not call command-only wait/control operations from a tool or an event already inside the agent loop: that can deadlock. Treat `await ctx.reload()` as the end of the current handler; return afterward.

Register subscriptions once in the factory. On lifecycle events, rebuild state or replace resources rather than registering another copy of every handler. Release resources on shutdown and handle session switching/branching where relevant. Verify the target runner's cleanup behavior on plugin reload separately from normal process exit.

## Tools

Use the injected schema builder supported by the target. In the inspected OMP version:

```ts
const z = pi.zod;

pi.registerTool({
  name: "acme_text_length",
  label: "Text Length",
  description: "Count Unicode code points in supplied text; does not read files",
  approval: "read",
  parameters: z.object({ text: z.string() }),
  async execute(_id, params, signal, _onUpdate, _ctx) {
    signal?.throwIfAborted();
    let count = 0;
    for (const _point of params.text) count++;
    return {
      content: [{ type: "text", text: String(count) }],
      details: { count },
    };
  },
});
```

The snippet belongs inside the factory. Code-point count is deliberately not grapheme count or terminal cell width.

- `pi.zod` is a Zod-compatible omptype builder, not a promise of all npm Zod features. `pi.arktype` is also exposed; `pi.typebox` is a legacy-compatible shim. Verify any advanced schema combinator and serialized provider schema.
- Keep schemas finite and understandable. Use real enums and bounds; do not accept arbitrary shell strings for a narrow operation.
- `content` is model-visible output; `details` is structured data used by renderers/history. Keep both bounded and avoid secrets in either.
- Handle `onUpdate` as partial output; the final result must remain independently meaningful.
- Pass the `AbortSignal` through filesystem/network/process operations that support it. Stop side effects after cancellation and dispose resources.
- Check current `approval`, `loadMode`, `hidden`, `defaultInactive`, and `strict` semantics before relying on them. Inspected defaults include `approval: "exec"` and `loadMode: "discoverable"`. Set an honest approval tier; do not mark writes or execution as read-only to avoid approval.
- Custom `renderCall` and `renderResult` must tolerate partial/incomplete data, error results, and collapsed/expanded views. They render; they must not execute the operation again.
- Avoid shadowing built-ins unless requested. In the inspected OMP version a shadowing tool may use optional `ctx.invokeTool` to invoke the native implementation of that same tool, not arbitrary tools. Verify availability and forwarding/cancellation semantics before using it.

## Events and message flow

Read exact payload/return unions for the selected event. Do not infer mutability from an object being available.

- `tool_call` may block or revise execution input. Exceptions are fail-closed in the inspected runtime. Validate revised input and do not accidentally block every tool by throwing on an unexpected shape.
- `tool_result` supports middleware-style result changes; later handlers see earlier changes. Test interaction with another relevant extension if ordering matters.
- `tool_execution_*` events observe execution; do not substitute them for interception.
- `input`, `before_agent_start`, and `context` serve different stages. Preserve original content when adding context unless replacement is explicitly intended.
- `message_end` is observational with a detached snapshot at the inspected revision. Mutating it is not a reliable way to alter persisted/provider context.
- `agent_end` is notification-only. OMP's `session_stop` is a distinct continuation-capable surface; use only with a finite condition and verify its subagent/background-job behavior.
- Verify provider-specific hook coverage for provider request interception; a declared hook need not be emitted by every provider.

For message injection, choose delivery deliberately. At the inspected revision `sendMessage` supports `steer`, `followUp`, `nextTurn`, and `aside`, with distinct idle/streaming semantics; `sendUserMessage` enters prompt flow. Do not assume `triggerTurn: false` prevents all work for every delivery mode. Test idle and streaming states if your feature can run in both. Prevent self-triggering loops when observing messages and injecting new ones.

Use package-qualified custom message/entry types, widget keys, and status keys. Core-reserved entry types and unrelated extension identifiers are not your namespace.

## Host services and state

Prefer context/API services instead of a second agent/session/settings instance:

- `ctx.models` offers list/current/resolve/family in the inspected version. Use native resolution and authenticated availability rather than hard-coded model catalogs or provider-name heuristics. Family identifiers are opaque and should not be persisted.
- Use host registry and thinking metadata for model/effort selection. Preserve exact selectors if persisting selectors; do not split suffixes with an invented parser.
- Use live session/context APIs for current working directory, tools, model, prompt, and state. Do not spawn a nested SDK agent merely to access host state.
- A Settings object is not guaranteed to be a field on every public `ExtensionAPI` version. Verify whether and how the target exposes live settings before using it. Never invent `pi.settings` or instantiate another store to imitate an unavailable live API.
- When settings access is supported, honor host storage scope and flush semantics. Do not edit host configuration bytes behind its in-memory settings store.
- Use host provider/auth integration for custom providers. Do not copy credentials into extension data or log request authorization headers.

Separate state by lifetime:

| State | Preferred owner |
|---|---|
| Open dialog selection/loading state | Component or command invocation |
| Per-session reconstructible state | Qualified custom entries/tool details and current session branch |
| Extension-wide user preferences/data | Versioned extension-owned file under a host-resolved data location |
| Host settings | Host's live settings API, if exposed and supported |
| Credentials | Host auth/provider facilities or an explicitly approved secret store |

For session state, rebuild from `ctx.sessionManager.getBranch()` on relevant start/switch/branch/tree paths. Whole-history reconstruction can leak state from another branch. Check entry type and camelCase message roles (`toolResult`, not event name `tool_result`) before reading payloads. Do not mutate history objects to persist changes.

For external files, resolve the actual profile/agent directory through a supported host API rather than hard-coding `~/.omp/agent`. Use schema validation, explicit migrations, atomic replace, and locking/conflict detection when concurrent sessions can write. Distinguish missing files from malformed files; do not reset corrupted data silently. Do not create data files on read-only startup unless required.

## Background work and security

Extensions run in-process, without isolation. A raw timer or detached promise failure can crash the session. Where supported, prefer `ctx.setInterval`, `ctx.setTimeout`, and `ctx.clearTimer`; their inspected implementation contains callback failures and clears timers on shutdown. On older supported hosts, own explicit error containment and cleanup rather than pretending managed APIs exist.

- Catch errors at actual asynchronous boundaries; report through `pi.logger` and appropriate UI/tool results.
- Do not use `console.log` or direct stdout writes in TUI mode; they corrupt rendering and can corrupt RPC framing elsewhere.
- Bound retries, output, caches, polling, and network timeouts. Avoid continuous background work for a command that can query on demand.
- Clean listeners, timers, watchers, subprocesses, sockets, temporary files, status entries, and widgets that you own. Make cleanup idempotent.
- Avoid shell interpolation; prefer executable-plus-arguments APIs and validated paths. Handle Windows paths/spaces and do not embed platform-specific shell syntax in portable code.
- Treat external text as untrusted terminal content, not just untrusted HTML. Sanitize control sequences before display.
- Policy hooks are not OS sandboxing. Do not promise security from matching `rm -rf` or a path substring.
