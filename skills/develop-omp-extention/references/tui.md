# Native TUI development

Read the target revision's `docs/tui.md`, extension UI types, mode controller, and the relevant `@oh-my-pi/pi-tui` component implementation before coding. Names below describe the inspected 18.2.0 snapshot, not a frozen interface.

## Prefer native surfaces in this order

1. `ctx.ui.notify`, `setStatus`, `setWorkingMessage`, or `setWidget` for information that does not need a modal.
2. `select`, `confirm`, `input`, and `editor` for standard user decisions and text editing. Use `askDialog` only if present and supported in the target mode.
3. `ctx.ui.custom` composed from native `Container`, `Text`, `Box`, `Spacer`, `SelectList`, `SettingsList`, `Input`, `Editor`, `ScrollView`, `TabBar`, `Loader`, or `CancellableLoader`, as appropriate and actually exported by the target.
4. A custom `Component` only for genuinely new layout or interaction requirements.

Reuse native selection, filtering, scrolling, editor behavior, and keyboard parsing. Do not build a custom model catalog/browser, effort enum, theme engine, terminal driver, or key parser when the host provides a suitable supported facility. If a model browser is an internal deep export rather than public UI API, check its constructor, callbacks, required services, and compatibility cost before choosing it. Native reuse does not mean copying its implementation into the extension.

Use native theme adapters such as `getSelectListTheme`, `getSettingsListTheme`, `getEditorTheme`, and `getMarkdownTheme` where the target exports them. Avoid new terminal UI frameworks or a second TUI instance competing with OMP's renderer.

## Check the runtime mode

`ctx.hasUI` is a necessary guard for many UI flows, not proof that every method works.

| Mode at inspected snapshot | Standard dialogs | Custom component |
|---|---|---|
| Interactive OMP terminal | Native | Mounted |
| Print/headless, some subagent paths | No-op/default when no UI context | Not interactive |
| RPC | Client request/response bridge | Unsupported; not mounted |
| ACP | Elicitation bridge, client capability-dependent | Not a native terminal surface |

Do not invent a `ctx.mode` property or rely solely on truthiness of `ui.custom`. Verify mode/capability detection in the target API. If the host exposes no reliable custom-TUI capability check, keep that feature explicitly terminal-only and provide a standard-dialog/argument-based path for other supported modes. A missing UI must not silently authorize destructive work or leave a promise waiting forever.

Inspected interactive `setFooter` and `setHeader` are no-ops despite their type declarations. Use working status/widget surfaces unless newer implementation proves otherwise.

## Mounting and ownership

The inspected factory order is:

```ts
ctx.ui.custom<Result>((tui, theme, keybindings, done) => component, options);
```

The third argument is the keybindings manager; the fourth is `done`. `Result` and `component` above stand for your concrete result type and returned native/custom component, not globally defined variables.

- Default custom UI replaces the editor area. Overlay mode is requested with `{ overlay: true }`.
- Current `ExtensionCustomOptions` also includes `overlayOptions`, `onHandle`, and `signal`; consult types and implementation, not only the shorter docs signature.
- Let the host mount/unmount, restore editor text/focus, and hide the overlay. Do not manually seize raw terminal mode or alternate-screen ownership for an ordinary dialog.
- Close with `done(result)` on success and the defined cancellation result on user cancel. Settle once, even if cancel races with asynchronous completion.
- If aborting via `options.signal`, the inspected controller rejects the promise and cleans up. Handle that separately from an ordinary selected value or `undefined` cancellation.
- Implement idempotent `dispose()` whenever the component owns resources. Prevent stale completion callbacks from rendering into a closed dialog.
- Keep terminal-only actions out of `render`. Render must not perform network calls, save files, or change settings.

## Layout, rendering, and performance

The inspected `Component` contract includes `render(width): readonly string[]`, optional `handleInput`, `invalidate`, `dispose`, and `wantsKeyRelease`. `Focusable` is separate, with `focused`; cursor placement uses `CURSOR_MARKER`, not an invented `getCursorPosition` method.

- Count visible terminal cells with `visibleWidth`, not JavaScript `length`.
- Wrap/truncate with ANSI-aware `wrapTextWithAnsi` and `truncateToWidth`. Subtract border, padding, icon, and scrollbar width before laying out content.
- Normalize tabs with `replaceTabs`; use the target's appropriate sanitization for carriage returns, ANSI/OSC escape sequences, and other untrusted controls. `replaceTabs` alone is not a complete sanitizer.
- Fit every row to the supplied width. The renderer's emergency truncation is not your layout strategy.
- Budget height from actual available terminal/overlay dimensions. `render(width)` itself does not supply height. Account for headers, footers, borders, and scroll affordances.
- Keep the active item visible. For small terminals, use scrolling or a compact layout instead of negative dimensions or hidden confirmation buttons.
- Invalidate caches on content, width, theme, and symbol-preset changes. Request repaint through `tui.requestRender` after input or asynchronous state changes.
- Return the same immutable row array when content is unchanged if practical. Return a new array on changes; do not mutate arrays already returned to the renderer. Advanced `RenderStablePrefix` handling is only justified for a measured need.
- Avoid rebuilding large datasets, reading disk, or allocating theme dictionaries on every paint. Cache based on real state and invalidate correctly; do not cache stale host state indefinitely.
- Test long names, empty lists, CJK, combining characters, pasted newlines, and terminal control sequences. Do not assume every glyph has one-cell width.

## Input and focus

Use native components' input handlers, `matchesKey`, and the provided keybindings manager. Do not compare escape-sequence byte strings manually.

The inspected interactive controller passes `KeybindingsManager.inMemory()` to custom factories, carrying defaults rather than the user's full `keybindings.yml`. Match action IDs where available, but do not promise custom dialogs automatically honor every user remapping. Recheck this behavior on upgrades.

- Preserve keyboard-only operation and a visible focused state.
- Keep focus, current cursor selection, checked items, active state, and disabled state distinguishable.
- Use native `Input`/`Editor` for editing, paste, cursor movement, and IME behavior.
- Do not consume unrelated keys. Do not register reserved global shortcuts; inspect the current reserved set and existing bindings.
- Forward focus state to focused child editors where their API requires it. Avoid two simultaneous input owners.
- Key-release events are filtered by default; request them only for an actual feature, then use native release/repeat helpers.
- Keep footer hints consistent with actual handlers and the active dialog mode.
- Confirm destructive actions. Cancel must not mutate persisted state unless the UX explicitly documents immediate apply.

## Nerd Icons and user preferences

Use Nerd Icons where they help identify actions, files, status, navigation, or settings **through OMP's native theme symbol layer**. Respect the user's resolved symbol preset; never force Nerd Font glyphs or enable the setting on the user's behalf.

### Actual inspected setting and API

The relevant OMP setting is not an assumed `nerdIcons: boolean`. In the inspected `settings-schema.ts`:

- `symbolPreset` accepts `"unicode"`, `"nerd"`, or `"ascii"`.
- The default is `"unicode"`.
- Appearance labels this setting **Symbol Preset**, with **Unicode**, **Nerd Font**, and **ASCII** choices.
- `statusLine.preset` is a separate layout/presentation choice; do not use it as the global icon-enabled flag.

The resolved `Theme` exposes:

- `theme.symbol(key)` for a typed `SymbolKey`.
- Semantic accessors such as `theme.status.success`, `theme.status.error`, `theme.nav.cursor`, `theme.nav.selected`, `theme.icon.folder`, `theme.icon.file`, `theme.icon.model`, `theme.checkbox`, `theme.boxRound`, and `theme.boxSharp`.
- `theme.getSymbolPreset()` for the effective `"unicode" | "nerd" | "ascii"` preset.
- `theme.getLangIcon(language)` and, where exported, `getLangIconStyled(language)` for native language/file-kind presentation.

Prefer a semantic symbol lookup to explicit preset branches. The theme combines preset defaults with symbol overrides; consulting a raw config value or copying the Nerd Font table would bypass those overrides.

```ts
// Inside a renderer with the live host Theme and actual width:
const glyph = theme.symbol("icon.folder");
const label = glyph ? `${glyph} Projects` : "Projects";
const line = truncateToWidth(theme.fg("accent", label), width);
```

Import `truncateToWidth` from the verified `@oh-my-pi/pi-tui` export. The example deliberately does not embed a private-use glyph or assume a fixed icon width.

### Rendering rules

1. Use existing semantic keys for matching concepts. Do not repurpose an unrelated icon solely because it looks attractive.
2. When Nerd Font is enabled, native keys select the appropriate Nerd glyphs. When disabled, allow the theme's Unicode or ASCII equivalents; do not leak Nerd private-use code points into those modes.
3. If a genuinely new icon has no native key, prefer readable text. Only add a tiny extension-local preset-aware fallback if the icon is important: Nerd glyph in verified `nerd` mode, ordinary Unicode in `unicode`, ASCII/text in `ascii`. Keep that mapping local and documented, not a parallel icon system.
4. Pair important icons with text. Selection, failure, and destructive meaning must remain understandable without color, a specific font, or the icon alone.
5. Measure the complete rendered label, including icon and spacing. An icon can be empty, multi-character in ASCII, or wider than expected in a terminal/font. Avoid leaving an extra blank column when the icon is empty.
6. Fetch resolved symbols from the current theme at render/update time. Do not resolve all icons once at module load. The inspected host can replace the Theme object when the preset changes; invalidate theme-dependent caches and use the host's current theme path on repaint/reopen.
7. Do not infer font support from OS, terminal name, environment variables, or the mere presence of a Nerd Font on disk. Honor the user's choice. If glyphs render as boxes, explain the font prerequisite and offer the existing Unicode/ASCII setting; do not silently change it.
8. Keep machine-readable tool data, persisted identifiers, and protocol payloads free of decorative icons. Add symbols in presentation, not canonical data.

### Required icon checks for UI changes

- Nerd Font preset with a suitable terminal font: meaningful native glyphs and correct spacing.
- Unicode preset with Nerd Icons disabled: no hard-coded Nerd private-use glyphs from the extension.
- ASCII preset: readable equivalents, no assumed one-character markers.
- Switch preset or reopen after changing it: no stale glyph or width cache.
- Narrow terminal and long labels in each relevant preset.
- Theme overrides and light/dark themes where the extension customizes presentation.

If terminal font rendering cannot be observed, report glyph visibility as unverified even when width and preset-selection tests pass. For a pi port, re-research pi's theme capabilities; do not assume OMP's symbol APIs or setting names exist there.

## Async operations and errors

Prefer `CancellableLoader` for cancellable work when its current API fits. Forward its signal; catch rejection; close or display the real error according to the workflow. The short official cancellation example is a pattern, not permission to omit rejection handling or allow double settlement.

Represent at least idle/loading/success/error or cancelled states as needed. Disable duplicate submissions while committing. Do not dismiss a settings/storage error and imply the operation succeeded. Preserve enough user input to retry safely, without adding speculative retries.

## Live smoke checklist

Open the actual command in OMP; navigate with keyboard; exercise selection/edit/apply; cancel and reopen; trigger one real error; resize; check both width and height; verify editor text/focus restoration; switch/reopen under Nerd/Unicode/ASCII; ensure no leaked widget, timer, or duplicated handler after reload. Tests of returned strings are useful boundary checks, not a substitute for this interaction.
