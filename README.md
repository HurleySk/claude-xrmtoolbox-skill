# Claude Code XrmToolBox Plugin Skill

A [Claude Code](https://claude.ai/claude-code) skill for building XrmToolBox plugins for Dynamics 365 / Dataverse.

## What It Does

Provides Claude with deep knowledge of XrmToolBox plugin architecture, patterns, and conventions so it can:

- Scaffold new XrmToolBox plugins from the [official template](https://github.com/HurleySk/XrmToolBox-Plugin-Template)
- Guide you through plugin development with correct patterns (MEF exports, `WorkAsync`, `ExecuteMethod`, `UpdateConnection`)
- Configure NuGet packaging correctly (DLLs in `Plugins/` folder, dependency bundling)
- Deploy and test locally
- Publish to the XrmToolBox Tool Store via nuget.org

## Installation

### As a Claude Code Plugin

```bash
claude plugin add HurleySk/claude-xrmtoolbox-skill
```

### As a Global Skill (manual)

Copy `skills/xrmtoolbox-plugin/SKILL.md` to `~/.claude/skills/xrmtoolbox-plugin/SKILL.md`.

## Usage

Once installed, use the slash command in any Claude Code session:

```
/xrmtoolbox-plugin new MyAwesomePlugin
/xrmtoolbox-plugin deploy
/xrmtoolbox-plugin pack
/xrmtoolbox-plugin help
```

Or just mention XrmToolBox plugin development in conversation and Claude will use this skill's knowledge automatically.

## Related

- [XrmToolBox Plugin Template](https://github.com/HurleySk/XrmToolBox-Plugin-Template) - GitHub template for scaffolding new plugins
- [XrmToolBox](https://www.xrmtoolbox.com/) - The tool platform by Tanguy Touzard

## License

MIT
