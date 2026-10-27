# Packaging, imports, installation, and releases

Commands describe the inspected OMP 18.2.0 behavior. Inspect the actual target's `omp --help` and `omp plugin --help` before running them. All names, owners, repositories, and release tags in examples are illustrative; substitute real values. Do not execute example public installs blindly.

## Minimal package structure

```text
my-omp-extension/
  package.json
  src/index.ts
  test/                 # behavior tests when useful
  tsconfig.json
  README.md
  CHANGELOG.md          # if the project uses one
  lockfile
```

Do not create empty directories or mandatory files without a purpose. TypeScript source can be the entrypoint; a compiled bundle is not required simply because this is a plugin.

A minimal distribution manifest:

```json
{
  "name": "my-omp-extension",
  "version": "0.1.0",
  "type": "module",
  "description": "A concise description of the actual behavior",
  "omp": {
    "extensions": ["./src/index.ts"]
  },
  "files": ["src", "README.md", "CHANGELOG.md"]
}
```

Add a license, repository metadata, scripts, engines, and real dependency declarations appropriate to the project. Include only files that exist and are needed. `files` is a packaging inclusion list, not an OMP API-compatibility declaration. `private: true` prevents npm publication; it does not turn a public GitHub repository into a private repository or preclude source-based distribution.

OMP prefers `omp` over the legacy `pi` manifest key. Use `omp` for OMP-only work. Multiple extension entries are allowed, but list only actual factories; do not list every helper source file. Each entry is loaded independently, so duplicate registration is possible if factories import/register each other.

Installed-plugin entries support `.ts`, `.js`, `.mjs`, `.cjs`, and directories with supported index files at the inspected revision. Native auto-discovery scans have different rules. Explicit entrypoints avoid accidental helper-file discovery and unreliable implicit index resolution.

Plugin manifests can include capabilities and optional feature groups beyond extensions. Inspect `PluginManifest` and the discovery implementation before adding them; do not assume every directory name or a field copied from another plugin is recognized. No invented `omp.minVersion` field: express host compatibility through real package constraints and documentation unless the target defines a dedicated manifest field.

## Dependencies and imports

### Host-provided packages

For OMP-native code, use canonical target exports such as:

```ts
import type { ExtensionAPI, ExtensionContext } from "@oh-my-pi/pi-coding-agent";
import { Container, Text, SelectList } from "@oh-my-pi/pi-tui";
```

Only import symbols actually used and exported in the target. Use `import type` for type-only dependencies. Do not import source through an absolute developer path, another repository's `node_modules`, or `@oh-my-pi/pi-coding-agent/src/...` unless that exact path is a documented export, which should not be assumed.

Prefer package-root public exports. If a supported capability only has a deep exported path, inspect `package.json.exports`, the actual implementation, and host loader resolution; document that tighter compatibility dependency. A wildcard export does not make every internal module stable.

For packages supplied by the host, normally declare tested compatible versions as `peerDependencies` and install exact development versions as `devDependencies` for typechecking/tests. Declare only host packages you actually import. Choose a bounded range based on real tests; an unbounded `>=18.x` asserts compatibility with unknown future breaking releases. Exact pins are honest for a narrow initial support claim; widen only after verification. npm peer constraints are not proof that OMP's runtime loader enforces every compatibility condition.

The inspected loader remaps canonical and several legacy pi package namespaces to host copies/shims. Do not bundle another runtime copy of the host coding-agent/TUI just to satisfy imports. If bundling is necessary for a specific deployment, externalize verified host package imports and test the packaged result through OMP. A successful standalone Bun import can resolve differently from an extension loaded by the host.

### Third-party dependencies

- Put runtime libraries used by shipped code in `dependencies`, not only `devDependencies`.
- Use ordinary package imports resolved from the extension's own package graph. Install them in the extension directory before loading a linked or explicit-path extension.
- Do not rely on a library incidentally present in OMP's transitive dependencies.
- Use the injected OMP schema builders rather than adding a second schema library without a reason. Compatibility shims do not cover every subpath of TypeBox/Zod or every advanced library API.
- Use `node:` built-ins where supported. Prefer Node-compatible domain code if a pi port is requested, but do not claim all Bun APIs work in pi.
- Resolve shipped assets relative to `import.meta.url`, not process cwd. Test the installed copy and symlinked checkout.
- Check package exports, TS module resolution, runtime resolution, and assets together. Avoid compiler-only `paths` aliases that the runtime cannot resolve.

For a new Bun-oriented project, ESM, strict TypeScript, `noEmit`, and an appropriate modern module resolution mode are reasonable. Select exact compiler/lib/types settings against the real installed packages instead of copying an upstream monorepo tsconfig with workspace aliases or catalog dependencies.

## One-off local loading

From the extension repository:

```sh
bun install
omp --extension ./src/index.ts
```

Or load its manifest directory:

```sh
omp -e .
```

Use the repository's existing package manager if different and supported. `-e` is repeatable at the inspected revision. For an isolated loading investigation, the inspected `--no-extensions` disables ambient extension discovery but still permits explicit extension paths; verify target help/source first. It does not mean every other ambient capability or plugin resource is disabled.

The usual native discovery roots are `<cwd>/.omp/extensions/` and the active agent directory's `extensions/`. Default agent root is `~/.omp/agent`, but profiles and environment overrides change it. Directory scanning is shallow and differs from explicit manifest resolution. Do not tell users to scatter helper `.ts` files directly in an auto-scanned root.

## Persistent local development link

From the intended checkout:

```sh
bun install
omp plugin link .
omp plugin list --json
```

The inspected `link` requires a `package.json` with a package name, creates a user-root symlink, and records an enabled plugin. It may replace an existing target of the same name. Check the current installation first and do not overwrite an unrelated source accidentally.

Important scope detail: inspected `PluginManager.link()` links into the user plugin root. Do not claim `--scope project` works for this path merely because the top-level CLI exposes a scope flag; that flag primarily applies to marketplace operations here. `omp plugin install ./path` routes to local linking in the inspected CLI as well. Prefer `link` to make the intention explicit.

After source edits:

1. Reinstall dependencies if manifest/lockfile changed.
2. Run the relevant checks.
3. Refresh `omp plugin link .` when package name/manifest/entrypoints change, or when project workflow requires it.
4. Inspect the plugin list for the expected enabled package, source/link, feature selection, and extension entrypoints.
5. Reload only through a mechanism verified to reload runtime extensions; otherwise restart OMP.
6. Invoke the command/tool and smoke-test the changed behavior.

Do not copy source files into OMP's plugin directory alongside the link. Do not reinstall a source-only edit as a separate package copy. A running process may retain imported code until actual runtime reload/restart even though the symlink points at changed files.

### Reload is version-sensitive

Do not equate `/reload-plugins`, a catalog refresh, `ctx.reload()`, and process restart. Inspect their target-version implementations. Some plugin-refresh paths reload skills/commands/MCP without rebuilding initialized extension/tool registrations. A displayed “reloaded” message is not proof that your new factory ran. Restart is the conservative verification path when the applicable reload behavior is uncertain; then verify a behavior that distinguishes the changed code from the old code.

## Install from GitHub after publication

For a real public repository containing the package and entrypoint:

```sh
omp plugin install github:OWNER/REPOSITORY
omp plugin list --json
```

For reproducible releases, publish an actual tag and use the inspected Git shorthand ref syntax:

```sh
omp plugin install "github:OWNER/REPOSITORY#v0.2.0"
```

The syntax is `github:owner/repo`, not `github:@owner/repo`. npm syntax such as `@scope/package` is a different distribution channel and requires a published npm package. A GitHub repository alone does not create one.

Direct-install update examples:

```sh
omp plugin install "github:OWNER/REPOSITORY#v0.2.1" --force
```

Or reinstall the unpinned repository source with `--force` when intentionally following its default branch. Verify the resulting package version and source/ref; unpinned branches are not reproducible releases. The inspected Git installer handles ref changes/cache refresh, but do not assume a package's version string alone changes the Git ref.

Restart or use a proven runtime reload, inspect diagnostics, and run the feature. Preserve extension-owned data. Test direct GitHub installation from a clean environment before advertising it; a linked checkout may hide ignored files, missing runtime dependencies, or missing assets.

## Marketplace and npm are separate flows

Marketplace registration/install uses its actual catalog and identity:

```sh
omp plugin marketplace add OWNER/MARKETPLACE
omp plugin install PLUGIN@MARKETPLACE --scope user
omp plugin upgrade PLUGIN@MARKETPLACE --scope user
```

Substitute the registered marketplace name, which may differ from the repository name. Verify catalog schema and scope semantics. Marketplace catalog update is not necessarily installed plugin upgrade. In the inspected runtime, `plugin upgrade` is for marketplace-managed plugins, not the universal update command for linked/direct GitHub packages.

If separately published to npm, use a real package/version:

```sh
omp plugin install @SCOPE/PACKAGE@0.2.0
```

Do not advertise unpublished registry coordinates or imply every marketplace source type is implemented just because a parser recognizes it.

## Versioning policy

Keep three versions distinct:

1. Extension release version (`package.json.version`, and any real authoritative release metadata).
2. Compatible host OMP version/range.
3. Extension persistent-data schema version.

Follow the project's SemVer and pre-1.0 policy:

| Change | Typical release decision |
|---|---|
| Backward-compatible bug fix | Patch |
| Backward-compatible user feature | Minor |
| Breaking commands/tool schemas/config/data semantics or dropped supported host | Major, or explicitly documented pre-1.0 breaking convention |
| Internal unreleased iteration | Usually no bump yet |

A new OMP patch/minor can still break an extension. Review actual changes and test; do not rely solely on the host's SemVer label. Raising the minimum host version can break existing users even if extension source changes are small.

Use existing version/release tooling. Some package-manager version commands create commits/tags automatically; check flags before invoking. Update relevant lockfile root metadata through the package manager rather than blindly replacing every matching version string. Do not bump all OMP development packages solely because the extension version changed.

### Release checklist

1. Select the next extension version and supported host range from actual behavior/tests.
2. Update package/version authority, release notes, README installation/compatibility instructions, and any migration notes.
3. Install/update dependencies as needed, typecheck, run required tests, and smoke-test in target OMP.
4. Inspect the distributable using the project's pack tooling or a clean checkout. Confirm entrypoints, assets, runtime dependencies, license, and absence of credentials/local files.
5. Verify clean installation without access to development-only dependencies.
6. With authorization, commit/tag/push and publish only to the requested channel. Keep tag and package version consistent; do not silently overwrite a published tag.
7. Install the actual published GitHub tag/package in a clean test environment and exercise it.
8. Document update and rollback to an earlier known-good ref. If a schema migration is irreversible, make that constraint explicit before promising rollback.

For linked checkout updates, `git pull --ff-only` is reasonable only when the user requested updating from upstream and local branch/worktree state permits it. It is not a mandatory command after every local source edit.
