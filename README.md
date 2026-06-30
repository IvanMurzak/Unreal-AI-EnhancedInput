<h1 align="center">Unreal AI EnhancedInput</h1>

<p align="center">
  An <b>MCP tool extension</b> for
  <a href="https://github.com/IvanMurzak/Unreal-MCP">AI Game Developer (Unreal-MCP)</a> that lets an
  AI agent author <b>Enhanced Input</b> assets — Input Actions, Mapping Contexts, and key bindings —
  directly in the Unreal Editor.
</p>

---

`Unreal AI EnhancedInput` is a normal Unreal Engine **`Type=Editor` plugin** (`UnrealAIEnhancedInput`)
that implements the Unreal-MCP contract `IUnrealMcpToolProvider` and registers its tools through a
fluent builder. Unreal-MCP discovers it at boot (and live, when it loads later) and merges its tools
into the advertised tool set, so an agent can create and inspect Enhanced Input assets the same way a
designer would in the editor.

It **wraps Unreal's built-in `EnhancedInput` engine plugin**. That dependency is the *gating*: the
extension won't compile or load without `EnhancedInput` — which is exactly the point, the tools have
no meaning without it.

> Authoring is **C++** (unlike Unity's C# `[McpPluginTool]`). The handlers are deliberately thin and
> **defensive** — a proof of capability over the Enhanced Input surface, not exhaustive coverage.

## Tools

| Tool | Kind | Purpose |
| --- | --- | --- |
| `enhanced-input-action-create` | mutating | Create a `UInputAction` asset (value type) at a content path. |
| `enhanced-input-context-create` | mutating | Create a `UInputMappingContext` asset at a content path. |
| `enhanced-input-add-mapping` | mutating | Add a key → action mapping to a mapping context. |
| `enhanced-input-list` | read-only | List the project's Input Actions / Mapping Contexts (Asset Registry). |
| `enhanced-input-get` | read-only | Read an action's / context's scalar config. |

## What's in here

```
UnrealAIEnhancedInput/                                   the UE plugin
├── UnrealAIEnhancedInput.uplugin                        descriptor; Plugins: [ UnrealMCP, EnhancedInput ]
└── Source/UnrealAIEnhancedInput/
    ├── UnrealAIEnhancedInput.Build.cs                   deps: UnrealMcpRuntime + UnrealMcpEditor + EnhancedInput(+Editor)
    └── Private/
        ├── UnrealAIEnhancedInputModule.cpp              provider + module + the enhanced-input-* tools
        └── Tests/UnrealAIEnhancedInputSpec.cpp          UE Automation specs (1 per tool)
commands/                                                bump-version / get-version / update-core
Tests/e2e/                                               E2E unreal-mcp-cli tool checks (1 per tool)
extension.json                                           install-catalog / compat manifest
.github/workflows/                                       CI (test_pull_request + release)
```

# Working on this extension

### 1. The gating engine plugin

The extension takes a real code dependency on Enhanced Input in two places (already wired):

1. `UnrealAIEnhancedInput.uplugin` lists it in `Plugins`:
   ```json
   "Plugins": [
     { "Name": "UnrealMCP",     "Enabled": true },
     { "Name": "EnhancedInput", "Enabled": true }
   ]
   ```
2. `Source/UnrealAIEnhancedInput/UnrealAIEnhancedInput.Build.cs` depends on `"EnhancedInput"` and
   `"EnhancedInputEditor"`.

`install-extension` enables `EnhancedInput` in the consumer `.uproject` alongside this plugin.

### 2. Build against a UE project (UBT)

The extension is compiled by Unreal Build Tool inside a host project that also has the **UnrealMCP
core plugin** available. The fastest local loop is a directory junction:

```powershell
# From a UE 5.7 C++ project that has Plugins/UnrealMCP available:
cmd /c mklink /J "<UEProject>\Plugins\UnrealAIEnhancedInput" "<thisRepo>\UnrealAIEnhancedInput"

& "C:\Program Files\Epic Games\UE_5.7\Engine\Build\BatchFiles\Build.bat" `
  <UEProject>Editor Win64 Development -project="<UEProject>\<UEProject>.uproject" -WaitMutex
```

A clean build compiles the module, its tools, and the Automation specs. Run the specs with:

```powershell
& "C:\Program Files\Epic Games\UE_5.7\Engine\Binaries\Win64\UnrealEditor-Cmd.exe" `
  "<UEProject>\<UEProject>.uproject" -nullrhi -nosplash -unattended `
  -ExecCmds="Automation RunTests UnrealAIEnhancedInput; Quit" -ReportExportPath="<dir>" -log
```

### 3. Register & see the tools in AI Game Developer

Enable both plugins in the project, open the editor, and connect AI Game Developer (the Unreal-MCP
UI / sidecar). `StartupModule` registers the provider as a modular feature, so the `enhanced-input-*`
tools appear in the tool list immediately. Toggling the extension live-updates the advertised tools.

### 4. Add or change a tool

Edit `Source/UnrealAIEnhancedInput/Private/UnrealAIEnhancedInputModule.cpp` → `RegisterTools()`. Tool
ids are **kebab-case** and the handler runs on the **game thread**, so you may call editor / engine
APIs directly — but validate engine state first and return `FUnrealMcpToolResult::Error(...)` rather
than dereferencing a null asset (handlers must never crash the editor):

```cpp
Registry.Tool(TEXT("enhanced-input-action-create"))
    .Title(TEXT("Create Input Action"))
    .Description(TEXT("Creates a UInputAction asset (value type) at the given content path."))
    .ParamString(TEXT("path"), TEXT("Content path, e.g. /Game/Input/IA_Jump"))
    .Handle([](const FUnrealMcpToolCall& Call) -> FUnrealMcpToolResult
    {
        const FString Path = Call.GetString(TEXT("path"));
        if (Path.IsEmpty()) { return FUnrealMcpToolResult::Error(TEXT("'path' is required.")); }
        // ... use the Enhanced Input + AssetTools editor APIs here ...
        return FUnrealMcpToolResult::Success(FString::Printf(TEXT("Created %s"), *Path));
    });
```

For each tool, add **(1)** a focused UE Automation spec (an `It(...)` block in
`Tests/UnrealAIEnhancedInputSpec.cpp`) and **(2)** an E2E check (`Tests/e2e/tools/<tool>.e2e.ps1`).
See the [Unreal-MCP extension author guide](https://github.com/IvanMurzak/Unreal-MCP/blob/main/docs/EXTENSIONS.md).

### 5. Release

Versioning is single-sourced from the `.uplugin` `VersionName`. Bump it in lock-step:

```powershell
./commands/bump-version.ps1 -NewVersion "0.2.0"   # updates .uplugin + GetExtensionVersion() + extension.json
```

Push to `main`. **`release.yml` is version-gated**: when the `VersionName` is a new value with no
existing tag, it runs the full test suite, packages the plugin **source** into a single
`UnrealAIEnhancedInput-<version>.zip`, and creates an **atomic GitHub Release** (tag `v<version>`)
carrying that zip — the exact asset the installer downloads. The extension ships as source and UE
compiles it on the consumer's next editor open. (Track the core version floor with
`./commands/update-core.ps1`.)

### 6. Install via the CLI

Once released, install into a UE project (the project path is a **positional** argument):

```bash
unreal-mcp-cli install-extension com.ivanmurzak.unreal-ai-enhanced-input <UEProject>
# offline / from a local checkout (no published release needed):
unreal-mcp-cli install-extension com.ivanmurzak.unreal-ai-enhanced-input <UEProject> --source <path-to-plugin-dir>
```

The CLI resolves the release zip
(`releases/download/v<version>/UnrealAIEnhancedInput-<version>.zip`), places the plugin in
`Plugins/UnrealAIEnhancedInput/`, enables it (and `EnhancedInput`) in the `.uproject`, and the editor
recompiles it on next open (or pass `--build` to compile now via UBT). The same capability backs the
AI-Game-Dev desktop app button and the in-editor Extensions panel.

---

## CI & secrets

| Workflow | When | What |
| --- | --- | --- |
| `test_unreal_plugin.yml` | reusable | UBT build + UE Automation specs for one UE version |
| `test_pull_request.yml` | PR | the reusable test per UE version (5.6/5.7) + E2E `unreal-mcp-cli` tool checks |
| `release.yml` | push to `main` | version-gated → full tests → package source zip `UnrealAIEnhancedInput-<version>.zip` → atomic GitHub Release (tag `v<version>`) |
| `bump_version.yml` | manual | runs `bump-version.ps1`, opens a release PR |

The plugin/E2E jobs run on a **self-hosted Windows UE runner** and are **never red-by-absence** —
they stay *skipped* until a runner is registered and the repo variables are set:

- `UNREAL_RUNNER_READY = true` — enables the UBT build + Automation legs.
- `UNREAL_E2E_READY = true` — enables the E2E `install-extension` + tool-invocation leg.
- `UNREAL_HOST_PROJECT` — absolute path on the runner to a host `.uproject` with UnrealMCP available.

See [`docs/claude/ci.md`](docs/claude/ci.md) and [`docs/claude/release.md`](docs/claude/release.md).

## License

[Apache-2.0](LICENSE).
