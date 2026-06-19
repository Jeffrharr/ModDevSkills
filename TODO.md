# TODO

## Expand /setup-mod-ci

The current skill covers git, GitHub, and GitHub Actions. It should also wire up
the tooling below as part of the same setup flow, or as composable sub-steps that
`/setup-mod-ci` calls into.

---

## Standard C# tooling

These are well-established in the .NET ecosystem and should be part of every mod's
baseline regardless of RimWorld specifics.

### .editorconfig
A root `.editorconfig` enforcing consistent formatting: indentation, line endings,
trailing whitespace, charset. Most RimWorld mod code uses tabs/4-space indentation.
Should be committed to the repo and respected by `dotnet format`.

### dotnet format
Built into the .NET SDK — no install needed. Enforces `.editorconfig` rules and
runs Roslyn analyzer fixes. Add as a CI step (`dotnet format --verify-no-changes`)
so formatting drift fails the build.

### Roslyn analyzers (via NuGet)
Add to the mod's `.csproj` as development-time analyzer packages:
- **Microsoft.CodeAnalysis.NetAnalyzers** — the default .NET quality rules (ships
  with the SDK but pinning the NuGet gives version control)
- **Roslynator.Analyzers** — broad quality and simplification rules, actively
  maintained, good signal-to-noise ratio for mod-sized codebases
- **Microsoft.Unity.Analyzers** — Unity-specific rules; catches common Unity/Mono
  pitfalls that affect RimWorld mods (coroutine misuse, expensive Update patterns)

Configure severity in `.editorconfig` rather than in the csproj to keep it
editor-visible. Start permissive (suggestion/warning) and tighten over time.

### Directory.Build.props
A `Directory.Build.props` at the repo root that sets shared properties across all
projects (mod, tests): `Nullable`, `LangVersion`, `TreatWarningsAsErrors` for
analyzer rules, `NoWarn` for known-acceptable suppressions. Keeps individual
`.csproj` files clean.

---

## RimWorld-specific tooling

### harmony-check lint in CI
Currently the pre-commit hook runs `harmony-check lint` locally. It should also run
in the CI workflow so PRs from contributors get the same check. The `harmony-check`
binary needs to either be built from source in CI or published as a .NET tool
(`dotnet tool install`). Publishing as a dotnet global tool is the cleanest path —
add to TODO for HarmonyConflictChecker.

### API compatibility test scaffold
Every mod should have a small test project using Mono.Cecil that verifies the
RimWorld types and methods the mod patches still exist after a game update.
See `PerformanceSearch/Tests` for the reference pattern.
The `/setup-mod-ci` skill should detect whether this test project exists and offer
to scaffold it if not, extracting the patched types from the mod's `[HarmonyPatch]`
attributes automatically.

### Krafs.Rimworld.Ref version pinning
The mod's `.csproj` references `Krafs.Rimworld.Ref` for RimWorld assemblies. The
skill should check that the version is pinned to `1.6.*` (or current) and warn if
it's using a wildcard that could silently pick up a breaking update.

### Hot reload support (HotSwappable)
Some mods use the `[HotSwappable]` attribute for in-game code reloading during
development. The skill could offer to add this as an optional dev dependency with
a note on when it's useful.

---

## New skills to build

### /setup-linting
Focused skill that adds `.editorconfig`, `dotnet format` CI step, and Roslyn
analyzer packages to an existing mod project. Composable — called by
`/setup-mod-ci` but also useful standalone for existing mods.

### /setup-api-compat-tests
Scaffolds the Cecil-based API compatibility test project. Reads the mod's
`[HarmonyPatch]` attributes to infer which RimWorld types to verify, generates
the test stubs, and wires them into CI.

### /analyze-performance
Agentic skill: decompiles the mod DLL, reads the hot paths (GUI methods, tick
methods), and reasons about per-frame or per-tick work that should be cached.
Produces a report of findings with suggested fixes. See the BetterArchitect
floors-tab fix as a reference example of what this should catch automatically.
