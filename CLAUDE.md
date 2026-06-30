# Unreal-AI-EnhancedInput

This repo is the **`Unreal AI EnhancedInput`** extension for
[AI Game Developer (Unreal-MCP)](https://github.com/IvanMurzak/Unreal-MCP) — a C++ `Type=Editor` UE
plugin (`UnrealAIEnhancedInput`) that implements `IUnrealMcpToolProvider` and contributes MCP tools
an AI agent can call against the Unreal Editor to author **Enhanced Input** assets. It wraps Unreal's
built-in **`EnhancedInput`** engine plugin; that dependency **is the gating** — the extension does not
compile or load without `EnhancedInput` present.

It was scaffolded from [`Unreal-AI-Template`](https://github.com/IvanMurzak/Unreal-AI-Template) via
`commands/init.ps1` and is now its own extension — do not edit it as if it were the template.

## The tool family (Enhanced Input)

A thin, defensive proof-of-capability over the Enhanced Input authoring surface (kebab-case ids,
`enhanced-input-*`):

| Tool | Kind | Purpose |
| --- | --- | --- |
| `enhanced-input-action-create` | mutating | Create a `UInputAction` asset (value type) at a content path. |
| `enhanced-input-context-create` | mutating | Create a `UInputMappingContext` asset at a content path. |
| `enhanced-input-add-mapping` | mutating | Add a key → action mapping to a mapping context. |
| `enhanced-input-list` | read-only | List the project's Input Actions / Mapping Contexts (Asset Registry). |
| `enhanced-input-get` | read-only | Read an action's or context's scalar config. |

The exact UE 5.7 Enhanced Input API surface is verified and the handlers authored in the
implementation step; treat this table as the intended family and keep it in sync with
`extension.json` `tools[]` and the README as tools land.

## The contract (read before editing tools)

- `IUnrealMcpToolProvider` (in Unreal-MCP `UnrealMcpRuntime/Public/IUnrealMcpToolProvider.h`):
  `GetExtensionId()` / `GetDisplayName()` / `GetExtensionVersion()` / `RegisterTools(FUnrealMcpToolRegistry&)`.
  This extension's provider is `FUnrealAIEnhancedInputProvider`
  (`Source/UnrealAIEnhancedInput/Private/UnrealAIEnhancedInputModule.cpp`).
- Tools are declared with the fluent builder: `Registry.Tool("kebab-id").Title(...).Description(...).Param*(...).Handle([](const FUnrealMcpToolCall&){...})`.
  `.Description(...)` is the `tools/list` text and is REQUIRED.
- The provider is registered as a **modular feature** in `StartupModule` and unregistered in
  `ShutdownModule`; Unreal-MCP discovers it on boot or live.
- Handlers run on the **game thread** (call editor/engine APIs directly) and must be **defensive**:
  UE has no C++ exceptions, so validate required params and engine state (`GEditor`, null assets),
  then return `FUnrealMcpToolResult::Error(...)` — never an unchecked deref (a crash in a handler is
  an editor crash).
- The module is **unity-built** (every `.cpp` concatenated into one TU), so an anonymous namespace
  does NOT make a helper file-private — give file-local specs/helpers a module-unique name (prefix
  `UnrealAIEnhancedInput_`).
- Tool ids MUST match `^[a-z0-9]+(-[a-z0-9]+)*$` or the registry drops them. Do not call
  `.ExtensionId(...)` — it is stamped from the provider.

## Commands

```powershell
./commands/bump-version.ps1 -NewVersion "0.2.0"   # .uplugin VersionName + GetExtensionVersion() + extension.json (lock-step)
./commands/get-version.ps1                        # prints the .uplugin VersionName (single source of truth)
./commands/update-core.ps1                        # refreshes extension.json minCoreVersion from Unreal-MCP releases
```

(`commands/init.ps1` was the one-time scaffolding step and is not run again.)

## Build / test (local loop)

Junction this plugin into a UE 5.7 project that has the `UnrealMCP` core plugin available, then build
the editor target with UBT (see `README.md` step 4). Run Automation specs with the filter set to the
module name `UnrealAIEnhancedInput`. One UE Automation `It(...)` + one `Tests/e2e/tools/<tool>.e2e.ps1`
check per tool; read-only tools assert a well-formed success, mutating/required-input tools assert a
defensive error. CI does the same on a self-hosted Windows UE runner.

## Conventions

- **Naming:** repo `Unreal-AI-EnhancedInput` (hyphens); plugin + module `UnrealAIEnhancedInput`
  (no hyphens — UE module names can't contain `-`); C++ prefixes `F*`/`U*`/`I*`; tool ids kebab-case.
- **C++ style:** Unreal — tabs, braces on new lines, UE types. File header: the
  `// Copyright (c) 2026 ...` Apache-2.0 one-liner.
- **Versioning:** the `.uplugin` `VersionName` is the single source of truth; never hand-edit one
  version location alone — use `bump-version.ps1`.
- **Distribution:** a single GitHub-Release **source zip** `UnrealAIEnhancedInput-<version>.zip` at
  tag `v<version>`; UE compiles it on the consumer's next editor open. Not NuGet, not precompiled.
- **Secrets:** never commit `.env` or tokens.

## Find detail in

- `README.md` — the full build → register → release → install walkthrough for this extension.
- `docs/claude/architecture.md` — extension shape, the contract, layout.
- `docs/claude/ci.md` — workflows, required repo variables, self-hosted runner gating.
- `docs/claude/release.md` — version gate + atomic release mechanics.
