# RimWorld Mod Dev Skills

Claude Code skills for RimWorld mod development. Each skill is a slash command
that instructs Claude how to handle a specific recurring task — CI setup, Workshop
publishing, compatibility checking, and more.

## Installation

Copy the `.claude/commands/` directory into your mod's repo:

```bash
curl -sSL https://github.com/Jeffrharr/ModDevSkills/archive/main.tar.gz \
  | tar -xz --strip=2 ModDevSkills-main/.claude
```

Or clone and symlink if you want to stay up to date:

```bash
git clone https://github.com/Jeffrharr/ModDevSkills.git ~/.rimworld-mod-skills
ln -s ~/.rimworld-mod-skills/.claude/commands ~/.claude/commands/rimworld
```

## Skills

| Command | Description |
|---|---|
| `/setup-mod-project` | Scaffold a new mod or retrofit an existing one: directory layout, About.xml, csproj, cross-platform build scripts, symlink |
| `/setup-linting` | Add C# linting: prompts for analyzer choice (StyleCop, Roslynator, both, or built-in only), .editorconfig, dotnet format CI step |
| `/setup-mod-ci` | Set up pre-commit hook, GitHub repo, and GitHub Actions for a mod |

## Contributing

Skills are plain markdown files in `.claude/commands/`. Each file is one slash
command. The filename becomes the command name.
