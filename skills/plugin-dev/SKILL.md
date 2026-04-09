---
name: plugin-dev
description: Create, scaffold, and develop XrmToolBox plugins for Dynamics 365 / Dataverse. Use when the user wants to create a new XrmToolBox plugin, add features to an existing one, set up packaging, or deploy locally.
argument-hint: "[new|deploy|pack|publish|submit|help] [plugin-name]"
---

# XrmToolBox Plugin Development

You are an expert at building XrmToolBox plugins for Dynamics 365 / Dataverse. Follow the patterns and conventions below.

## Template Repository

The official template is at: https://github.com/HurleySk/XrmToolBox-Plugin-Template

When creating a new plugin, clone or use that template, then apply the find-and-replace tokens listed in the template README.

## Architecture

XrmToolBox plugins follow this structure:

```
MyXrmToolBoxPlugin/
  MyXrmToolBoxPlugin.sln
  MyXrmToolBoxPlugin.csproj        # SDK-style, targets net48
  MyPluginPlugin.cs                # MEF entry point (exports IXrmToolBoxPlugin)
  MyPluginControl.cs               # Main UI control (inherits PluginControlBase)
  MyPluginControl.Designer.cs      # WinForms designer
  MyPluginControl.resx
  Properties/
    AssemblyInfo.cs                # Manual (GenerateAssemblyInfo=false)
    Resources.resx
    Resources.Designer.cs
  Resources/
    icon-128.png                   # NuGet package icon
    icon-80.png                    # Large icon in XrmToolBox
    icon-32.png                    # Small icon in XrmToolBox
  build.bat / build.ps1
  deploy.ps1
```

## Key Patterns

### Plugin Entry Point (MEF Export)

```csharp
[Export(typeof(IXrmToolBoxPlugin))]
[ExportMetadata("Name", "My Plugin Name")]
[ExportMetadata("Description", "What it does")]
[ExportMetadata("SmallImageBase64", "...")]  // base64 of icon-32.png
[ExportMetadata("BigImageBase64", "...")]    // base64 of icon-80.png
[ExportMetadata("BackgroundColor", "White")]
[ExportMetadata("PrimaryFontColor", "Black")]
[ExportMetadata("SecondaryFontColor", "Gray")]
[ExportMetadata("IsOpenSource", true)]
public class MyPluginPlugin : PluginBase, IGitHubPlugin
{
    public string RepositoryName => "MyXrmToolBoxPlugin";
    public string UserName => "GitHubUsername";

    public override IXrmToolBoxPluginControl GetControl()
    {
        return new MyPluginControl();
    }
}
```

### Main Control (PluginControlBase)

Every plugin control must implement these patterns:

1. **UpdateConnection override** -- handle connection changes:
```csharp
public override void UpdateConnection(IOrganizationService newService,
    ConnectionDetail detail, string actionName, object parameter)
{
    base.UpdateConnection(newService, detail, actionName, parameter);
    // Update UI state based on connection
}
```

2. **ExecuteMethod wrapper** -- checks connection before executing:
```csharp
private void btnAction_Click(object sender, EventArgs e)
{
    ExecuteMethod(DoWork);  // Prompts connect if not connected
}
```

3. **WorkAsync for background operations** -- never block the UI thread:
```csharp
private void DoWork()
{
    WorkAsync(new WorkAsyncInfo
    {
        Message = "Loading...",
        Work = (worker, args) =>
        {
            // Background thread -- use 'Service' (IOrganizationService) here
            args.Result = Service.Execute(new WhoAmIRequest());
        },
        PostWorkCallBack = (args) =>
        {
            if (args.Error != null)
            {
                MessageBox.Show(args.Error.Message, "Error",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
                return;
            }
            // Update UI with args.Result
        }
    });
}
```

## Project File (.csproj) Requirements

Essential settings for XrmToolBox plugins:

```xml
<TargetFramework>net48</TargetFramework>
<UseWindowsForms>true</UseWindowsForms>
<GenerateAssemblyInfo>false</GenerateAssemblyInfo>
<CopyLocalLockFileAssemblies>true</CopyLocalLockFileAssemblies>
<IncludeBuildOutput>false</IncludeBuildOutput>
<NoWarn>$(NoWarn);NU5100;NU5128</NoWarn>
```

### NuGet Dependencies (core)

```xml
<PackageReference Include="XrmToolBoxPackage" Version="1.2025.7.71" />
<PackageReference Include="MscrmTools.Xrm.Connection" Version="1.2025.7.63" />
<PackageReference Include="Microsoft.CrmSdk.CoreAssemblies" Version="9.0.2.59" />
<PackageReference Include="Microsoft.CrmSdk.Deployment" Version="9.0.2.34" />
<PackageReference Include="Microsoft.CrmSdk.XrmTooling.CoreAssembly" Version="9.1.1.65" />
<PackageReference Include="Microsoft.CrmSdk.CoreTools" Version="9.0.3.1" PrivateAssets="All" />
<PackageReference Include="System.Resources.Extensions" Version="4.7.1" />
```

### Framework Reference

```xml
<Reference Include="System.ComponentModel.Composition" />
```

### NuGet Packaging

The main plugin DLL must go in `Plugins/` (not `lib/`). Extra dependency DLLs must go in `Plugins/Dependencies/` to avoid the XrmToolBox Tool Store version mismatch validator:

```xml
<None Include="$(OutputPath)\$(AssemblyName).dll" Pack="true" PackagePath="Plugins\" />
<!-- Dependency DLLs go in a subfolder to avoid store version mismatch -->
<None Include="$(OutputPath)\SomeDependency.dll" Pack="true" PackagePath="Plugins\Dependencies\" />
```

## Commands

Based on the argument provided:

### `new [plugin-name]`
Scaffold a new XrmToolBox plugin from the GitHub template:

1. **Create the repo** from the template:
   ```bash
   gh repo create YOUR_GITHUB_USERNAME/MyXrmToolBoxPlugin --template HurleySk/XrmToolBox-Plugin-Template --public --clone
   cd MyXrmToolBoxPlugin
   ```
2. **Find and replace** these tokens across all files (in this order):
   | Token | Replace with | Example |
   |-------|-------------|---------|
   | `MyXrmToolBoxPlugin` | Full project name | `AttributeExporterXrmToolBoxPlugin` |
   | `My XrmToolBox Plugin` | Display name in XrmToolBox | `Attribute Metadata Exporter` |
   | `MyPlugin` | Class prefix | `AttributeExporter` |
   | `YOUR_GITHUB_USERNAME` | GitHub username | `HurleySk` |
   | `YOUR_NAME` | Author name | `Samuel Hurley` |
   | `YOUR_COMPANY` | Company name | `Procentrix` |
3. **Rename files** to match your plugin name:
   - `MyXrmToolBoxPlugin.sln` -> `YourName.sln`
   - `MyXrmToolBoxPlugin.csproj` -> `YourName.csproj`
   - `MyPluginPlugin.cs` -> `YourPrefixPlugin.cs`
   - `MyPluginControl.cs` -> `YourPrefixControl.cs`
   - `MyPluginControl.Designer.cs` -> `YourPrefixControl.Designer.cs`
   - `MyPluginControl.resx` -> `YourPrefixControl.resx`
4. **Generate a new GUID** for `Properties/AssemblyInfo.cs`:
   ```powershell
   [guid]::NewGuid().ToString()
   ```
5. **Replace placeholder icons** in `Resources/` with your own (128x128, 80x80, 32x32 PNG), then update the base64 strings in the plugin entry point:
   ```powershell
   [Convert]::ToBase64String([IO.File]::ReadAllBytes("Resources\icon-32.png"))
   [Convert]::ToBase64String([IO.File]::ReadAllBytes("Resources\icon-80.png"))
   ```
6. **Build to verify**: `dotnet build --configuration Release`
7. **Delete the "Getting Started" section** from the README

See the full template README for details: https://github.com/HurleySk/XrmToolBox-Plugin-Template

### `deploy`
Run `.\deploy.ps1 -Force` to build and deploy to local XrmToolBox.

### `pack`
1. `dotnet pack --configuration Release`
2. Verify the `.nupkg` contains the DLL in `Plugins/`

### `publish`
Full release workflow. Follow every step in order:

1. **Increment the version** in BOTH locations (they must match):
   - `.csproj`: `<Version>x.y.z.w</Version>`
   - `Properties/AssemblyInfo.cs`: `AssemblyVersion` and `AssemblyFileVersion`
2. **Update release notes** in `.csproj`: `<PackageReleaseNotes>` -- summarize what changed
3. **Build**: `dotnet build --configuration Release` -- must succeed with 0 errors
4. **Pack**: `dotnet pack --configuration Release` -- verify DLL is in `Plugins/` inside the `.nupkg`
5. **Commit**: Stage the changed files and commit with a message like:
   `Version x.y.z.w: <summary of changes>`
6. **Push**: `git push`
7. **Create GitHub release**: `gh release create vx.y.z.w --title "vx.y.z.w" --notes "<changelog>"`
8. **Publish to NuGet** — try the CLI first:
   ```bash
   dotnet nuget push bin\Release\YourPlugin.x.y.z.w.nupkg --api-key YOUR_KEY --source https://api.nuget.org/v3/index.json
   ```
   **If the CLI returns 403**: This is a known nuget.org issue (NuGet/Home#13403) where valid API keys are rejected via CLI, especially for new packages. Verify the API key has:
   - Scope: **"Push new packages and package versions"** (not "Push only new package versions")
   - Glob pattern: `*` (must be wildcard for packages that don't exist on nuget.org yet)

   **If CLI still fails after verifying the key**, fall back to the web upload:
   1. Go to https://www.nuget.org/packages/manage/upload
   2. Browse to the `.nupkg` file
   3. Review the metadata on the Verify page
   4. Click Submit

   The web upload authenticates via browser session and bypasses the API key entirely.

**IMPORTANT**: Always ask the user for the NuGet API key before publishing. Never hardcode or store API keys.

**First-time plugins**: After publishing to NuGet, run `submit` to register with the XrmToolBox Tool Store. This is a one-time step — subsequent versions are picked up automatically.

#### CI/CD NuGet Publishing

To automate NuGet publishing on tagged releases, add pack and push steps to `.github/workflows/release.yml`:

```yaml
- name: Pack NuGet package
  run: dotnet pack --configuration Release --no-build

- name: Push to NuGet
  run: dotnet nuget push bin\Release\*.nupkg --api-key ${{ secrets.NUGET_API_KEY }} --source https://api.nuget.org/v3/index.json --skip-duplicate
```

**Setup**: Add `NUGET_API_KEY` as a repository secret (Settings > Secrets and variables > Actions). The key needs "Push new packages and package versions" scope with `*` glob pattern.

**Note**: `--skip-duplicate` prevents failures when re-running a workflow for an already-published version.

### `submit`
Submit a plugin to the XrmToolBox Tool Store for the first time. This is required **in addition to** publishing on NuGet — the Tool Store does NOT auto-discover packages.

**Prerequisites**: The plugin must already be published on nuget.org (run `publish` first).

1. **Verify NuGet package is live**: Check `https://www.nuget.org/packages/YOUR_PACKAGE_ID` — it can take a few minutes after push for the package to become available.

2. **Open the XrmToolBox plugin registration portal**: https://www.xrmtoolbox.com/plugins/new/
   - Sign in with your XrmToolBox account (create one if needed at https://www.xrmtoolbox.com/register/)

3. **Enter the NuGet package ID** (e.g., `SpeedyNtoNAssociatePlugin`) — the portal pulls metadata from nuget.org automatically.

4. **Review the parsed metadata** — verify name, description, icon, and version are correct. Fix any issues in the `.csproj` and republish if needed.

5. **Submit for review** — an XrmToolBox administrator will validate the plugin. This typically takes 1-7 days.

6. **Wait for approval notification** — you'll receive an email when the plugin is approved and visible in the Tool Store.

**Validation checklist** (what reviewers check):
- Plugin loads without errors in XrmToolBox
- NuGet package has the `XrmToolBox` tag in `<PackageTags>`
- Package contains DLLs in `Plugins/` folder (not `lib/`)
- No bundled assemblies that conflict with XrmToolBox-provided ones
- Plugin implements `IXrmToolBoxPlugin` via MEF export
- Icon and metadata are present and reasonable

**After approval**: Future version updates published to NuGet are picked up automatically — you only need to submit once per plugin. Use `publish` for subsequent releases.

### `help`
Show this skill's available commands and XrmToolBox plugin development guidance.

## Version Management

XrmToolBox plugins use 4-part versions: `Major.Minor.Build.Revision` (e.g., `2.0.1.3`).

The version MUST be set in two places and they MUST match:
1. `.csproj` `<Version>` element
2. `Properties/AssemblyInfo.cs` `[assembly: AssemblyVersion]` and `[assembly: AssemblyFileVersion]`

When incrementing, bump the last segment for patches/fixes, third segment for features, etc.

## Common Mistakes to Avoid

- **Missing dependency DLLs in NuGet package**: If your plugin uses extra NuGet packages (e.g., CsvHelper), you MUST add their DLLs to the pack ItemGroup in the .csproj AND to `$pluginFiles` in deploy.ps1.
- **Bundling assemblies that XrmToolBox already provides**: Do NOT ship `System.Threading.Tasks.Extensions.dll`, `Microsoft.Bcl.AsyncInterfaces.dll`, `System.Buffers.dll`, `System.Memory.dll`, `System.Runtime.CompilerServices.Unsafe.dll`, `System.Text.Json.dll`, `System.ValueTuple.dll`, or other framework/SDK assemblies in your NuGet package or deploy script. XrmToolBox provides these in its install root. Duplicating them in the `Plugins/` folder causes `ReflectionTypeLoadException` at startup due to assembly version conflicts. Only bundle DLLs that are unique to your plugin (e.g., CsvHelper).
- **Using Costura.Fody**: XrmToolBox needs separate DLLs, not merged assemblies. Do not use IL merging.
- **Blocking the UI thread**: Always use `WorkAsync()` for Dataverse operations. Never call `Service.Execute()` on the UI thread.
- **Forgetting `ExecuteMethod()`**: Always wrap button handlers with `ExecuteMethod()` to ensure connection is available.
- **Wrong NuGet package layout**: DLLs must be in `Plugins/` folder, not `lib/`. Use `IncludeBuildOutput=false` and explicit `Pack` items.
- **Dependency DLLs at root Plugins/ causing store version mismatch**: The XrmToolBox Tool Store validator scans ALL DLLs at the root `Plugins/` level and compares their assembly version to the NuGet package version. If a third-party dependency (e.g., SQLitePCLRaw.core.dll v2.1.10.2445) is at root `Plugins/`, the store rejects the package with "Nuget package and Assembly must have the same version". **Fix**: Pack dependency DLLs to `Plugins/Dependencies/` (a subfolder), not root `Plugins/`. XrmToolBox loads assemblies from subfolders at runtime. Only your main plugin DLL should be at the root `Plugins/` level. Example:
  ```xml
  <None Include="$(OutputPath)\$(AssemblyName).dll" Pack="true" PackagePath="Plugins\" />
  <None Include="$(OutputPath)\SomeDependency.dll" Pack="true" PackagePath="Plugins\Dependencies\" />
  ```
  Note: The local `deploy.ps1` still puts dependencies directly in the Plugins folder — the subfolder is only needed in the NuGet package for the store validator.
- **Version mismatch**: Forgetting to update version in BOTH `.csproj` and `AssemblyInfo.cs`. They must always match.
- **Forgetting to update `<PackageReleaseNotes>`**: The release notes in `.csproj` show up on nuget.org and in the XrmToolBox Tool Store.
- **Publishing without committing first**: Always commit and push before publishing to NuGet so the source matches the published package.

## Publishing to XrmToolBox Tool Store

Publishing to NuGet is **not enough** to appear in the XrmToolBox Tool Store. New plugins must be submitted and approved through the XrmToolBox registration portal (see the `submit` command). After initial approval, subsequent version updates published to NuGet are picked up automatically.

Requirements for Tool Store discovery:
- The package must have the `XrmToolBox` tag in its `.csproj` `<PackageTags>`
- The plugin must be registered and approved at https://www.xrmtoolbox.com/plugins/new/
- DLLs must be in the `Plugins/` folder inside the `.nupkg` (not `lib/`)

## Refactoring Patterns

When XrmToolBox plugin controls grow beyond ~500 lines, consider these proven extraction patterns:

### Parameter Objects
When methods exceed 5-6 parameters, bundle configuration into a dedicated class. Example: an `AssociationRunOptions` class replacing 11 positional parameters (relationship, parallelism, bypass, retries, batch size, etc.) with a single options object. Keep parameters with different lifetimes separate (e.g., `CancellationToken`, service instances).

### Runtime Context Objects
For internal methods that pass the same pool of runtime state (client arrays, locks, counters, stopwatch), bundle them into a private `RunContext` class. This is distinct from the public options object — it holds mutable state that only the engine uses internally.

### Syntax Highlighting Extraction
XML and SQL colorization logic (regex patterns, color constants, colorize methods) is pure presentation with no dependency on plugin state. Extract to a `SyntaxHighlighter` service class. Keep suppression flags in the control since they gate UI event handlers — pass them via `ref bool`.

### Shared Result Types
Replace `Tuple<List<T>, int>` and `Tuple<List<T>, int, string>` returns from data services with a named class (e.g., `DataSourceResult` with `Pairs`, `SkippedCount`, `DiagnosticLog`). This eliminates `.Item1`/`.Item2` at call sites and makes the API self-documenting.

### Error Classification Utilities
When error-checking methods (IsTransient, IsDuplicateKey, IsConnectionError) are scattered across a large engine class, extract them to a static `ErrorClassifier` utility. Move related constants (transient patterns, error strings) with them.

### Preview Logic Consolidation
When multiple data source tabs (CSV, FetchXML, SQL) have identical post-processing in their `PostWorkCallBack` handlers, extract a shared `HandlePreviewResult` method that sets loaded pairs, populates the preview grid, toggles panel visibility, and updates progress.
