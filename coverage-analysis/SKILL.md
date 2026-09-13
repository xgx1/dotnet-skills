---
name: coverage-analysis
description: >
  Project-wide code coverage and CRAP (Change Risk Anti-Patterns) score
  analysis for .NET projects. Calculates CRAP scores per method and surfaces
  risk hotspots — complex code with low coverage that is dangerous to modify.
  Use to diagnose why coverage is stuck or plateaued, identify what methods
  block improvement, or get project-wide coverage analysis with risk ranking.
  USE FOR: coverage stuck, coverage plateau, can't increase coverage, what's
  blocking coverage, coverage gap, CRAP scores, risk hotspots, where to add
  tests, coverage analysis, coverage report.
  DO NOT USE FOR: targeted single-method CRAP analysis (use crap-score);
  auditing test code for coverage-touching or other anti-patterns (use
  test-anti-patterns); writing tests; running tests (use run-tests). Requires
  or produces coverage (Cobertura) and CRAP metrics.
license: MIT
---

# Coverage Analysis

> **Platform**: this machine runs Linux (Arch) — `bash` blocks are the default and directly executable. Windows-only steps live in `Windows (PowerShell)` subsections and are not mixed into Linux instructions.

## Purpose

Raw coverage percentages answer "what code was executed?" — they don't answer what you actually need to know:

- **What tests should I write next?** — ranked by risk and impact
- **Which uncovered code is risky vs. trivial?** — CRAP scores separate the two
- **Why has coverage plateaued?** — identify the files blocking further gains
- **Is this code safe to refactor?** — complex + uncovered = dangerous to change

This skill bridges that gap: from a bare .NET solution to a prioritized risk hotspot list, with no manual tool configuration required.

## When to Use

Use this skill when the user mentions test coverage, coverage gaps, code risk, CRAP scores, where to add tests, why coverage plateaued, or wants to know which code is safest to refactor — even if they don't explicitly say "coverage analysis".

## When Not to Use

- **Targeted single-method CRAP analysis** — use the `crap-score` skill instead
- **Writing or generating tests** — this skill identifies where tests are needed, not write them
- **General test execution** unrelated to coverage or CRAP analysis
- **Coverage reporting without CRAP context** — use `dotnet test` with coverage collection directly

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| Project/solution path | No | Current directory | Path to the .NET solution or project |
| Line coverage threshold | No | 80% | Minimum acceptable line coverage |
| Branch coverage threshold | No | 70% | Minimum acceptable branch coverage |
| CRAP threshold | No | 30 | Maximum acceptable CRAP score before flagging |
| Top N hotspots | No | 10 | Number of risk hotspots to surface |

### Prerequisites

- .NET SDK installed (`dotnet` on PATH)
- **Platform**: the `dotnet` CLI is cross-platform, so most commands below run unchanged in either shell. Every shell-specific step is split into a `Linux (bash)` subsection (default, directly executable here) and a `Windows (PowerShell)` subsection. Two dotnet-specific differences to keep in mind: environment variables are `export MSBUILDUSESERVER=1` in bash vs `$env:MSBUILDUSESERVER=1` in PowerShell, and the MSBuild binlog placeholder is `-bl:{}` in bash but must be escaped as `-bl:{{}}` in PowerShell. `pwsh` is **not installed** on this machine, so the bundled `scripts/*.ps1` helpers cannot be executed here — Phase 3 provides verified `python3` equivalents.
- At least one test project referencing the production code (xUnit, NUnit, or MSTest) — only required for the from-scratch path; not needed when the user supplies an existing Cobertura XML
- **Optional, only for the from-scratch path:** internet/NuGet access for `dotnet add package coverlet.collector` (or `Microsoft.Testing.Extensions.CodeCoverage`) when a test project has no coverage provider yet. Skip when the user supplies an existing Cobertura XML.
- **Optional, only for Phase 5:** internet access for `dotnet tool install` (ReportGenerator). Core CRAP/coverage analysis works from Cobertura XML alone — ReportGenerator only adds HTML/CSV reports as an optional post-summary extra.

The skill auto-detects coverage provider state per test project and selects the least-invasive execution strategy:

- unified Microsoft CodeCoverage when all projects use it,
- unified Coverlet when no project uses Microsoft CodeCoverage,
- per-project provider execution when the solution is truly mixed.

No pre-existing runsettings files or manually installed tools required.

## Workflow

> **MANDATORY: deliver the final assistant response with the CRAP/risk-hotspot summary BEFORE any optional work.** As soon as the Phase 3 analysis commands (`Compute-CrapScores.ps1` / `Extract-MethodCoverage.ps1` on Windows, their `python3` equivalents on Linux) return data, your **next** assistant response must contain the user-facing analysis (CRAP table, blocking methods, recommendations). Do not run ReportGenerator (Phase 5), do not install global tools, and do not start any heavy parallel work before that response is delivered. The user is judged on the final assistant message, not on side-effect files.
>
> If a phase fails, times out, or budget is running low, skip remaining optional work and immediately return a partial summary containing: (1) what was found in the Cobertura XML, (2) any CRAP/risk-hotspot data already extracted, (3) which methods are blocking coverage, and (4) failures encountered.

If the user provides a path to existing Cobertura XML (or coverage data is already present in `TestResults/`), **skip Phase 2 entirely** (no test execution) **and skip Phase 5 by default** (no ReportGenerator install or HTML report) — go directly from Phase 3 (analysis scripts) to Phase 4 (user-facing summary). Only run Phase 5 if the user explicitly asks for HTML/CSV reports. The Risk Hotspots table and CRAP scores are mandatory in every output — they are the skill's core value-add over raw coverage numbers.

The workflow runs in five phases. Phases 1–4 are required; Phase 5 (ReportGenerator HTML/CSV reports) is strictly optional and runs **after** the user-facing summary has been delivered. Do not parallelize Phase 5 with earlier phases — the heavy `dotnet tool install` for ReportGenerator can crash the session before Phase 4 completes.

### Phase 1 — Setup (sequential)

#### Step 1: Locate the solution or project

Given the user's path (default: current directory), find the entry point:

##### Linux (bash)

Uses `find`, `grep`, `git`, and `mapfile` only — no PowerShell dependency.

```bash
root="<user-provided-path-or-current-directory>"
root=$(realpath "$root")

# Prefer solution file; fall back to project file
sln=$(find "$root" -maxdepth 2 -name '*.sln' -type f 2>/dev/null | head -n 1)
if [ -n "$sln" ]; then
    echo "ENTRY_TYPE:Solution"; echo "ENTRY:$(realpath "$sln")"
else
    project=$(find "$root" -maxdepth 2 -name '*.csproj' -type f 2>/dev/null | head -n 1)
    if [ -n "$project" ]; then
        echo "ENTRY_TYPE:Project"; echo "ENTRY:$(realpath "$project")"
    else
        echo "ENTRY_TYPE:NotFound"
    fi
fi

# Test projects: search path first, then git root, then parent
search_roots=("$root")
git_root=$(git -C "$root" rev-parse --show-toplevel 2>/dev/null)
if [ -n "$git_root" ]; then git_root=$(realpath "$git_root"); fi
if [ -n "$git_root" ] && [ "$git_root" != "$root" ]; then search_roots+=("$git_root"); fi
parent_path=$(dirname "$root")
if [ -n "$parent_path" ] && [ "$parent_path" != "$root" ] && [ "$parent_path" != "$git_root" ]; then search_roots+=("$parent_path"); fi

# $1 = search root, $2 = "content" (test-framework reference) or "name" (file-name convention)
collect_test_projects() {
    local sr="$1" mode="$2"
    find "$sr" -maxdepth 5 -name '*.csproj' -type f 2>/dev/null \
        | grep -v -E '/(obj|bin)/' \
        | while IFS= read -r proj; do
              if [ "$mode" = content ]; then
                  grep -q -E 'Microsoft\.NET\.Test\.Sdk|xunit|nunit|MSTest\.TestAdapter|"MSTest"|MSTest\.TestFramework|TUnit' "$proj" \
                      && printf '%s\n' "$proj"
              else
                  basename "$proj" | grep -q -i -E '(test|spec)' && printf '%s\n' "$proj"
              fi
          done
}

test_projects=()
for sr in "${search_roots[@]}"; do
    mapfile -t test_projects < <(collect_test_projects "$sr" content)
    if [ "${#test_projects[@]}" -gt 0 ]; then
        if [ "$sr" != "$root" ]; then echo "SEARCHED:$sr"; fi
        break
    fi
done
if [ "${#test_projects[@]}" -eq 0 ]; then
    for sr in "${search_roots[@]}"; do
        mapfile -t test_projects < <(collect_test_projects "$sr" name)
        if [ "${#test_projects[@]}" -gt 0 ]; then
            if [ "$sr" != "$root" ]; then echo "SEARCHED:$sr"; fi
            break
        fi
    done
fi
echo "TEST_PROJECTS:${#test_projects[@]}"
for tp in "${test_projects[@]}"; do echo "TEST_PROJECT:$tp"; done

# Resolve the test output root (where coverage-analysis artifacts will be written)
if [ "${#test_projects[@]}" -eq 0 ]; then
    if [ -n "$git_root" ]; then test_output_root="$git_root"; else test_output_root="$root"; fi
elif [ "${#test_projects[@]}" -eq 1 ]; then
    test_output_root=$(dirname "${test_projects[0]}")
else
    # Multiple test projects — find their deepest common parent directory
    common=$(dirname "${test_projects[0]}")
    for tp in "${test_projects[@]:1}"; do
        d=$(dirname "$tp")
        while [ "$d" != "$common" ] && [ "${d#"$common"/}" = "$d" ]; do
            prev="$common"
            common=$(dirname "$common")
            # Terminate if we can no longer move up (at filesystem root or no parent)
            if [ -z "$common" ] || [ "$common" = "$prev" ]; then common=""; break; fi
        done
    done
    if [ -z "$common" ]; then
        # Fallback when no common parent directory exists (e.g., projects under different mounts)
        if [ -n "$git_root" ]; then test_output_root="$git_root"; else test_output_root="$root"; fi
    else
        test_output_root="$common"
    fi
fi
echo "TEST_OUTPUT_ROOT:$test_output_root"
```

##### Windows (PowerShell)

```powershell
$root = "<user-provided-path-or-current-directory>"

# Prefer solution file; fall back to project file
$sln = Get-ChildItem -Path $root -Filter "*.sln" -Recurse -Depth 2 -ErrorAction SilentlyContinue |
    Select-Object -First 1
if ($sln) {
    Write-Host "ENTRY_TYPE:Solution"; Write-Host "ENTRY:$($sln.FullName)"
} else {
    $project = Get-ChildItem -Path $root -Filter "*.csproj" -Recurse -Depth 2 -ErrorAction SilentlyContinue |
        Select-Object -First 1
    if ($project) {
        Write-Host "ENTRY_TYPE:Project"; Write-Host "ENTRY:$($project.FullName)"
    } else {
        Write-Host "ENTRY_TYPE:NotFound"
    }
}

# Test projects: search path first, then git root, then parent
$searchRoots = @($root)
$gitRoot = (git -C $root rev-parse --show-toplevel 2>$null)
if ($gitRoot) { $gitRoot = [System.IO.Path]::GetFullPath($gitRoot) }
if ($gitRoot -and $gitRoot -ne $root) { $searchRoots += $gitRoot }
$parentPath = Split-Path $root -Parent
if ($parentPath -and $parentPath -ne $root -and $parentPath -ne $gitRoot) { $searchRoots += $parentPath }

$testProjects = @()
foreach ($sr in $searchRoots) {
    # Primary: match by .csproj content (test framework references)
    $testProjects = @(Get-ChildItem -Path $sr -Filter "*.csproj" -Recurse -Depth 5 -ErrorAction SilentlyContinue |
        Where-Object { $_.FullName -notmatch '([/\\]obj[/\\]|[/\\]bin[/\\])' } |
        Where-Object { (Select-String -Path $_.FullName -Pattern 'Microsoft\.NET\.Test\.Sdk|xunit|nunit|MSTest\.TestAdapter|"MSTest"|MSTest\.TestFramework|TUnit' -Quiet) })
    if ($testProjects.Count -gt 0) {
        if ($sr -ne $root) { Write-Host "SEARCHED:$sr" }
        break
    }
}

# Fallback: match by file name convention
if ($testProjects.Count -eq 0) {
    foreach ($sr in $searchRoots) {
        $testProjects = @(Get-ChildItem -Path $sr -Filter "*.csproj" -Recurse -Depth 5 -ErrorAction SilentlyContinue |
            Where-Object { $_.Name -match '(?i)(test|spec)' })
        if ($testProjects.Count -gt 0) {
            if ($sr -ne $root) { Write-Host "SEARCHED:$sr" }
            break
        }
    }
}
Write-Host "TEST_PROJECTS:$($testProjects.Count)"
$testProjects | ForEach-Object { Write-Host "TEST_PROJECT:$($_.FullName)" }

# Resolve the test output root (where coverage-analysis artifacts will be written)
if ($testProjects.Count -eq 0) {
    if ($gitRoot) {
        $testOutputRoot = $gitRoot
    } else {
        $testOutputRoot = $root
    }
} elseif ($testProjects.Count -eq 1) {
    $testOutputRoot = $testProjects[0].DirectoryName
} else {
    # Multiple test projects — find their deepest common parent directory
    $dirs = $testProjects | ForEach-Object { $_.DirectoryName }
    $common = $dirs[0]
    foreach ($d in $dirs[1..($dirs.Count-1)]) {
        $sep = [System.IO.Path]::DirectorySeparatorChar
        while (-not $d.StartsWith("$common$sep", [System.StringComparison]::OrdinalIgnoreCase) -and $d -ne $common) {
            $prevCommon = $common
            $common = Split-Path $common -Parent
            # Terminate if we can no longer move up (at filesystem root or no parent)
            if ([string]::IsNullOrEmpty($common) -or $common -eq $prevCommon) {
                $common = $null
                break
            }
        }
    }
    if ([string]::IsNullOrEmpty($common)) {
        # Fallback when no common parent directory exists (e.g., projects on different drives)
        if ($gitRoot) {
            $testOutputRoot = $gitRoot
        } else {
            $testOutputRoot = $root
        }
    } else {
        $testOutputRoot = $common
    }
}
Write-Host "TEST_OUTPUT_ROOT:$testOutputRoot"
```

- If `ENTRY_TYPE:NotFound` and test projects were found → use the test projects directly as entry points (run `dotnet test` on each test `.csproj`).
- If `ENTRY_TYPE:NotFound` and no test projects found → stop: `No .sln or test projects found under <path>. Provide the path to your .NET solution or project.`
- If `TEST_PROJECTS:0` and `EXISTING_COBERTURA_COUNT` > 0 (Step 2b) → continue with existing Cobertura XML analysis (no `dotnet test` run).
- If `TEST_PROJECTS:0` and `EXISTING_COBERTURA_COUNT` == 0 → stop: `No test projects found (expected projects with 'Test' or 'Spec' in the name), and no existing Cobertura XML was provided. Add a test project or provide a Cobertura file path.`

#### Step 2: Create the output directory

##### Linux (bash)

```bash
coverage_dir="$test_output_root/TestResults/coverage-analysis"
rm -rf "$coverage_dir"
mkdir -p "$coverage_dir"
echo "COVERAGE_DIR:$coverage_dir"
```

##### Windows (PowerShell)

```powershell
$coverageDir = Join-Path $testOutputRoot "TestResults" "coverage-analysis"
if (Test-Path $coverageDir) { Remove-Item $coverageDir -Recurse -Force }
New-Item -ItemType Directory -Path $coverageDir -Force | Out-Null
Write-Host "COVERAGE_DIR:$coverageDir"
```

This step only manages the `TestResults/coverage-analysis/` subdirectory (skill-owned outputs). It must never delete user-supplied Cobertura files — those live one level up at `TestResults/coverage.cobertura.xml` (or wherever the user pointed). If the user provided a path that *is* `TestResults/coverage-analysis/...`, copy the file aside before this step recreates the directory.

#### Step 2b: Discover or accept existing Cobertura XML (required for the existing-data path)

If the user supplied a Cobertura XML path explicitly, use it. Otherwise probe well-known locations and any path the user mentioned:

##### Linux (bash)

```bash
# 1. Honor a user-supplied path first (highest priority)
cobertura_files=()
if [ -n "$user_supplied_cobertura_path" ] && [ -e "$user_supplied_cobertura_path" ]; then
    cobertura_files=("$user_supplied_cobertura_path")
fi

# 2. Otherwise scan TestResults/ at the repo/test root for any *.cobertura.xml
if [ "${#cobertura_files[@]}" -eq 0 ]; then
    for sp in "$test_output_root/TestResults" "$root/TestResults"; do
        [ -d "$sp" ] || continue
        mapfile -t found < <(find "$sp" -name '*.cobertura.xml' -type f 2>/dev/null \
            | grep -v -E '/coverage-analysis/raw/')
        if [ "${#found[@]}" -gt 0 ]; then cobertura_files=("${found[@]}"); break; fi
    done
fi

echo "EXISTING_COBERTURA_COUNT:${#cobertura_files[@]}"
for f in "${cobertura_files[@]}"; do echo "EXISTING_COBERTURA:$f"; done
```

##### Windows (PowerShell)

```powershell
# 1. Honor a user-supplied path first (highest priority)
$coberturaFiles = @()
if ($userSuppliedCoberturaPath -and (Test-Path $userSuppliedCoberturaPath)) {
    $coberturaFiles = @(Get-Item $userSuppliedCoberturaPath)
}

# 2. Otherwise scan TestResults/ at the repo/test root for any *.cobertura.xml
if ($coberturaFiles.Count -eq 0) {
    $searchPaths = @(
        (Join-Path $testOutputRoot "TestResults"),
        (Join-Path $root "TestResults")
    ) | Where-Object { $_ -and (Test-Path $_) } | Select-Object -Unique
    foreach ($sp in $searchPaths) {
        $found = @(Get-ChildItem -Path $sp -Filter "*.cobertura.xml" -Recurse -ErrorAction SilentlyContinue |
            Where-Object { $_.FullName -notmatch '[/\\]coverage-analysis[/\\]raw[/\\]' })
        if ($found.Count -gt 0) { $coberturaFiles = $found; break }
    }
}

Write-Host "EXISTING_COBERTURA_COUNT:$($coberturaFiles.Count)"
$coberturaFiles | ForEach-Object { Write-Host "EXISTING_COBERTURA:$($_.FullName)" }
```

- If `EXISTING_COBERTURA_COUNT` > 0 → **skip Phase 2 entirely** and pass these paths to the Phase 3 scripts.
- If `EXISTING_COBERTURA_COUNT` == 0 → run Phase 2 to generate fresh coverage; the file paths to feed Phase 3 will be discovered from `<COVERAGE_DIR>/raw/` after `dotnet test`.

#### Step 2c: Recommend ignoring `TestResults/`

##### Linux (bash)

```bash
pattern='**/TestResults/'
git_root=$(git -C "$test_output_root" rev-parse --show-toplevel 2>/dev/null)
if [ -n "$git_root" ]; then
    git_root=$(realpath "$git_root")
    gitignore_path="$git_root/.gitignore"
    if [ -f "$gitignore_path" ] \
        && grep -q -E '^[[:space:]]*(\*\*/)?TestResults/?[[:space:]]*$' "$gitignore_path"; then
        echo "GITIGNORE_RECOMMENDATION:already-present"
    else
        echo "GITIGNORE_RECOMMENDATION:$pattern"
    fi
else
    echo "GITIGNORE_RECOMMENDATION:$pattern"
fi
```

##### Windows (PowerShell)

```powershell
$pattern = "**/TestResults/"
$gitRoot = (git -C $testOutputRoot rev-parse --show-toplevel 2>$null)
if ($gitRoot) { $gitRoot = [System.IO.Path]::GetFullPath($gitRoot) }
if ($gitRoot) {
    $gitignorePath = Join-Path $gitRoot ".gitignore"
    $alreadyIgnored = $false
    if (Test-Path $gitignorePath) {
        $alreadyIgnored = (Select-String -Path $gitignorePath -Pattern '^\s*(\*\*/)?TestResults/?\s*$' -Quiet)
    }
    if ($alreadyIgnored) {
        Write-Host "GITIGNORE_RECOMMENDATION:already-present"
    } else {
        Write-Host "GITIGNORE_RECOMMENDATION:$pattern"
    }
} else {
    Write-Host "GITIGNORE_RECOMMENDATION:$pattern"
}
```

### Phase 2 — Test execution (skip when Cobertura XML already exists)

Run only when no Cobertura XML is present. If the user already has coverage data, skip directly to Phase 3.

#### Step 3: Detect coverage provider and run `dotnet test` with coverage collection

Before running tests, detect which coverage provider the test projects use. Projects may reference
`Microsoft.Testing.Extensions.CodeCoverage` (Microsoft's built-in provider, common on .NET 9+) or
`coverlet.collector` (open-source, the default in xUnit templates). The provider determines which
`dotnet test` arguments to use — both produce Cobertura XML.

##### Linux (bash)

```bash
# Detect coverage provider per test project
coverage_provider="unknown"  # will be set to "ms-codecoverage" or "coverlet"
ms_codecov_projects=()
coverlet_projects=()
neither_projects=()

for tp in "${test_projects[@]}"; do
    if grep -q -E 'Microsoft\.Testing\.Extensions\.CodeCoverage' "$tp"; then
        ms_codecov_projects+=("$tp")
    elif grep -q -E 'coverlet\.collector' "$tp"; then
        coverlet_projects+=("$tp")
    else
        neither_projects+=("$tp")
    fi
done

# Determine the provider strategy
if [ "${#ms_codecov_projects[@]}" -gt 0 ] && [ "${#coverlet_projects[@]}" -eq 0 ]; then
    coverage_provider="ms-codecoverage"
    echo "COVERAGE_PROVIDER:ms-codecoverage (ms:${#ms_codecov_projects[@]}, none:${#neither_projects[@]})"
elif [ "${#coverlet_projects[@]}" -gt 0 ] && [ "${#ms_codecov_projects[@]}" -eq 0 ]; then
    coverage_provider="coverlet"
    echo "COVERAGE_PROVIDER:coverlet (coverlet:${#coverlet_projects[@]}, none:${#neither_projects[@]})"
elif [ "${#ms_codecov_projects[@]}" -gt 0 ] && [ "${#coverlet_projects[@]}" -gt 0 ]; then
    coverage_provider="mixed-project"
    echo "COVERAGE_PROVIDER:mixed-project (ms:${#ms_codecov_projects[@]}, coverlet:${#coverlet_projects[@]}, none:${#neither_projects[@]})"
else
    coverage_provider="coverlet"
    echo "COVERAGE_PROVIDER:none-detected — defaulting to coverlet"
fi
```

##### Windows (PowerShell)

```powershell
# Detect coverage provider per test project
$coverageProvider = "unknown"  # will be set to "ms-codecoverage" or "coverlet"
$msCodeCovProjects = @()
$coverletProjects = @()
$neitherProjects = @()

foreach ($tp in $testProjects) {
    $hasMsCodeCov = Select-String -Path $tp.FullName -Pattern 'Microsoft\.Testing\.Extensions\.CodeCoverage' -Quiet
    $hasCoverlet = Select-String -Path $tp.FullName -Pattern 'coverlet\.collector' -Quiet
    if ($hasMsCodeCov) { $msCodeCovProjects += $tp }
    elseif ($hasCoverlet) { $coverletProjects += $tp }
    else { $neitherProjects += $tp }
}

# Determine the provider strategy
if ($msCodeCovProjects.Count -gt 0 -and $coverletProjects.Count -eq 0) {
    $coverageProvider = "ms-codecoverage"
    Write-Host "COVERAGE_PROVIDER:ms-codecoverage (ms:$($msCodeCovProjects.Count), none:$($neitherProjects.Count))"
} elseif ($coverletProjects.Count -gt 0 -and $msCodeCovProjects.Count -eq 0) {
    $coverageProvider = "coverlet"
    Write-Host "COVERAGE_PROVIDER:coverlet (coverlet:$($coverletProjects.Count), none:$($neitherProjects.Count))"
} elseif ($msCodeCovProjects.Count -gt 0 -and $coverletProjects.Count -gt 0) {
    $coverageProvider = "mixed-project"
    Write-Host "COVERAGE_PROVIDER:mixed-project (ms:$($msCodeCovProjects.Count), coverlet:$($coverletProjects.Count), none:$($neitherProjects.Count))"
} else {
    $coverageProvider = "coverlet"
    Write-Host "COVERAGE_PROVIDER:none-detected — defaulting to coverlet"
}
```

If any discovered test projects have no provider, add one based on the selected strategy:

##### Linux (bash)

```bash
if [ "$coverage_provider" = "ms-codecoverage" ] && [ "${#neither_projects[@]}" -gt 0 ]; then
    echo "ADDING_MS_CODECOVERAGE:${#neither_projects[@]} project(s)"
    for tp in "${neither_projects[@]}"; do
        dotnet add "$tp" package Microsoft.Testing.Extensions.CodeCoverage --no-restore
        echo "  ADDED_MS_CODECOVERAGE:$tp"
    done
    for tp in "${neither_projects[@]}"; do
        dotnet restore "$tp" --quiet
    done
fi

if { [ "$coverage_provider" = "coverlet" ] || [ "$coverage_provider" = "mixed-project" ]; } \
    && [ "${#neither_projects[@]}" -gt 0 ]; then
    echo "ADDING_COVERLET:${#neither_projects[@]} project(s)"
    for tp in "${neither_projects[@]}"; do
        dotnet add "$tp" package coverlet.collector --no-restore
        echo "  ADDED:$tp"
    done
    for tp in "${neither_projects[@]}"; do
        dotnet restore "$tp" --quiet
    done
fi
```

##### Windows (PowerShell)

```powershell
if ($coverageProvider -eq "ms-codecoverage" -and $neitherProjects.Count -gt 0) {
    Write-Host "ADDING_MS_CODECOVERAGE:$($neitherProjects.Count) project(s)"
    foreach ($tp in $neitherProjects) {
        dotnet add $tp.FullName package Microsoft.Testing.Extensions.CodeCoverage --no-restore
        Write-Host "  ADDED_MS_CODECOVERAGE:$($tp.FullName)"
    }
    foreach ($tp in $neitherProjects) {
        dotnet restore $tp.FullName --quiet
    }
}

if (($coverageProvider -eq "coverlet" -or $coverageProvider -eq "mixed-project") -and $neitherProjects.Count -gt 0) {
    Write-Host "ADDING_COVERLET:$($neitherProjects.Count) project(s)"
    foreach ($tp in $neitherProjects) {
        dotnet add $tp.FullName package coverlet.collector --no-restore
        Write-Host "  ADDED:$($tp.FullName)"
    }
    foreach ($tp in $neitherProjects) {
        dotnet restore $tp.FullName --quiet
    }
}
```

Log each addition to the console so the developer sees what changed. Document the additions in the final report (see Output Format).

Run one `dotnet test` per entry point for the selected strategy:

- In `ms-codecoverage` or `coverlet` mode: run a single command for the solution entry (or one per test project if no `.sln` was found).
- In `mixed-project` mode: run one command per test project, using that project's existing provider to avoid dual-provider conflicts.

**Coverlet** (`coverlet.collector`):

##### Linux (bash)

```bash
raw_dir="<COVERAGE_DIR>/raw"
dotnet test "<ENTRY>" \
    --collect:"XPlat Code Coverage" \
    --results-directory "$raw_dir" \
    -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=cobertura \
    -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Include="[*]*" \
    -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Exclude="[*.Tests]*,[*.Test]*,[*Tests]*,[*Test]*,[*.Specs]*,[*.Testing]*" \
    -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.SkipAutoProps=true
```

Bash passes the same arguments as the PowerShell form (verified: the quoted glob-ish values such as `Include="[*]*"` stay literal and `--collect:"XPlat Code Coverage"` remains a single argument).

##### Windows (PowerShell)

```powershell
$rawDir = Join-Path "<COVERAGE_DIR>" "raw"
dotnet test "<ENTRY>" `
    --collect:"XPlat Code Coverage" `
    --results-directory $rawDir `
    -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=cobertura `
    -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Include="[*]*" `
    -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Exclude="[*.Tests]*,[*.Test]*,[*Tests]*,[*Test]*,[*.Specs]*,[*.Testing]*" `
    -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.SkipAutoProps=true
```

**Microsoft CodeCoverage** (`Microsoft.Testing.Extensions.CodeCoverage`):

The command syntax depends on the .NET SDK version. In .NET 9, Microsoft.Testing.Platform arguments
must be passed after the `--` separator. In .NET 10+, `--coverage` is a top-level `dotnet test` flag.

##### Linux (bash)

```bash
raw_dir="<COVERAGE_DIR>/raw"

# Detect SDK version for correct argument placement
sdk_version=$(dotnet --version 2>/dev/null)
major=$(printf '%s' "$sdk_version" | sed -n 's/^\([0-9]\{1,\}\)\..*/\1/p')
[ -n "$major" ] || major=9

if [ "$major" -ge 10 ]; then
    # .NET 10+: --coverage is a first-class dotnet test flag
    dotnet test "<ENTRY>" \
        --results-directory "$raw_dir" \
        --coverage \
        --coverage-output-format cobertura \
        --coverage-output "$raw_dir"
else
    # .NET 9: pass Microsoft.Testing.Platform arguments after the -- separator
    dotnet test "<ENTRY>" \
        --results-directory "$raw_dir" \
        -- --coverage --coverage-output-format cobertura --coverage-output "$raw_dir"
fi
```

##### Windows (PowerShell)

```powershell
$rawDir = Join-Path "<COVERAGE_DIR>" "raw"

# Detect SDK version for correct argument placement
$sdkVersion = (dotnet --version 2>$null)
$major = if ($sdkVersion -match '^(\d+)\.') { [int]$Matches[1] } else { 9 }

if ($major -ge 10) {
    # .NET 10+: --coverage is a first-class dotnet test flag
    dotnet test "<ENTRY>" `
        --results-directory $rawDir `
        --coverage `
        --coverage-output-format cobertura `
        --coverage-output $rawDir
} else {
    # .NET 9: pass Microsoft.Testing.Platform arguments after the -- separator
    dotnet test "<ENTRY>" `
        --results-directory $rawDir `
        -- --coverage --coverage-output-format cobertura --coverage-output $rawDir
}
```

**Mixed-project mode** (`Microsoft.Testing.Extensions.CodeCoverage` + `coverlet.collector` in the same solution):

##### Linux (bash)

```bash
raw_dir="<COVERAGE_DIR>/raw"
sdk_version=$(dotnet --version 2>/dev/null)
major=$(printf '%s' "$sdk_version" | sed -n 's/^\([0-9]\{1,\}\)\..*/\1/p')
[ -n "$major" ] || major=9

for tp in "${test_projects[@]}"; do
    if grep -q -E 'Microsoft\.Testing\.Extensions\.CodeCoverage' "$tp"; then
        if [ "$major" -ge 10 ]; then
            dotnet test "$tp" --results-directory "$raw_dir" --coverage --coverage-output-format cobertura --coverage-output "$raw_dir"
        else
            dotnet test "$tp" --results-directory "$raw_dir" -- --coverage --coverage-output-format cobertura --coverage-output "$raw_dir"
        fi
    else
        dotnet test "$tp" \
            --collect:"XPlat Code Coverage" \
            --results-directory "$raw_dir" \
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=cobertura \
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Include="[*]*" \
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Exclude="[*.Tests]*,[*.Test]*,[*Tests]*,[*Test]*,[*.Specs]*,[*.Testing]*" \
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.SkipAutoProps=true
    fi
done
```

##### Windows (PowerShell)

```powershell
$rawDir = Join-Path "<COVERAGE_DIR>" "raw"
$sdkVersion = (dotnet --version 2>$null)
$major = if ($sdkVersion -match '^(\d+)\.') { [int]$Matches[1] } else { 9 }

foreach ($tp in $testProjects) {
    $hasMsCodeCov = Select-String -Path $tp.FullName -Pattern 'Microsoft\.Testing\.Extensions\.CodeCoverage' -Quiet
    if ($hasMsCodeCov) {
        if ($major -ge 10) {
            dotnet test $tp.FullName --results-directory $rawDir --coverage --coverage-output-format cobertura --coverage-output $rawDir
        } else {
            dotnet test $tp.FullName --results-directory $rawDir -- --coverage --coverage-output-format cobertura --coverage-output $rawDir
        }
    } else {
        dotnet test $tp.FullName `
            --collect:"XPlat Code Coverage" `
            --results-directory $rawDir `
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=cobertura `
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Include="[*]*" `
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Exclude="[*.Tests]*,[*.Test]*,[*Tests]*,[*Test]*,[*.Specs]*,[*.Testing]*" `
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.SkipAutoProps=true
    }
}
```

Exit code handling:

- **0** — all tests passed, coverage collected
- **1** — some tests failed (coverage still collected — proceed with a warning)
- **Other** — build failure; stop and report the error

After the run, locate coverage files:

##### Linux (bash)

```bash
raw_dir="<COVERAGE_DIR>/raw"
mapfile -t cobertura_files < <(find "$raw_dir" -name 'coverage.cobertura.xml' -type f 2>/dev/null)
echo "COBERTURA_COUNT:${#cobertura_files[@]}"
for f in "${cobertura_files[@]}"; do echo "COBERTURA:$f"; done
mapfile -t vs_cov_files < <(find "$raw_dir" -name '*.coverage' -type f 2>/dev/null)
if [ "${#vs_cov_files[@]}" -gt 0 ]; then echo "VS_BINARY_COVERAGE:${#vs_cov_files[@]}"; fi
```

##### Windows (PowerShell)

```powershell
$coberturaFiles = Get-ChildItem -Path (Join-Path "<COVERAGE_DIR>" "raw") -Filter "coverage.cobertura.xml" -Recurse
Write-Host "COBERTURA_COUNT:$($coberturaFiles.Count)"
$coberturaFiles | ForEach-Object { Write-Host "COBERTURA:$($_.FullName)" }
$vsCovFiles = Get-ChildItem -Path (Join-Path "<COVERAGE_DIR>" "raw") -Filter "*.coverage" -Recurse -ErrorAction SilentlyContinue
if ($vsCovFiles) { Write-Host "VS_BINARY_COVERAGE:$($vsCovFiles.Count)" }
```

If `COBERTURA_COUNT` is 0:

- If `VS_BINARY_COVERAGE` > 0: warn the user — *"Found .coverage files (VS binary format) but no Cobertura XML. These were likely produced by Visual Studio's built-in collector, which outputs a binary format by default. This skill needs Cobertura XML. Re-running with the detected provider configured for Cobertura output."* Then re-run the appropriate `dotnet test` command above (Coverlet or Microsoft CodeCoverage) with Cobertura format.
- If no `.coverage` files either: stop and report — *"Coverage files not generated. Ensure `dotnet test` completed successfully and check the build output for errors."*

##### Linux (bash) — reading `.coverage` binaries

Visual Studio's built-in collector is Windows-only, but the `.coverage` binary format can be converted to Cobertura on Linux with the cross-platform `dotnet-coverage` tool (verified on this machine: `dotnet tool install dotnet-coverage` succeeds and `merge -f cobertura` is a supported output format):

```bash
mapfile -t cov_inputs < <(find "<COVERAGE_DIR>/raw" -name '*.coverage' -type f)
if [ "${#cov_inputs[@]}" -gt 0 ]; then
    dotnet tool install dotnet-coverage --tool-path "<COVERAGE_DIR>/.tools"
    "<COVERAGE_DIR>/.tools/dotnet-coverage" merge "${cov_inputs[@]}" \
        -f cobertura -o "<COVERAGE_DIR>/raw/merged.cobertura.xml"
fi
```

待验证 (unverified): only tool installation and the `-f cobertura` output format were verified here — the full `.coverage` → Cobertura round-trip was not exercised (no `.coverage` sample was available on this machine). If the merge fails, fall back to re-running `dotnet test` with Coverlet or Microsoft CodeCoverage configured for Cobertura output, as described above.

### Phase 3 — Analysis (sequential)

Run the two bundled PowerShell scripts. Both are cheap and complete in seconds. On Linux the `.ps1` files cannot run (`pwsh` is not installed on this machine) — use the `python3` equivalents given in Steps 4 and 5, which emit the same output lines. **Do not** install or invoke ReportGenerator here — that belongs in optional Phase 5, after the user-facing summary has been delivered.

#### Step 4: Calculate CRAP scores using the bundled script

Run `scripts/Compute-CrapScores.ps1` (co-located with this SKILL.md). It reads all Cobertura XML files, applies `CRAP(m) = comp² × (1 − cov)³ + comp` per method, and returns the top-N hotspots as JSON.

To locate the script: find the directory containing this skill's `SKILL.md` file (the skill loader provides this context), then resolve `scripts/Compute-CrapScores.ps1` relative to it. If the script path cannot be determined, calculate CRAP scores inline using the formula below.

##### Linux (bash)

`pwsh` is not installed on this machine, so the bundled `.ps1` cannot be executed here. This self-contained `python3` equivalent reads the same Cobertura XML, applies the same merge/CRAP logic, and prints the same `OVERALL_*`, `TOTAL_METHODS`, `FLAGGED_METHODS`, and `HOTSPOTS:` lines (verified against a Cobertura fixture on this machine):

```bash
# Substitute the <...> placeholders before running (they are quoted so bash parses them as arguments).
# $cobertura_files comes from Step 2b / Step 3; you can also list paths explicitly:
#   python3 - 30 10 path/a.cobertura.xml path/b.cobertura.xml
python3 - "<crap_threshold>" "<top_n>" "${cobertura_files[@]}" <<'PY'
import json
import sys
import xml.etree.ElementTree as ET

crap_threshold = int(sys.argv[1])
top_n = int(sys.argv[2])
paths = sys.argv[3:]

methods = {}
lines_covered = lines_valid = 0
branches_covered = branches_valid = 0
fallback_line_rates, fallback_branch_rates = [], []

for path in paths:
    try:
        tree = ET.parse(path)
    except (OSError, ET.ParseError) as exc:
        print("Failed to parse Cobertura XML: %s: %s" % (path, exc), file=sys.stderr)
        sys.exit(2)
    root = tree.getroot()
    cov = root if root.tag == "coverage" else root.find("coverage")
    if cov is None:
        print("Not a Cobertura document: %s" % path, file=sys.stderr)
        sys.exit(2)

    lc, lv = cov.get("lines-covered"), cov.get("lines-valid")
    bc, bv = cov.get("branches-covered"), cov.get("branches-valid")
    if lc is not None and lv is not None:
        lines_covered += int(lc)
        lines_valid += int(lv)
    elif cov.get("line-rate") is not None:
        fallback_line_rates.append(float(cov.get("line-rate")))
    if bc is not None and bv is not None:
        branches_covered += int(bc)
        branches_valid += int(bv)
    elif cov.get("branch-rate") is not None:
        fallback_branch_rates.append(float(cov.get("branch-rate")))

    for pkg in cov.findall("./packages/package"):
        for cls in pkg.findall("./classes/class"):
            for meth in cls.findall("./methods/method"):
                key = "|".join((cls.get("name", ""), meth.get("name", ""),
                                meth.get("signature", ""), cls.get("filename", "")))
                # Cyclomatic complexity is stored as an XML attribute in Cobertura format
                complexity = max(int(meth.get("complexity") or 1), 1)
                entry = methods.setdefault(key, {
                    "Class": cls.get("name", ""), "Method": meth.get("name", ""),
                    "Signature": meth.get("signature", ""), "File": cls.get("filename", ""),
                    "Complexity": complexity, "LineHits": {}})
                # Accumulate hit counts per line number across files
                for line in meth.findall("./lines/line"):
                    num = line.get("number")
                    entry["LineHits"][num] = entry["LineHits"].get(num, 0) + int(line.get("hits") or 0)

results = []
for entry in methods.values():
    total_lines = len(entry["LineHits"])
    covered_lines = sum(1 for hits in entry["LineHits"].values() if hits > 0)
    line_coverage = (covered_lines / total_lines) if total_lines else 0.0

    # Alberto Savoia's CRAP formula: comp^2 * (1 - cov)^3 + comp
    # The cubic exponent on (1-cov) sharply penalizes low coverage:
    # at 0% coverage the risk multiplier is 1.0; at 50% it drops to 0.125.
    # Higher scores = more complex AND less covered = riskier to change
    uncovered = 1.0 - line_coverage
    crap_score = round(entry["Complexity"] ** 2 * uncovered ** 3 + entry["Complexity"], 2)

    results.append({
        "Class": entry["Class"], "Method": entry["Method"], "Signature": entry["Signature"],
        "File": entry["File"], "TotalLines": total_lines, "CoveredLines": covered_lines,
        "LineCoverage": round(line_coverage * 100, 1), "Complexity": entry["Complexity"],
        "CrapScore": crap_score})

if lines_valid > 0:
    overall_line_rate = lines_covered / lines_valid
else:
    # Fallback approximation when Cobertura aggregate counters and per-file rates are unavailable.
    # This uses merged method line totals and may under/over-estimate if Cobertura
    # includes executable lines outside method nodes.
    merged_total_lines = sum(r["TotalLines"] for r in results)
    merged_covered_lines = sum(r["CoveredLines"] for r in results)
    if merged_total_lines > 0:
        overall_line_rate = merged_covered_lines / merged_total_lines
    elif fallback_line_rates:
        overall_line_rate = sum(fallback_line_rates) / len(fallback_line_rates)
    else:
        overall_line_rate = 0.0

if branches_valid > 0:
    overall_branch_rate = branches_covered / branches_valid
elif fallback_branch_rates:
    overall_branch_rate = sum(fallback_branch_rates) / len(fallback_branch_rates)
else:
    overall_branch_rate = 0.0

hotspots = sorted(results, key=lambda r: r["CrapScore"], reverse=True)[:top_n]
flagged = [r for r in results if r["CrapScore"] > crap_threshold]

print("OVERALL_LINE_COVERAGE:%s" % round(overall_line_rate * 100, 1))
print("OVERALL_BRANCH_COVERAGE:%s" % round(overall_branch_rate * 100, 1))
print("TOTAL_METHODS:%d" % len(results))
print("FLAGGED_METHODS:%d" % len(flagged))
print("HOTSPOTS:%s" % (json.dumps(hotspots, separators=(",", ":")) if hotspots else "[]"))
PY
```

##### Windows (PowerShell)

```powershell
& "<skill-directory>/scripts/Compute-CrapScores.ps1" `
    -CoberturaPath @(<all COBERTURA file paths as array>) `
    -CrapThreshold <crap_threshold> `
    -TopN <top_n>
```

Script outputs: `OVERALL_LINE_COVERAGE:<n>`, `OVERALL_BRANCH_COVERAGE:<n>` (aggregated project-wide rates across all provided Cobertura files), `TOTAL_METHODS:<n>`, `FLAGGED_METHODS:<n>`, `HOTSPOTS:<json>` (top-N sorted by CrapScore descending). The OVERALL_* values are exactly what the Phase 4 summary needs for the "Line Coverage" / "Branch Coverage" rows — no separate XML parsing tool call is required.

#### Step 5: Extract per-method coverage gaps

Run `scripts/Extract-MethodCoverage.ps1` to get per-method coverage data for the Coverage Gaps table:

##### Linux (bash)

`python3` equivalent of the bundled script — same per-method metrics (line/branch coverage, complexity, uncovered lines), same filters, same JSON-to-stdout contract (verified against Cobertura fixtures, including multi-file merging):

```bash
# Join the Cobertura paths into the single colon-separated argument the script expects
cobertura_paths=$(IFS=:; printf '%s' "${cobertura_files[*]}")
python3 - "$cobertura_paths" "<line_threshold>" "<branch_threshold>" below-threshold <<'PY'
import json
import re
import sys
import xml.etree.ElementTree as ET

paths = sys.argv[1]
coverage_threshold = int(sys.argv[2])
branch_threshold = int(sys.argv[3])
mode = sys.argv[4]  # uncovered | below-threshold | all

for path in paths.split(":"):
    if not path:
        continue
    try:
        with open(path, "rb"):
            pass
    except OSError:
        print("Cobertura file not found: %s" % path, file=sys.stderr)
        sys.exit(2)

# Merge methods across all Cobertura files using a stable key (Class|Method|Signature|File).
# Line hits and branch data are accumulated so coverage reflects all test projects.
methods = {}

for path in paths.split(":"):
    if not path:
        continue
    try:
        root = ET.parse(path).getroot()
    except (OSError, ET.ParseError) as exc:
        print("Failed to parse Cobertura XML: %s: %s" % (path, exc), file=sys.stderr)
        sys.exit(2)

    for pkg in root.findall("./packages/package"):
        for cls in pkg.findall("./classes/class"):
            for meth in cls.findall("./methods/method"):
                key = "|".join((cls.get("name", ""), meth.get("name", ""),
                                meth.get("signature", ""), cls.get("filename", "")))
                entry = methods.setdefault(key, {
                    "Class": cls.get("name", ""), "Method": meth.get("name", ""),
                    "Signature": meth.get("signature", ""), "File": cls.get("filename", ""),
                    "Complexity": max(int(meth.get("complexity") or 1), 1),
                    "LineHits": {}, "BranchData": {}})

                for line in meth.findall("./lines/line"):
                    num = line.get("number")
                    entry["LineHits"][num] = entry["LineHits"].get(num, 0) + int(line.get("hits") or 0)

                    if line.get("branch") == "true" and line.get("condition-coverage"):
                        m = re.search(r"\((\d+)/(\d+)\)", line.get("condition-coverage"))
                        if m:
                            covered, total = int(m.group(1)), int(m.group(2))
                            if num in entry["BranchData"]:
                                # Merge branch coverage across files by accumulating covered branches (capped at total)
                                prev = entry["BranchData"][num]
                                if prev["Total"] != total:
                                    print("Branch total mismatch for %s at line %s: %s vs %s"
                                          % (key, num, prev["Total"], total), file=sys.stderr)
                                merged_total = max(prev["Total"], total)
                                merged_covered = min(prev["Covered"] + covered, merged_total)
                                entry["BranchData"][num] = {"Covered": merged_covered, "Total": merged_total}
                            else:
                                entry["BranchData"][num] = {"Covered": covered, "Total": total}

rows = []
for entry in methods.values():
    total_lines = len(entry["LineHits"])
    covered_line_count = sum(1 for hits in entry["LineHits"].values() if hits > 0)
    line_coverage_percent = round((covered_line_count / total_lines) * 100, 1) if total_lines else 0

    branches_total = sum(b["Total"] for b in entry["BranchData"].values())
    branches_covered = sum(b["Covered"] for b in entry["BranchData"].values())
    branch_coverage_percent = round((branches_covered / branches_total) * 100, 1) if branches_total else 0

    # Apply filter
    if mode == "uncovered" and line_coverage_percent > 0:
        continue
    if mode == "below-threshold":
        line_ok = line_coverage_percent >= coverage_threshold
        branch_ok = (branches_total == 0) or (branch_coverage_percent >= branch_threshold)
        if line_ok and branch_ok:
            continue

    rows.append({
        "Class": entry["Class"], "Method": entry["Method"], "Signature": entry["Signature"],
        "File": entry["File"], "Complexity": entry["Complexity"],
        "LineCoverage": line_coverage_percent, "BranchCoverage": branch_coverage_percent,
        "CoveredLines": covered_line_count, "TotalLines": total_lines,
        "UncoveredLines": total_lines - covered_line_count,
        "CoveredBranches": branches_covered, "TotalBranches": branches_total})

# Sort by uncovered lines descending, then by line coverage ascending
rows.sort(key=lambda r: (-r["UncoveredLines"], r["LineCoverage"], r["Class"], r["Method"]))

print(json.dumps(rows, indent=2) if rows else "[]")
print("METHODS_FILTERED:%d" % len(rows))
print("UNCOVERED_METHODS:%d" % sum(1 for r in rows if r["LineCoverage"] == 0))
PY
```

##### Windows (PowerShell)

```powershell
& "<skill-directory>/scripts/Extract-MethodCoverage.ps1" `
    -CoberturaPath @(<all COBERTURA file paths as array>) `
    -CoverageThreshold <line_threshold> `
    -BranchThreshold <branch_threshold> `
    -Filter below-threshold
```

Script outputs: JSON array of methods below the coverage threshold, sorted by coverage ascending. Use this data to populate the Coverage Gaps by File table in the report.

### Phase 4 — User-facing summary (MANDATORY — your next assistant response)

As soon as Phase 3 completes, **your immediately next assistant response must contain the user-facing analysis** — do not interleave any other tool calls before it. This is the response the user (and any judge) sees. Skipping or deferring this in favor of Phase 5 (ReportGenerator) is a hard failure.

The response must include, at minimum:

1. Overall line and branch coverage — read directly from the `OVERALL_LINE_COVERAGE:` / `OVERALL_BRANCH_COVERAGE:` lines emitted by `Compute-CrapScores.ps1` (no extra Cobertura parsing required)
2. The Risk Hotspots table built from `Compute-CrapScores.ps1` `HOTSPOTS:` output (CRAP scores, complexity, coverage)
3. Identification of the highest-risk method(s) and what is blocking coverage
4. 1–3 prioritized, specific recommendations (which method to test, expected CRAP/coverage impact)

Use `references/output-format.md` verbatim for fixed headings, table structures, symbols, and emoji. Use `references/guidelines.md` for prioritization rules and style.

If Phase 5 has not yet run when you compose this summary, mark the `## 📁 Reports` section's HTML/Text/CSV/GitHub-markdown rows as `Not generated (optional — request HTML reports to enable)`. Only the `coverage-analysis.md` and raw Cobertura paths are guaranteed to exist.

Attempt to save the same content to `TestResults/coverage-analysis/coverage-analysis.md` before delivering the response (use the editor's create/edit tool — do not shell out). If the file write fails, still deliver the summary and note the file-write failure explicitly.

### Phase 5 — Optional: ReportGenerator HTML/CSV reports (post-summary)

Phase 5 is **strictly optional** and runs **only after** Phase 4 has been delivered. Skip Phase 5 entirely when:

- The user supplied existing Cobertura XML and only asked for analysis (the default for the existing-data path).
- The user is diagnosing a coverage plateau or asking "what's blocking me?" — they want the answer, not a static-site report.
- ReportGenerator is not already installed and you have no clear signal the user wants HTML reports.

Run Phase 5 only when the user explicitly asks for HTML/CSV reports, or when the project flow requires them (e.g., a CI artifact upload step).

#### Step 6: Verify or install ReportGenerator (only if running Phase 5)

##### Linux (bash)

`dotnet tool install` and the `reportgenerator` tool are cross-platform, so this runs unchanged on Linux — only the PATH handling differs (`export PATH=...` instead of `$env:PATH`):

```bash
rg_available=false
if command -v reportgenerator >/dev/null 2>&1; then
    rg_available=true
    echo "RG_INSTALLED:already-present"
else
    rg_tool_path="<COVERAGE_DIR>/.tools"
    if dotnet tool install dotnet-reportgenerator-globaltool --tool-path "$rg_tool_path"; then
        export PATH="$rg_tool_path:$PATH"
        if command -v reportgenerator >/dev/null 2>&1; then
            rg_available=true
            echo "RG_INSTALLED:true (tool-path: $rg_tool_path)"
        else
            echo "RG_INSTALLED:false"
            echo "RG_INSTALL_ERROR:reportgenerator-not-available"
        fi
    else
        echo "RG_INSTALLED:false"
        echo "RG_INSTALL_ERROR:reportgenerator-not-available"
    fi
fi
echo "RG_AVAILABLE:$rg_available"
```

##### Windows (PowerShell)

```powershell
$rgAvailable = $false
$rgCommand = Get-Command reportgenerator -ErrorAction SilentlyContinue
if ($rgCommand) {
    $rgAvailable = $true
    Write-Host "RG_INSTALLED:already-present"
} else {
    $rgToolPath = Join-Path "<COVERAGE_DIR>" ".tools"
    dotnet tool install dotnet-reportgenerator-globaltool --tool-path $rgToolPath
    if ($LASTEXITCODE -eq 0) {
        $env:PATH = "$rgToolPath$([System.IO.Path]::PathSeparator)$env:PATH"
        $rgCommand = Get-Command reportgenerator -ErrorAction SilentlyContinue
        if ($rgCommand) {
            $rgAvailable = $true
            Write-Host "RG_INSTALLED:true (tool-path: $rgToolPath)"
        } else {
            Write-Host "RG_INSTALLED:false"
            Write-Host "RG_INSTALL_ERROR:reportgenerator-not-available"
        }
    } else {
        Write-Host "RG_INSTALLED:false"
        Write-Host "RG_INSTALL_ERROR:reportgenerator-not-available"
    }
}
Write-Host "RG_AVAILABLE:$rgAvailable"
```

If installation fails (no internet), keep `RG_AVAILABLE:false`, leave the existing user-facing summary as the final output, and note that HTML reports were skipped.

#### Step 7: Generate HTML/CSV reports

##### Linux (bash)

The `;` inside `-reporttypes:` / `-reports:` is ReportGenerator's own list separator, not a shell separator — it must stay quoted in bash:

```bash
reports_dir="<COVERAGE_DIR>/reports"
if [ "$rg_available" = true ]; then
    reportgenerator \
        -reports:"<semicolon-separated COBERTURA paths>" \
        -targetdir:"$reports_dir" \
        -reporttypes:"Html;TextSummary;MarkdownSummaryGithub;CsvSummary" \
        -title:"Coverage Report" \
        -tag:"coverage-analysis-skill"

    cat "$reports_dir/Summary.txt" 2>/dev/null
else
    echo "REPORTGENERATOR_SKIPPED:true"
fi
```

##### Windows (PowerShell)

```powershell
$reportsDir = Join-Path "<COVERAGE_DIR>" "reports"
if ($rgAvailable) {
    reportgenerator `
        -reports:"<semicolon-separated COBERTURA paths>" `
        -targetdir:$reportsDir `
        -reporttypes:"Html;TextSummary;MarkdownSummaryGithub;CsvSummary" `
        -title:"Coverage Report" `
        -tag:"coverage-analysis-skill"

    Get-Content (Join-Path $reportsDir "Summary.txt") -ErrorAction SilentlyContinue
} else {
    Write-Host "REPORTGENERATOR_SKIPPED:true"
}
```

After Phase 5 completes successfully, you may follow up with a short message pointing the user to the generated HTML report (one paragraph, no need to repeat the summary).

## Validation

- Verify that at least one `coverage.cobertura.xml` file was generated after `dotnet test` (or already exists when the user supplied one)
- Confirm the assistant response contained the CRAP/risk-hotspot table — saving the markdown file is secondary
- Confirm `TestResults/coverage-analysis/coverage-analysis.md` was written and contains data
- Spot-check one method's CRAP score: `comp² × (1 − cov)³ + comp` — a method with 100% coverage should have CRAP = complexity
- If Phase 5 ran, verify `TestResults/coverage-analysis/reports/index.html` exists; otherwise the report file should mark HTML/Text/CSV rows as `Not generated`

## Common Pitfalls

- **No Cobertura XML generated** — the test project may lack a coverage provider. The skill auto-adds one, but if `dotnet add package` fails (offline/proxy), coverage collection silently produces nothing. Check for `.coverage` binary files as a fallback indicator.
- **Test failures (exit code 1)** — coverage is still collected from passing tests. Do not abort; proceed with partial data and note the failures in the summary.
- **Premature end before user-facing summary** — never start Phase 5 (ReportGenerator install/run) before the Phase 4 assistant response is delivered. The heavy `dotnet tool install` can crash the session or exhaust budget, leaving the user with no analysis even though the CRAP scores were already computed.
- **ReportGenerator install failure** — if `dotnet tool install` fails (no internet) during Phase 5, leave the existing Phase 4 summary as the final output and note that HTML reports were skipped. Do not retry or block on the install.
- **Method name mismatches in Cobertura** — async methods, lambdas, and local functions may have compiler-generated names. The scripts use the Cobertura method name/signature directly; verify against source if results look unexpected.
- **Mixed coverage providers** — when a solution contains both Coverlet and Microsoft CodeCoverage projects, the skill runs per-project to avoid dual-provider conflicts. This is slower but correct.
- **`pwsh` is not installed on this Linux machine** — the bundled `scripts/*.ps1` helpers (Steps 4 and 5) cannot run as-is. Use the `python3` equivalents in this file; they were verified here against Cobertura fixtures and emit the same `OVERALL_*` / `HOTSPOTS:` / method-JSON output, so the rest of the workflow is unchanged.
- **Bash argument quoting** — the `dotnet test` coverage arguments (Step 3) rely on per-argument quoting so the shell does not glob or split values such as `--collect:"XPlat Code Coverage"` and `Include="[*]*"`. Keep the quotes exactly as written when editing the commands.
