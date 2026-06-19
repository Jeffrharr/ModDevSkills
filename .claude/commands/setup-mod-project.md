# /setup-mod-project

Scaffold a new RimWorld mod project from scratch, or bring an existing one up to
the standard structure. Works on Linux and Windows. Explains the reasoning behind
every configuration decision so the author understands what they're looking at.

## What this does

- Creates the standard mod directory layout
- Writes `About/About.xml` with correct field formatting
- Creates the C# project targeting `net481` with the right dependencies
- Adds `Directory.Build.props` to handle cross-platform build differences
- Writes cross-platform `build.sh` (Linux/Mac) and `build.ps1` (Windows)
- Writes `test.sh` / `test.ps1` if a test project exists or is being created
- Adds a `DESIGN.md` stub
- Symlinks the mod into the RimWorld Mods folder
- Optionally modifies an existing project to be cross-platform compatible

## Detect mode: new vs existing

Check whether the current directory already has a `.csproj` or `About/About.xml`.
If yes, run in **retrofit mode**: audit what's there, identify what's missing or
non-portable, and make targeted changes. If no, run in **new project mode** from
the top.

---

## New project mode

### Step 1 — Gather info

Ask the user (or infer from context) for:
- Mod name (PascalCase, e.g. `PerformanceSearch`)
- Author name / handle (used in packageId and About.xml)
- One-sentence description
- Whether they need a test project

### Step 2 — Directory structure

```
ModName/
  About/
    About.xml
  Source/
    ModName.csproj
    ModNameMod.cs          ← optional stub
  1.6/
    Assemblies/            ← build output (gitignored)
  DESIGN.md
  build.sh
  build.ps1
  .gitignore
```

Explain to the user: RimWorld expects `About/About.xml` at the mod root and
DLLs in `<version>/Assemblies/`. The `Source/` directory keeps C# separate from
the mod files that ship to players. The `1.6/` prefix scopes assets and assemblies
to that game version — RimWorld loads the most specific version folder it finds.

### Step 3 — About.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<ModMetaData>
  <name>ModName</name>
  <author>AuthorHandle</author>
  <packageId>authorhandle.modname</packageId>
  <supportedVersions>
    <li>1.6</li>
  </supportedVersions>
  <description>One-sentence description.</description>
</ModMetaData>
```

Explain: `packageId` must be globally unique and in `author.modname` dot-notation,
lowercase only. RimWorld uses it to track the mod across updates and to resolve
cross-mod dependencies. A malformed ID (no dot, uppercase, spaces) causes a silent
load failure.

### Step 4 — C# project file

`Source/ModName.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net481</TargetFramework>
    <AssemblyName>ModName</AssemblyName>
    <RootNamespace>ModName</RootNamespace>
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <OutputPath>../1.6/Assemblies/</OutputPath>
    <AppendTargetFrameworkToOutputPath>false</AppendTargetFrameworkToOutputPath>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Krafs.Rimworld.Ref" Version="1.6.*" />
    <PackageReference Include="Lib.Harmony" Version="2.3.*" />
  </ItemGroup>
</Project>
```

Explain each decision:
- **`net481`**: RimWorld runs on Mono, a .NET Framework 4.x implementation.
  Targeting `net481` tells the compiler to use the right API surface. Using
  `net6`/`net8` would compile fine but produce a DLL that crashes on load because
  Mono doesn't implement the modern runtime.
- **`Krafs.Rimworld.Ref`**: Provides RimWorld's assemblies as NuGet references so
  the project compiles without needing the game installed. The `*` patch wildcard
  tracks minor updates; pin to a specific version once you've tested against it.
- **`Lib.Harmony`**: The patching library virtually all mods use. Included here as
  a build reference; RimWorld loads the actual Harmony DLL at runtime.
- **`OutputPath` → `1.6/Assemblies/`**: Puts the compiled DLL exactly where
  RimWorld expects it, so you can load the game immediately after building without
  a copy step.

### Step 5 — Directory.Build.props (cross-platform key)

Create `Directory.Build.props` at the repo root:

```xml
<Project>
  <!--
    On Linux, the .NET SDK doesn't know where Mono's net481 framework assemblies
    live. FrameworkPathOverride tells MSBuild to look in the system Mono install.
    On Windows this isn't needed — the SDK finds net481 automatically via the
    installed .NET Framework. The MSBuild condition keeps the override Linux-only.
  -->
  <PropertyGroup Condition="$([MSBuild]::IsOSPlatform('Linux'))">
    <FrameworkPathOverride>/usr/lib/mono/4.8-api</FrameworkPathOverride>
  </PropertyGroup>
</Project>
```

Explain: without this, `dotnet build` on Linux fails with "unable to find framework
net481" even though Mono is installed. On Windows it's a no-op. This replaces the
need to prefix every build command with `FrameworkPathOverride=...` in the shell —
which is the manual workaround most Linux modding guides tell you to use.

### Step 6 — Build scripts

`build.sh` (Linux/Mac):
```bash
#!/bin/bash
set -e
cd "$(dirname "$0")/Source"
dotnet build -c Release "$@"
```

`build.ps1` (Windows):
```powershell
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'
Push-Location "$PSScriptRoot/Source"
try { dotnet build -c Release @args }
finally { Pop-Location }
```

Explain: both scripts are thin wrappers around `dotnet build`. The real
cross-platform work is done in `Directory.Build.props` — the scripts just handle
directory navigation and error propagation idiomatically per platform. The `"$@"`
/ `@args` forwarding lets callers pass extra flags (e.g. `./build.sh -v d` for
verbose output).

Make `build.sh` executable: `chmod +x build.sh`.

### Step 7 — Test scripts (if test project exists)

`test.sh`:
```bash
#!/bin/bash
set -e
cd "$(dirname "$0")"
./build.sh
dotnet test Tests/ModName.Tests/ModName.Tests.csproj "$@"
```

`test.ps1`:
```powershell
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'
& "$PSScriptRoot/build.ps1"
dotnet test "$PSScriptRoot/Tests/ModName.Tests/ModName.Tests.csproj" @args
```

Make `test.sh` executable.

### Step 8 — .gitignore

```
1.6/Assemblies/
Source/obj/
Source/bin/
Tests/**/obj/
Tests/**/bin/
```

Explain: compiled DLLs are build artifacts — they shouldn't be committed. They're
reproducible from source, and committing them causes noisy diffs every build. The
`1.6/Assemblies/` line is intentionally version-prefixed to match the output path.

### Step 9 — DESIGN.md stub

```markdown
# ModName — Design

## Purpose

TODO: describe what this mod does and why.

## Architecture

TODO: describe how it works.

## Harmony patches

TODO: list patched methods and explain why each patch is necessary.
```

### Step 10 — Symlink into RimWorld Mods folder

```bash
ln -s "$(pwd)" ~/.local/share/Steam/steamapps/common/RimWorld/Mods/ModName
```

On Windows (run as Administrator in PowerShell):
```powershell
New-Item -ItemType SymbolicLink `
  -Path "$env:PROGRAMFILES\Steam\steamapps\common\RimWorld\Mods\ModName" `
  -Target (Get-Location)
```

Explain: the symlink means you edit source in your dev directory and the game
always loads the current built output — no manual copying. Without it you'd need
to copy the DLL to the Mods folder after every build.

Detect the RimWorld install path automatically:
- Linux default: `~/.local/share/Steam/steamapps/common/RimWorld/Mods/`
- Windows default: `C:\Program Files (x86)\Steam\steamapps\common\RimWorld\Mods\`
- Warn if the path doesn't exist and ask the user to confirm.

---

## Retrofit mode

For existing projects, audit and fix these in order:

1. **`About.xml`** — check packageId format, required fields
2. **`TargetFramework`** — must be `net481`, not `net6`/`net8`/`netstandard`
3. **`Directory.Build.props`** — add if missing; this is usually why the project
   only builds on one OS
4. **`FrameworkPathOverride` in build scripts** — if present, explain that
   `Directory.Build.props` replaces it and remove it from the scripts
5. **`OutputPath`** — confirm it points to `<version>/Assemblies/`
6. **Build scripts** — add the missing platform's script if only one exists
7. **`.gitignore`** — add `Assemblies/` if compiled DLLs are tracked
8. **Symlink** — check if it exists, offer to create it

Report what was changed and what was already correct.
