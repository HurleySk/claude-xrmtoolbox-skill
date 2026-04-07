---
name: xrmtoolbox-plugin
description: Create, scaffold, and develop XrmToolBox plugins for Dynamics 365 / Dataverse. Use when the user wants to create a new XrmToolBox plugin, add features to an existing one, set up packaging, or deploy locally.
argument-hint: "[new|deploy|pack|help] [plugin-name]"
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

DLLs must go in `Plugins/` (not `lib/`). Include main DLL and any extra dependencies:

```xml
<None Include="$(OutputPath)\$(AssemblyName).dll" Pack="true" PackagePath="Plugins\" />
<!-- For each extra dependency DLL: -->
<None Include="$(OutputPath)\SomeDependency.dll" Pack="true" PackagePath="Plugins\" />
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
8. **Publish to NuGet**:
   ```bash
   dotnet nuget push bin\Release\YourPlugin.x.y.z.w.nupkg --api-key YOUR_KEY --source https://api.nuget.org/v3/index.json
   ```

**IMPORTANT**: Always ask the user for the NuGet API key before publishing. Never hardcode or store API keys.

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
- **Version mismatch**: Forgetting to update version in BOTH `.csproj` and `AssemblyInfo.cs`. They must always match.
- **Forgetting to update `<PackageReleaseNotes>`**: The release notes in `.csproj` show up on nuget.org and in the XrmToolBox Tool Store.
- **Publishing without committing first**: Always commit and push before publishing to NuGet so the source matches the published package.

## Publishing to XrmToolBox Tool Store

The XrmToolBox Tool Store automatically discovers packages from nuget.org that have the `XrmToolBox` tag. After publishing with `dotnet nuget push`, it may take a few minutes for the package to appear in the Tool Store.

The package must have the `XrmToolBox` tag in its `.csproj` `<PackageTags>` to be discovered.
