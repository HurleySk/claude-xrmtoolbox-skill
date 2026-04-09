# Claude Code XrmToolBox Skills

A [Claude Code](https://claude.ai/claude-code) plugin with skills for building and testing XrmToolBox plugins for Dynamics 365 / Dataverse.

## Skills

### plugin-dev

Scaffold, develop, package, and publish XrmToolBox plugins.

```
/xrmtoolbox:plugin-dev new MyAwesomePlugin
/xrmtoolbox:plugin-dev deploy
/xrmtoolbox:plugin-dev pack
/xrmtoolbox:plugin-dev publish
/xrmtoolbox:plugin-dev submit
/xrmtoolbox:plugin-dev help
```

- Scaffold from the [official template](https://github.com/HurleySk/XrmToolBox-Plugin-Template) with automated find-and-replace
- Correct patterns: MEF exports, `WorkAsync`, `ExecuteMethod`, `UpdateConnection`
- NuGet packaging (DLLs in `Plugins/`)
- Local deploy and publish to nuget.org
- Submit new plugins to the XrmToolBox Tool Store for approval

### testing

Scaffold tests, create mocks, write smoke tests, and run automated UI tests.

```
/xrmtoolbox:testing scaffold
/xrmtoolbox:testing mock
/xrmtoolbox:testing smoke
/xrmtoolbox:testing ui-test
/xrmtoolbox:testing help
```

- Test project scaffolding with proper exclusions
- Configurable mock `IOrganizationService` with request recording
- Smoke test templates for CSV loading, deduplication, concurrency, SQLite
- Automated UI testing via [XrmToolBox Test Harness](https://github.com/HurleySk/xrmtoolbox-testing-toolkit) + FlaUI-MCP

### ui-review

Review plugin UI against best practices and generate improvement recommendations.

```
/xrmtoolbox:ui-review review [plugin-path]
/xrmtoolbox:ui-review checklist
/xrmtoolbox:ui-review help
```

- Static source analysis of `.cs` and `.Designer.cs` for patterns (WorkAsync, ExecuteMethod, Anchor/Dock, naming)
- Live UI inspection via Test Harness + FlaUI-MCP (layout, spacing, connection states, accessibility)
- Structured report with severity levels (Critical, Warning, Suggestion) and A-F grading
- 29 automated checks across 8 categories (Layout, Connection, Async, Feedback, Data, Input, Error Handling, Accessibility)

## Installation

### Via Marketplace

```bash
claude plugin marketplace add HurleySk/claude-plugins-marketplace
claude plugin install xrmtoolbox
```

### Direct

```bash
claude plugin add HurleySk/claude-xrmtoolbox-skill
```

## Related

- [XrmToolBox Plugin Template](https://github.com/HurleySk/XrmToolBox-Plugin-Template) - GitHub template for scaffolding new plugins
- [XrmToolBox Testing Toolkit](https://github.com/HurleySk/xrmtoolbox-testing-toolkit) - Standalone test harness for plugin UI testing
- [XrmToolBox](https://www.xrmtoolbox.com/) - The tool platform by Tanguy Touzard

## License

MIT
