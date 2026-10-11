# Unreal-AI-EnhancedInput

The **Unreal AI EnhancedInput** extension for AI Game Developer (Unreal-MCP): a C++ `Type=Editor` UE plugin
(`UnrealAIEnhancedInput`) implementing `IUnrealMcpToolProvider` that contributes MCP tools wrapping the engine's built-in **Enhanced Input** plugin (authoring Input Actions / Mapping Contexts). Scaffolded
from `IvanMurzak/Unreal-AI-Template` — it is now its own extension; do not edit it as the template.

**Gating:** the `EnhancedInput` engine plugin (`.Build.cs` module dep + `.uplugin` `Plugins[]` entry) — the extension won't compile or load without it.

## Layout

- `UnrealAIEnhancedInput/` — the plugin (the only folder the release zip ships): `UnrealAIEnhancedInput.uplugin` + `Source/UnrealAIEnhancedInput/`
  (`UnrealAIEnhancedInput.Build.cs`, `Private/UnrealAIEnhancedInputModule.cpp` = provider `FUnrealAIEnhancedInputProvider` + module, `Private/Tests/` = Automation specs).
- `Tests/e2e/` — `Run-ToolChecks.ps1` harness + `tools/<tool-id>.e2e.ps1`, one check per tool.
- `commands/` — `bump-version`, `get-version`, `update-core`; `init` was the one-time scaffold (not rerun).
- `extension.json` — install-catalog manifest (`version` mirrors the `.uplugin`; `minCoreVersion`, `enginePlugins`, `tools[]`).
- `.github/workflows/` — `test_pull_request.yml` (calls `test_unreal_plugin.yml` + E2E), version-gated `release.yml`, `bump_version.yml`.
- `docs/claude/` — `architecture.md` (extension shape, contract), `ci.md` (workflows, repo variables,
  self-hosted runner gating), `release.md` (version gate, atomic source-zip release).
- `README.md` — user walkthrough: tools → gating plugin → build → register → add a tool → release → install.

## Tools

`extension.json` `tools[]` and the README table are the source of truth — keep them in sync.

- `enhanced-input-list` — list Input Actions / Mapping Contexts (Asset Registry), read-only.
- `enhanced-input-get` — read an action's or context's scalar config, read-only.
- `enhanced-input-action-create` — create a `UInputAction` asset (value type) at a content path, mutating.
- `enhanced-input-context-create` — create a `UInputMappingContext` asset at a content path, mutating.
- `enhanced-input-add-mapping` — add a key → action mapping to a mapping context, mutating.

## The contract (read before editing tools)

- `IUnrealMcpToolProvider` (Unreal-MCP `UnrealMcpRuntime/Public/IUnrealMcpToolProvider.h`):
  `GetExtensionId()` / `GetDisplayName()` / `GetExtensionVersion()` / `RegisterTools(FUnrealMcpToolRegistry&)`.
- Tools use the fluent builder: `Registry.Tool("kebab-id").Title(...).Description(...).Param*(...).Handle([](const FUnrealMcpToolCall&){...})`.
  `.Description(...)` is the `tools/list` text — always set it. Do NOT call `.ExtensionId(...)` — it is stamped from the provider.
- The provider is registered as a **modular feature** in `StartupModule` and unregistered in
  `ShutdownModule`; Unreal-MCP discovers it on boot or live.
- Handlers run on the **game thread** (call editor/engine APIs directly) and must be **defensive**: UE has
  no C++ exceptions, so validate params and engine state (`GEditor`, null assets) and return
  `FUnrealMcpToolResult::Error(...)` — never an unchecked deref (a handler crash is an editor crash).
- Tool ids MUST match `^[a-z0-9]+(-[a-z0-9]+)*$` or the registry drops them.
- The module is **unity-built** (all `.cpp` in one TU): an anonymous namespace does NOT make a helper
  file-private — give file-local helpers a module-unique name (prefix `UnrealAIEnhancedInput_`).

## Commands

```powershell
./commands/bump-version.ps1 -NewVersion "<x.y.z>"  # .uplugin VersionName + GetExtensionVersion() + extension.json
./commands/get-version.ps1                         # prints the .uplugin VersionName
./commands/update-core.ps1                         # refreshes extension.json minCoreVersion from Unreal-MCP releases
```

## Build / test (local loop)

Junction `UnrealAIEnhancedInput/` into a UE project that has the `UnrealMCP` core plugin available (the workspace testbed
`engines/unreal/test-project` already junctions `Plugins/UnrealMCP`), build the editor target with UBT, and
run the Automation specs with filter `UnrealAIEnhancedInput` (`README.md` → "### 2. Build against a UE project (UBT)"). CI does the same on a
self-hosted Windows UE runner.

## Conventions

- **Naming:** repo `Unreal-AI-EnhancedInput` (hyphens); plugin + module `UnrealAIEnhancedInput` (no hyphens — UE module names can't
  contain `-`); C++ prefixes `F*`/`U*`/`I*`; tool ids kebab-case `enhanced-input-<op>`.
- **C++ style:** Unreal — tabs, braces on new lines, UE types. File header: the
  `// Copyright (c) 2026 ...` Apache-2.0 one-liner.
- **Versioning:** the `.uplugin` `VersionName` is the single source of truth (`GetExtensionVersion()` and
  `extension.json` mirror it); never hand-edit one location alone — use `bump-version.ps1`.
- **Tests:** one UE Automation spec + one E2E `unreal-mcp-cli` check **per tool**.
- **Distribution:** GitHub-Release source zip `UnrealAIEnhancedInput-<version>.zip` at tag `v<version>`; UE compiles it on
  the consumer's next editor open. Not NuGet, not precompiled binaries.
- **Secrets:** never commit `.env` or tokens.
