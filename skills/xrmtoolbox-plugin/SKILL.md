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
1. Clone the template from https://github.com/HurleySk/XrmToolBox-Plugin-Template
2. Run the find-and-replace for the plugin name tokens
3. Rename all files
4. Generate a new GUID for AssemblyInfo.cs
5. Build to verify

### `deploy`
Run `.\deploy.ps1 -Force` to build and deploy to local XrmToolBox.

### `pack`
1. `dotnet pack --configuration Release`
2. Verify the `.nupkg` contains the DLL in `Plugins/`

### `help`
Show this skill's available commands and XrmToolBox plugin development guidance.

## Common Mistakes to Avoid

- **Missing dependency DLLs in NuGet package**: If your plugin uses extra NuGet packages (e.g., CsvHelper), you MUST add their DLLs to the pack ItemGroup in the .csproj AND to `$pluginFiles` in deploy.ps1.
- **Using Costura.Fody**: XrmToolBox needs separate DLLs, not merged assemblies. Do not use IL merging.
- **Blocking the UI thread**: Always use `WorkAsync()` for Dataverse operations. Never call `Service.Execute()` on the UI thread.
- **Forgetting `ExecuteMethod()`**: Always wrap button handlers with `ExecuteMethod()` to ensure connection is available.
- **Wrong NuGet package layout**: DLLs must be in `Plugins/` folder, not `lib/`. Use `IncludeBuildOutput=false` and explicit `Pack` items.

## Publishing to XrmToolBox Tool Store

The XrmToolBox Tool Store pulls from nuget.org. To publish:

```bash
dotnet nuget push bin\Release\YourPlugin.x.y.z.nupkg --api-key YOUR_KEY --source https://api.nuget.org/v3/index.json
```

The package must have the `XrmToolBox` tag to be discovered.
