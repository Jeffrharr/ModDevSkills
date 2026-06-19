# /setup-linting

Add C# linting to an existing mod project: `.editorconfig`, Roslyn analyzer
package, and a `dotnet format` CI step. Works standalone or as a follow-up to
`/setup-mod-project`.

## Step 1 — Choose an analyzer package

Present the user with these options and let them decide before writing any files:

**Option A: StyleCop.Analyzers**
- Enforces formatting and style conventions: naming, spacing, brace placement,
  documentation headers
- Narrow focus — few false positives, low noise
- Good default for teams that just want consistent formatting
- ~230 rules, all style-oriented

**Option B: Roslynator.Analyzers**
- Broad quality + style rules: simplification, performance hints, potential bugs
- More opinionated — expect to suppress a handful of rules that don't fit
- Good default for authors who want a thorough second opinion on their code
- 500+ rules across style and correctness

**Option C: Both**
- StyleCop handles style, Roslynator handles quality — they don't overlap much
- Most comprehensive, but requires more initial configuration to tune noise
- Worth it for larger or longer-lived mods

**Option D: Built-in SDK analyzers only**
- No NuGet package needed — already active in .NET 5+ projects
- Covers the most critical correctness rules (nullable, unused vars, API misuse)
- Lowest friction; good for authors who want minimal setup

Ask the user which they'd like before continuing.

---

## Step 2 — Add the NuGet package(s)

Skip if Option D was chosen.

Add to `Source/<ModName>.csproj` inside `<ItemGroup>`:

**StyleCop:**
```xml
<PackageReference Include="StyleCop.Analyzers" Version="1.2.*">
  <PrivateAssets>all</PrivateAssets>
  <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
</PackageReference>
```

**Roslynator:**
```xml
<PackageReference Include="Roslynator.Analyzers" Version="4.*">
  <PrivateAssets>all</PrivateAssets>
  <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
</PackageReference>
```

Explain: `PrivateAssets=all` keeps the analyzer package out of the mod's published
output — it's a dev tool only. Without this, NuGet would try to copy the analyzer
DLL into `1.6/Assemblies/` alongside the mod DLL.

---

## Step 3 — Add .editorconfig

Create `.editorconfig` at the repo root. Start with a base that works for all
options, then add the relevant analyzer sections.

### Base (always include)

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 4
trim_trailing_whitespace = true
insert_final_newline = true

[*.cs]
# .NET / Roslyn built-in rules
dotnet_diagnostic.IDE0001.severity = warning  # simplify type name
dotnet_diagnostic.IDE0003.severity = warning  # remove this. qualification
dotnet_diagnostic.IDE0005.severity = warning  # remove unused usings
dotnet_diagnostic.IDE0051.severity = warning  # remove unused private member
dotnet_diagnostic.IDE0052.severity = warning  # remove unread private member
dotnet_diagnostic.CS8600.severity = warning   # nullable: converting null literal
dotnet_diagnostic.CS8602.severity = warning   # nullable: dereference of possibly null
dotnet_diagnostic.CS8603.severity = warning   # nullable: possible null return

[*.{csproj,props,targets}]
indent_size = 2
```

### StyleCop additions (Option A or C)

```ini
[*.cs]
# Disable documentation rules — mod code doesn't need XML doc comments
dotnet_diagnostic.SA1600.severity = none  # elements must be documented
dotnet_diagnostic.SA1601.severity = none  # partial elements must be documented
dotnet_diagnostic.SA1602.severity = none  # enum items must be documented
dotnet_diagnostic.SA1633.severity = none  # file header
dotnet_diagnostic.SA1652.severity = none  # enable XML documentation

# Disable file/namespace layout rules that conflict with typical mod structure
dotnet_diagnostic.SA1200.severity = none  # using directives inside namespace
```

Explain: StyleCop's documentation rules assume library code with public XML docs.
Mod code is typically internal — these rules add noise without value and should
be turned off up front.

### Roslynator additions (Option B or C)

```ini
[*.cs]
# Rules that commonly conflict with RimWorld/Harmony mod patterns
dotnet_diagnostic.RCS1090.severity = none  # add ConfigureAwait — not applicable (no async in mods)
dotnet_diagnostic.RCS1058.severity = none  # use compound assignment — sometimes less readable
```

---

## Step 4 — Directory.Build.props: enable analysis in CI

Check if `Directory.Build.props` already exists (it should if `/setup-mod-project`
was run). Add or merge these properties:

```xml
<PropertyGroup>
  <EnableNETAnalyzers>true</EnableNETAnalyzers>
  <AnalysisMode>Recommended</AnalysisMode>
  <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
</PropertyGroup>
```

Explain:
- `EnableNETAnalyzers` — turns on the built-in SDK analyzers explicitly (they're
  on by default in newer SDK versions, but explicit is better)
- `AnalysisMode=Recommended` — enables a curated subset of rules; `All` exists
  but is very noisy
- `EnforceCodeStyleInBuild` — makes IDE style rules (IDE0xxx) fail `dotnet build`,
  not just show squiggles in the editor; needed for CI to catch style violations

---

## Step 5 — Add dotnet format to CI

Check if `.github/workflows/ci.yml` exists. If yes, add a format check step after
the build step:

```yaml
- name: Check formatting
  run: dotnet format --verify-no-changes Source/<ModName>.csproj
```

Explain: `--verify-no-changes` makes `dotnet format` exit non-zero if it would
change any files, turning formatting drift into a CI failure. This keeps the style
rules meaningful — authors see failures on PRs rather than accumulating drift.

If there's no CI workflow yet, note that `/setup-mod-ci` will add one and that
the format check should be added at that point.

---

## Step 6 — First run

Run `dotnet format Source/<ModName>.csproj` to auto-fix any existing violations
before the first build. This avoids a wall of warnings on the first CI run from
pre-existing code.

Then build to confirm the analyzer configuration is valid:

```bash
./build.sh
```

Report how many warnings were produced. If it's more than ~20, suggest reviewing
the `.editorconfig` suppressions with the user before committing.

---

## Step 7 — Commit

```bash
git add .editorconfig Directory.Build.props Source/<ModName>.csproj .github/
git commit -m "Add C# linting: <chosen analyzer(s)> + dotnet format CI check"
```
