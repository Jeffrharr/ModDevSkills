# /setup-mod-ci

Set up full CI for a RimWorld mod: pre-commit hook, GitHub repo, and GitHub Actions
for build/test and Steam Workshop publishing.

## What this does

1. Creates a git pre-commit hook that builds the mod and runs harmony-check lint
2. Creates a GitHub repo under the authenticated gh account
3. Adds a CI workflow (build + test on every push/PR)
4. Adds a publish workflow (manual trigger → build + test + upload to Steam Workshop)
5. Adds a `workshop.vdf.template` describing the Workshop item

## Prerequisites

Verify these before starting:

- The mod has a valid `About/About.xml` with `packageId` in `author.modname` format
- `gh` is installed and authenticated (`gh auth status`)
- The mod is already a git repo (`git status`)
- `harmony-check` is built at the expected path (see step 2 below)

## Steps

### 1. Validate About.xml

Read `About/About.xml`. Check:
- `<packageId>` is in `author.modname` dot-notation (lowercase, letters, numbers, dots only)
- `<name>`, `<author>`, `<description>`, and `<supportedVersions>` are all present
- Fix any issues and confirm with the user before continuing

### 2. Create the pre-commit hook

Create `.git/hooks/pre-commit` with this content, then `chmod +x` it:

```bash
#!/bin/bash
set -e

REPO_ROOT="$(git rev-parse --show-toplevel)"
CHECKER="/home/deck/Developer/RimWorldMods/HarmonyConflictChecker/CLI/bin/Release/net8.0/harmony-check.dll"

# Build first so the DLL under test is current
"$REPO_ROOT/build.sh"

# Lint the mod for patch quality issues
/home/deck/.dotnet/dotnet "$CHECKER" lint "$REPO_ROOT"

# API compatibility tests — verify patched RimWorld methods still exist.
# Skipped automatically if Assembly-CSharp.dll is not found (see RIMWORLD_ASSEMBLY).
"$REPO_ROOT/test.sh"
```

If the mod has no test project yet, omit the `test.sh` line and note this to the user.

Dry-run the hook from the repo root to confirm it passes before continuing.

### 3. Create the GitHub repo

```bash
gh repo create <ModName> \
  --public \
  --description "<description from About.xml>" \
  --source <repo_root> \
  --remote origin \
  --push
```

Use the mod's `<name>` from About.xml as the repo name. Confirm the URL with the user.

### 4. Add CI workflow

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [master]
  pull_request:
    branches: [master]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Build
        run: dotnet build Source/<ModName>.csproj -c Release

      - name: Test
        run: dotnet test Tests/<ModName>.Tests/<ModName>.Tests.csproj
```

Adjust the project file paths to match the mod's actual structure. If there is no
test project, omit the Test step and note this to the user.

### 5. Add publish workflow

Create `.github/workflows/publish.yml`:

```yaml
name: Publish to Steam Workshop

on:
  workflow_dispatch:
    inputs:
      change_note:
        description: 'Change note shown on the Workshop listing'
        required: true
        default: 'Bug fixes and improvements'

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Build
        run: dotnet build Source/<ModName>.csproj -c Release

      - name: Test
        run: dotnet test Tests/<ModName>.Tests/<ModName>.Tests.csproj

      - name: Restore Steam config
        run: |
          mkdir -p $HOME/Steam/config
          echo "${{ secrets.STEAM_CONFIG_VDF }}" | base64 -d > $HOME/Steam/config/config.vdf

      - name: Install SteamCMD
        run: |
          sudo apt-get install -y lib32gcc-s1
          mkdir -p $HOME/steamcmd
          curl -sSL https://steamcdn-a.akamaihd.net/client/installer/steamcmd_linux.tar.gz \
            | tar -xz -C $HOME/steamcmd

      - name: Fill in workshop VDF
        run: |
          sed \
            -e "s|CONTENT_FOLDER|$GITHUB_WORKSPACE|g" \
            -e "s|CHANGE_NOTE|${{ github.event.inputs.change_note }}|g" \
            About/workshop.vdf.template > /tmp/workshop.vdf

      - name: Upload to Workshop
        run: |
          $HOME/steamcmd/steamcmd.sh \
            +login "${{ secrets.STEAM_USERNAME }}" "${{ secrets.STEAM_PASSWORD }}" \
            +workshop_build_item /tmp/workshop.vdf \
            +quit
```

Omit the Test step if there is no test project.

### 6. Add workshop.vdf.template

Create `About/workshop.vdf.template`:

```
"workshopitem"
{
    "appid"             "294100"
    "publishedfileid"   "0"
    "contentfolder"     "CONTENT_FOLDER"
    "previewfile"       "CONTENT_FOLDER/About/Preview.png"
    "visibility"        "0"
    "title"             "<name from About.xml>"
    "description"       "<description from About.xml>"
    "changenote"        "CHANGE_NOTE"
}
```

Note: `publishedfileid` starts as `0`. After the user does their first manual
in-game upload, they should paste the generated Workshop item ID here and commit it.

### 7. Push workflows

The `gh` token needs the `workflow` scope to push `.github/workflows/` files.
Check first:

```bash
gh auth status
```

If the `workflow` scope is missing, refresh it:

```bash
gh auth refresh -h github.com -s workflow
```

Then commit and push:

```bash
git add .github/workflows/ About/workshop.vdf.template
git commit -m "Add CI and Steam Workshop publish workflows"
git push
```

### 8. Protect the default branch

Block direct pushes to `master` (or `main`) so all changes must go through a PR.
This prevents accidental commits directly to the default branch — including from
Claude itself.

```bash
gh api repos/<owner>/<repo>/branches/master/protection \
  --method PUT \
  --field required_status_checks=null \
  --field enforce_admins=true \
  --field required_pull_request_reviews=null \
  --field restrictions=null \
  --field allow_force_pushes=false \
  --field allow_deletions=false
```

Replace `master` with `main` if that is the default branch name. The
`enforce_admins: true` flag ensures the rule applies to the repo owner too —
without it, owners can still push directly.

### 9. Finish

Tell the user:

1. **CI is live** — the build+test workflow will run on the next push
2. **Master is protected** — all changes now require a PR; direct pushes are blocked
3. **Three secrets needed** before the publish workflow will work:
   - `STEAM_USERNAME` and `STEAM_PASSWORD` for the publisher Steam account
   - `STEAM_CONFIG_VDF`: run `steamcmd +login <username> +quit` locally, then
     `base64 -w0 ~/Steam/config/config.vdf` and paste the output as the secret
4. **First Workshop upload must be done manually** in-game (dev mode → Mods →
   Upload). After that, update `publishedfileid` in `workshop.vdf.template` and
   commit so future CI runs update the same listing.
