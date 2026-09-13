---
name: android-tombstone-symbolication
description: Symbolicate the .NET runtime frames in an Android tombstone file. Extracts BuildIds and PC offsets from the native backtrace, downloads debug symbols from the Microsoft symbol server, and runs llvm-symbolizer to produce function names with source file and line numbers. USE FOR triaging a .NET MAUI or Mono Android app crash from a tombstone, resolving native backtrace frames in libmonosgen-2.0.so or libcoreclr.so to .NET runtime source code, or investigating SIGABRT, SIGSEGV, or other native signals originating from the .NET runtime on Android. DO NOT USE FOR pure Java/Kotlin crashes, managed .NET exceptions that are already captured in logcat, or iOS crash logs. INVOKES Symbolicate-Tombstone.ps1 script, llvm-symbolizer, Microsoft symbol server.
license: MIT
---

# Android Tombstone .NET Symbolication

> **Platform**: this machine runs Linux (Arch) — `bash` blocks are the default and directly executable. Windows-only steps live in `Windows (PowerShell)` subsections and are not mixed into Linux instructions.

Resolves native backtrace frames from .NET Android app crashes (MAUI, Xamarin, Mono) to function names, source files, and line numbers using ELF BuildIds and Microsoft's symbol server.

**Inputs:** Tombstone file or logcat crash output, `llvm-symbolizer` (from Android NDK or any LLVM 14+ toolchain), internet access for symbol downloads.

**Do not use when:** The crash is a managed .NET exception (visible in logcat with a managed stack trace), the crashing library is not a .NET component (e.g., `libart.so`), or the tombstone is from iOS.

---

## Workflow

### Step 1: Parse the Tombstone Backtrace

Each backtrace frame has this format:

```
#NN pc OFFSET  /path/to/library.so (optional_symbol+0xNN) (BuildId: HEXSTRING)
```

Extract: **frame number**, **PC offset** (hex, already library-relative), **library name**, and **BuildId** (32–40 hex chars).

Symbolicate all threads by default (background threads like GC/finalizer often have useful .NET frames). The crashing thread's backtrace is listed first; additional threads appear after `--- --- ---` markers.

**Format notes:**
- The script auto-detects `#NN pc` frame lines with or without a `backtrace:` header, and strips logcat timestamp/tag prefixes automatically.
- Logcat-captured tombstones often omit BuildIds. Recover via `adb shell readelf -n`, CI build artifacts, or the .NET runtime NuGet package.
- GitHub issue pastes may mangle `#1 pc` into issue links — replace `org/repo#N pc` with `#N pc` before saving to a file.
- If the script fails to parse a format, fall back to manual extraction of `#NN pc OFFSET library.so (BuildId: HEX)` tuples.

### Step 2: Identify .NET Runtime Libraries

Filter frames to .NET runtime libraries:

| Library | Runtime |
|---------|---------|
| `libmonosgen-2.0.so` | Mono (MAUI, Xamarin, interpreter) |
| `libcoreclr.so` | CoreCLR (JIT mode) |
| `libSystem.*.so` | .NET BCL native components (`Native`, `Globalization.Native`, `IO.Compression.Native`, `Security.Cryptography.Native.OpenSsl`, `Net.Security.Native`) |

**NativeAOT:** No `libcoreclr.so` or `libmonosgen-2.0.so` — the runtime is statically linked into the app binary (e.g., `libMyApp.so`). The `libSystem.*.so` BCL libraries remain separate and can be symbolicated via the symbol server. For the app binary itself, you need the app's own debug symbols.

Skip `libc.so`, `libart.so`, and other Android system libraries unless the user specifically asks.

### Step 3: Download Debug Symbols

For each unique .NET BuildId, download debug symbols:

```
https://msdl.microsoft.com/download/symbols/_.debug/elf-buildid-sym-<BUILDID>/_.debug
```

```bash
curl -sL "https://msdl.microsoft.com/download/symbols/_.debug/elf-buildid-sym-1eb39fc72918c7c6c0c610b79eb3d3d47b2f81be/_.debug" \
  -o libmonosgen-2.0.so.debug
```

Verify with `file libmonosgen-2.0.so.debug` — should show `ELF 64-bit ... with debug_info, not stripped`. If the download returns 404 or HTML, symbols are not published for that build. Do not add or subtract library base addresses — offsets in tombstones are already library-relative.

### Step 4: Symbolicate Each Frame

```bash
llvm-symbolizer --obj=libmonosgen-2.0.so.debug -f -C 0x222098
```

Output:
```
ves_icall_System_Environment_FailFast
/__w/1/s/src/runtime/src/mono/mono/metadata/icall.c:6244
```

The `/__w/1/s/` prefix is the CI workspace root — the meaningful path starts at `src/runtime/`, mapping to [dotnet/dotnet](https://github.com/dotnet/dotnet) VMR.

### Step 5: Present the Symbolicated Backtrace

Combine original frame numbers with resolved function names and source locations:

```
#00  libc.so              abort+164
#01  libmonosgen-2.0.so   ves_icall_System_Environment_FailFast        (mono/metadata/icall.c:6244)
#02  libmonosgen-2.0.so   do_icall                                     (mono/mini/interp.c:2457)
#03  libmonosgen-2.0.so   mono_interp_exec_method                      (mono/mini/interp.c)
```

For unresolved frames (`??`), keep the original line with BuildId and PC offset.

### Automation Script

[scripts/Symbolicate-Tombstone.ps1](scripts/Symbolicate-Tombstone.ps1) automates the full workflow (parse → BuildId/URL report → symbol download → symbolication). The script is PowerShell, so it needs `pwsh`; the manual steps 1–4 above are the equivalent and run with no PowerShell at all.

#### Linux (bash)

`pwsh` is **not installed** on this machine (`pwsh` → not found), and `llvm-symbolizer` is not on `PATH` either, so drive the workflow with the manual commands:

```bash
# Step 1: list frame lines (frame, PC offset, library, BuildId) from a raw tombstone
grep -nE '#[0-9]+ +pc +[0-9a-f]+' tombstone_01.txt | head -n 40

# Step 3: download symbols for one BuildId (repeat per unique BuildId)
curl -sL "https://msdl.microsoft.com/download/symbols/_.debug/elf-buildid-sym-<BUILDID>/_.debug" \
  -o libmonosgen-2.0.so.debug
file libmonosgen-2.0.so.debug

# Step 4: symbolicate — tombstone PC offsets are already library-relative
llvm-symbolizer --obj=libmonosgen-2.0.so.debug -f -C 0x222098
```

Install the symbolizer with `sudo pacman -S llvm` (Arch `extra`; this machine has only `llvm-libs` today), or use the Android NDK copy from [Finding llvm-symbolizer](#finding-llvm-symbolizer).

If you install PowerShell 7+ (`pwsh`) on Linux, the script itself is portable by inspection — it uses only cross-platform cmdlets (`Get-ChildItem`, `Test-Path`, `Join-Path`, `New-Item`, `Remove-Item`, `Write-Host`, `Get-Content`) and invokes `llvm-symbolizer`/`llvm-readelf` as external processes. 待验证 (unverified): that script has not been executed on Linux here, and its `-ParseOnly`/version-lookup paths depend on `llvm-readelf` being present.

#### Windows (PowerShell)

```powershell
pwsh scripts/Symbolicate-Tombstone.ps1 -TombstoneFile tombstone_01.txt -LlvmSymbolizer llvm-symbolizer
```

Flags: `-CrashingThreadOnly` (limit to crashing thread), `-OutputFile path` (write to file), `-ParseOnly` (report libraries/BuildIds/URLs without downloading), `-SkipVersionLookup` (skip runtime version identification).

---

## Finding llvm-symbolizer

Check the **Android NDK** first: `$ANDROID_NDK_ROOT/toolchains/llvm/prebuilt/*/bin/llvm-symbolizer` or `$ANDROID_HOME/ndk/*/toolchains/llvm/prebuilt/*/bin/llvm-symbolizer`. Also available via `brew install llvm`, `apt install llvm`, or `xcrun --find llvm-symbolizer` on macOS.

On this machine (Arch Linux), install it with:

```bash
# llvm is in the `extra` repo; only llvm-libs is installed here today
sudo pacman -S llvm
llvm-symbolizer --version   # also provides llvm-readelf, needed by version lookup
```

If unavailable, complete steps 1–3 and present the download commands and `llvm-symbolizer` commands for the user to run. Do not spend time installing LLVM.

---

## Understanding the Output

CI source paths use these prefixes:

| Path prefix | Maps to |
|---|---|
| `/__w/1/s/src/runtime/` | `src/runtime/` in [dotnet/dotnet](https://github.com/dotnet/dotnet) VMR |
| `/__w/1/s/src/mono/` | `src/mono/` in the VMR (older builds) |
| `/__w/1/s/` | VMR root |

### Runtime Version Identification

The script identifies the exact .NET runtime version by matching BuildIds against locally-installed runtime packs. It searches: SDK packs (`$DOTNET_ROOT/packs/`), NuGet cache (`~/.nuget/packages/`), and NuGet.org as an online fallback. When found, it extracts the version and source commit from the `.nuspec` `<repository commit="..." />` element. Pass `-SkipVersionLookup` to disable. Requires `llvm-readelf` (auto-discovered from the NDK).

---

## Validation

1. `file <debug-file>` shows `ELF ... with debug_info, not stripped`
2. At least one .NET frame resolves to a function name (not `??`)
3. Resolved paths contain recognizable .NET runtime structure (e.g., `mono/metadata/`, `mono/mini/`)

## Stop Signals

- **No .NET frames found**: Report parsed frames and stop.
- **All frames resolved**: Present symbolicated backtrace. Do not trace into source or attempt to build/debug the runtime.
- **Symbols not available (404)**: One attempt per BuildId, then stop. Report unsymbolicated frames with BuildIds and offsets.
- **llvm-symbolizer not available**: Use `-ParseOnly`, present manual commands. Do not install LLVM.

## Common Pitfalls

- **Missing BuildIds**: Logcat tombstones often omit BuildIds. Recover via: `adb shell readelf -n /path/to/lib.so`, CI build artifacts, or the runtime NuGet package (`~/.dotnet/packs/Microsoft.NETCore.App.Runtime.Mono.android-arm64/<version>/`). Prefer pulling raw tombstone files (`adb shell cat /data/tombstones/tombstone_XX`) which always include BuildIds.
- **Symbols not found (404)**: Pre-release/internal builds may not publish symbols. Check for local unstripped `.so`/`.so.dbg` in build artifacts or the NuGet runtime pack.
- **NativeAOT**: No runtime `.so` in the tombstone — runtime is in the app binary. `libSystem.*.so` BCL libraries still work with the symbol server; the app binary needs its own debug symbols.
- **Wrong llvm-symbolizer version**: Use LLVM 14+ for best DWARF compatibility.
- **Multiple BuildIds**: Each .NET library has its own BuildId — download symbols for each separately.
