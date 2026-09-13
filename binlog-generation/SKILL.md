---
name: binlog-generation
description: "Generate MSBuild binary logs (binlogs) for build diagnostics and analysis. USE FOR: adding /bl:{} to any dotnet build, test, pack, publish, or restore command to capture a full build execution trace, prerequisite for binlog-failure-analysis and build-perf-diagnostics skills, enabling post-build investigation of errors or performance. Requires MSBuild 17.8+ / .NET 8 SDK+ for {} placeholder; PowerShell needs -bl:{{}}. DO NOT USE FOR: non-MSBuild build systems (npm, Maven, CMake), analyzing an existing binlog (use binlog-failure-analysis instead)."
license: MIT
---

# Generate Binary Logs

> **Platform**: this machine runs Linux (Arch) — `bash` blocks are the default and directly executable. Windows-only steps live in `Windows (PowerShell)` subsections and are not mixed into Linux instructions.

**Pass the `/bl` switch when running any MSBuild-based command.** This is a non-negotiable requirement for all .NET builds.

## Commands That Require /bl

You MUST add the `/bl:{}` flag to:
- `dotnet build`
- `dotnet test`
- `dotnet pack`
- `dotnet publish`
- `dotnet restore`
- `msbuild` or `msbuild.exe` (`msbuild.exe` is Windows-only; on Linux the entry point is `dotnet msbuild`, which works on both platforms)
- Any other command that invokes MSBuild

## Preferred: Use `{}` for Automatic Unique Names

> **Note:** The `{}` placeholder requires MSBuild 17.8+ / .NET 8 SDK or later.

The `{}` placeholder in the binlog filename is replaced by MSBuild with a unique identifier, guaranteeing no two builds ever overwrite each other — without needing to track or check existing files.

### Linux (bash)

```bash
# Every invocation produces a distinct file automatically
dotnet build /bl:{}
dotnet test /bl:{}
dotnet build --configuration Release /bl:{}
```

`bash` passes `{}` through literally here — braces only trigger brace expansion when they contain a comma or a range (e.g. `{a,b}`), so no escaping is needed.

### Windows (PowerShell)

PowerShell requires escaping the braces:

```powershell
# PowerShell: escape { } as {{ }}
dotnet build -bl:{{}}
dotnet test -bl:{{}}
```

In `cmd.exe` (unlike PowerShell) the braces pass through unescaped, so `dotnet build /bl:{}` works there too.

## Why This Matters

1. **Unique names prevent overwrites** - You can always go back and analyze previous builds
2. **Failure analysis** - When a build fails, the binlog is already there for immediate analysis
3. **Comparison** - You can compare builds before and after changes
4. **No re-running builds** - You never need to re-run a failed build just to generate a binlog

## Examples

### Linux (bash)

```bash
# ✅ CORRECT - {} generates a unique name automatically (bash/cmd)
dotnet build /bl:{}
dotnet test /bl:{}

# ❌ WRONG - Missing /bl flag entirely
dotnet build
dotnet test

# ❌ WRONG - No filename (overwrites the same msbuild.binlog every time)
dotnet build /bl
dotnet build /bl
```

### Windows (PowerShell)

```powershell
# ✅ CORRECT - PowerShell escaping
dotnet build -bl:{{}}
dotnet test -bl:{{}}

# ❌ WRONG - Missing /bl flag entirely
dotnet build
dotnet test

# ❌ WRONG - No filename (overwrites the same msbuild.binlog every time)
dotnet build /bl
dotnet build /bl
```

## When a Specific Filename Is Required

If the binlog filename needs to be known upfront (e.g., for CI artifact upload), or if `{}` is not available in the installed MSBuild version, pick a name that won't collide with existing files:

1. Check for existing `*.binlog` files in the directory
2. Choose a name not already taken (e.g., by incrementing a counter from the highest existing number)

### Linux (bash)

```bash
# List existing binlogs, newest first
ls -1t *.binlog 2>/dev/null | head -n 1
```

### Windows (PowerShell)

```powershell
# List existing binlogs
Get-ChildItem *.binlog | Select-Object -First 1
```

Then build with the chosen name (identical on both platforms):

```bash
# Example: directory contains 3.binlog — use 4.binlog
dotnet build /bl:4.binlog
```

## Cleaning the Repository

When cleaning the repository with `git clean`, **always exclude binlog files** to preserve your build history:

```bash
# ✅ CORRECT - Exclude binlog files from cleaning
git clean -fdx -e "*.binlog"

# ❌ WRONG - This deletes binlog files (they're usually in .gitignore)
git clean -fdx
```

This is especially important when iterating on build fixes - you need the binlogs to analyze what changed between builds.
