---
name: ui-review
description: Review XrmToolBox plugin UI and suggest improvements based on best practices. Use when the user wants a UI review, UX audit, accessibility check, or layout analysis of an XrmToolBox plugin.
argument-hint: "[review|checklist|help] [plugin-path]"
---

# XrmToolBox Plugin UI Review

You are an expert at reviewing XrmToolBox plugin user interfaces against best practices for WinForms-based Dynamics 365 / Dataverse tools. Follow the patterns and conventions below.

## Overview

This skill performs a structured UI review of an XrmToolBox plugin by combining:
- **Static analysis** of plugin source code (`.cs`, `.Designer.cs` files) for patterns like WorkAsync, ExecuteMethod, Anchor/Dock, naming conventions
- **Live UI analysis** via the XrmToolBox Test Harness and FlaUI-MCP for visual layout, spacing, accessibility, and interactive behavior

The review produces a structured report with severity levels: **Critical** (breaks usability), **Warning** (degrades experience), **Suggestion** (polish/improvement).

## Commands

Based on the argument provided:

### `review [plugin-path]`

Full UI review workflow. Execute each step in order, checking results before proceeding.

**Terminology**: Throughout this workflow, `$HARNESS_EXE`, `$HARNESS_REPO`, `$PLUGIN_DIR`, and `$PLUGIN_DLL` are conceptual variables — remember these paths as you discover them in Step 1. `$PLUGIN_DIR` is the path the user provides (or the current working directory if not provided).

#### Step 1: Verify Prerequisites

Execute these checks in order. Fix any that fail before proceeding.

**1a. Locate the test harness repo and exe.**

Glob for `XrmToolBox.TestHarness.exe` on disk:
```bash
find "$HOME/source/repos" -path "*/xrmtoolbox-testing-toolkit/*/bin/Release/*/XrmToolBox.TestHarness.exe" 2>/dev/null | head -1
```

- **Check**: The find returns a path. Remember it as `$HARNESS_EXE`. Derive `$HARNESS_REPO` as the repo root (ancestor containing `XrmToolBox.TestHarness.slnx`).
- **Fix if not found**: Clone and build:
```bash
git clone https://github.com/HurleySk/xrmtoolbox-testing-toolkit.git "$HOME/source/repos/xrmtoolbox-testing-toolkit"
cd "$HOME/source/repos/xrmtoolbox-testing-toolkit"
dotnet build --configuration Release
```
Then re-run the find to get `$HARNESS_EXE` and `$HARNESS_REPO`.

**1b. Check that FlaUI-MCP is installed and registered.**

```bash
test -f "C:/tools/FlaUI-MCP/FlaUI.Mcp.exe" && echo "INSTALLED" || echo "NOT INSTALLED"
```

- **Check**: Output says `INSTALLED`.
- **Fix if NOT INSTALLED**: Run the setup script from the harness repo:
```powershell
powershell -ExecutionPolicy Bypass -File "$HARNESS_REPO\setup-flaui-mcp.ps1"
```

> **IMPORTANT — Restart Required**: After installing FlaUI-MCP for the first time, Claude Code **must be restarted** before the `flaui-mcp` MCP tools become available. If you just ran `setup-flaui-mcp.ps1`, **STOP HERE** and instruct the user to restart Claude Code, then re-run `/xrmtoolbox:ui-review review`. Do NOT proceed — FlaUI-MCP tools will not work in the current session.

- **Verify after restart**: Confirm FlaUI-MCP is loaded by checking that `windows_list_windows` is available as a tool.

**1c. Locate the plugin DLL.**

First, find the `.csproj` file name to know the DLL name:
```bash
ls "$PLUGIN_DIR"/*.csproj
```

The plugin DLL name matches the `.csproj` stem (e.g., `MyPlugin.csproj` → `MyPlugin.dll`). Search for it:
```bash
find "$PLUGIN_DIR/bin/Release" -name "MyPlugin.dll" 2>/dev/null | head -1
```
Replace `MyPlugin.dll` with the actual name derived from the `.csproj`.

- **Check**: The find returns a path.
- **Fix if not found**: Build the plugin:
```bash
dotnet build "$PLUGIN_DIR"/*.csproj --configuration Release
```
- Remember the full path as `$PLUGIN_DLL`.

#### Step 2: Static Source Code Analysis

Analyze the plugin source code against the best practices checklist **before** launching the harness. This catches architectural issues that don't need a running UI.

**2a. Locate source files.**

```bash
find "$PLUGIN_DIR" -name "*.cs" -not -path "*/obj/*" -not -path "*/bin/*" -not -path "*/Tests/*" | sort
```

Identify these key files:
- `*Control.cs` — main plugin control (business logic, event handlers)
- `*Control.Designer.cs` — WinForms designer (layout, control properties)
- `*Plugin.cs` — MEF entry point

**2b. Analyze Designer.cs for layout and control properties.**

Read each `*Control.Designer.cs` file and check:

| Check ID | What to Look For | How to Detect | Severity |
|----------|-----------------|---------------|----------|
| LAYOUT-01 | Controls use Anchor or Dock | Search for `.Anchor =` and `.Dock =` assignments. Controls with neither are fixed-position and won't resize. | Warning |
| LAYOUT-02 | GroupBox used for logical grouping | Search for `new System.Windows.Forms.GroupBox()`. Absence with 5+ interactive controls suggests poor grouping. | Suggestion |
| LAYOUT-03 | Consistent margins from edges | Check `Location` values — controls at x < 12 or y < 12 are too close to edges. | Suggestion |
| LAYOUT-04 | TableLayoutPanel or FlowLayoutPanel for complex layouts | Search for `TableLayoutPanel` or `FlowLayoutPanel`. Their absence with 10+ controls suggests manual positioning. | Suggestion |
| NAME-01 | Controls use Hungarian notation | Check that control names start with a known prefix (btn, txt, cbo, cmb, dgv, chk, nud, rtb, lbl, grp, pnl, tab, progress). Names like `button1`, `dataGridView1` are auto-generated and unhelpful. | Warning |
| NAME-02 | Meaningful control names | Names like `btnButton1` or `textBox2` lack semantic meaning. Button names should indicate action (e.g., `btnLoadEntities`). | Suggestion |
| ACCESS-01 | Tab order is explicitly set | Search for `.TabIndex =` assignments. If absent, tab order is undefined and keyboard navigation will be unpredictable. | Warning |
| ACCESS-02 | Font size is readable | Search for `new System.Drawing.Font(` and check point size. Below 8.25pt is too small for standard use. | Suggestion |
| TOOLTIP-01 | Tooltips defined for complex controls | Search for `ToolTip` component and `.SetToolTip(` calls. Absence with complex input controls suggests missing guidance. | Suggestion |

For each check, record the finding as:
```
[SEVERITY] CHECK-ID: Description of what was found
  Evidence: specific line or pattern
  Recommendation: what to change
```

**2c. Analyze Control.cs for patterns and async behavior.**

Read each `*Control.cs` file and check:

| Check ID | What to Look For | How to Detect | Severity |
|----------|-----------------|---------------|----------|
| ASYNC-01 | WorkAsync used for long operations | Search for `WorkAsync(`. If button handlers call `Service.Execute`, `Service.RetrieveMultiple`, etc. without wrapping in WorkAsync, the UI will freeze. | Critical |
| ASYNC-02 | ExecuteMethod wraps connection-dependent handlers | Search for `ExecuteMethod(`. Button click handlers that use `Service` should be wrapped with `ExecuteMethod()` to ensure connection exists. | Critical |
| CONN-01 | UpdateConnection is overridden | Search for `override.*UpdateConnection`. Missing means the plugin doesn't react to connection changes. | Warning |
| CONN-02 | Connection status shown to user | Search for labels or status text updated in `UpdateConnection` or a connection state method (e.g., `lblStatus`, `lblConnection`, `tslConnection`). | Warning |
| CONN-03 | Buttons disabled when disconnected | Search for `.Enabled = ` in `UpdateConnection` or a connection state method. Key action buttons should be disabled until connected. | Warning |
| ERR-01 | Error handling in PostWorkCallBack | In `WorkAsync` calls, check that `PostWorkCallBack` handles `args.Error != null`. Missing error handling silently swallows failures. | Critical |
| ERR-02 | MessageBox with appropriate icons | Search for `MessageBox.Show(`. Check that error dialogs use `MessageBoxIcon.Error`, warnings use `MessageBoxIcon.Warning`, etc. Calls without icons lack visual severity. | Suggestion |
| ERR-03 | Validation before operations | Check that button handlers validate inputs (e.g., checking for empty textboxes, null selections) before calling WorkAsync or ExecuteMethod. | Warning |
| FEED-01 | Status labels updated after operations | In `PostWorkCallBack`, check that a label or status control is updated with results (e.g., "Loaded 42 records"). | Suggestion |
| FEED-02 | Progress reporting for batch operations | For loops inside `WorkAsync`, check for `worker.ReportProgress()` or `ProgressChanged`. Long loops without progress leave users uncertain. | Suggestion |
| DATA-01 | DataGridView with sorting | If `DataGridView` exists, check if `SortMode` is set on columns, or `DataSource` is a sortable type (DataTable, BindingList, BindingSource). | Suggestion |
| DATA-02 | Context menus for grids | Search for `ContextMenuStrip` associated with DataGridView controls. Right-click menus are expected for grid operations (copy, export, etc.). | Suggestion |

**2d. Run generate-mockdata.ps1 for control inventory.**

```powershell
powershell -ExecutionPolicy Bypass -File "$HARNESS_REPO\generate-mockdata.ps1" -PluginSourceDir "$PLUGIN_DIR"
```

The script produces two files in `$PLUGIN_DIR`:
- `test-control-inventory.json` — all UI controls with names, types, and interaction categories
- `test-mockdata.json` — starter mock data with responses for discovered SDK patterns

Verify both files exist:
```bash
test -f "$PLUGIN_DIR/test-control-inventory.json" && echo "OK" || echo "MISSING: control inventory"
test -f "$PLUGIN_DIR/test-mockdata.json" && echo "OK" || echo "MISSING: mock data"
```

If either is MISSING, check the script output for errors. Common causes: plugin source has no `*Control.Designer.cs` file, or the `-PluginSourceDir` path is wrong.

Read `$PLUGIN_DIR/test-control-inventory.json`. Use the `summary` to assess:
- **No feedback controls** (labels, progress bars): Warning — users get no status feedback
- **No data-display controls** (grids): Suggestion if the plugin loads data
- **No input-dropdown controls**: Suggestion if the plugin selects from lists
- **All buttons lack semantic names**: Warning — indicates auto-generated names

#### Step 3: Launch Test Harness for Live UI Analysis

**3a. Launch the harness.**

```bash
"$HARNESS_EXE" --plugin "$PLUGIN_DLL" --mockdata "$PLUGIN_DIR/test-mockdata.json" --screenshots "$PLUGIN_DIR/screenshots" --record "$PLUGIN_DIR/calls.json" &
```

Wait 3 seconds (`sleep 3`), then verify via FlaUI-MCP: call `windows_list_windows` and look for a window with title containing `"Test Harness"`. Remember its handle (e.g., `w1`).

**3b. If the harness fails to launch**, check these common causes in order:
1. **Plugin DLL dependencies missing**: Copy dependency DLLs from harness output dir alongside plugin DLL, or vice versa.
2. **JSON syntax error**: Validate `test-mockdata.json` is valid JSON (`python -m json.tool "$PLUGIN_DIR/test-mockdata.json"`).
3. **Plugin throws on load**: Check console output; the plugin may require specific mock data during initialization (e.g., WhoAmI response must be present in mock data).

#### Step 4: Live UI Review via FlaUI-MCP

Drive the UI through multiple states and capture screenshots + snapshots at each state. For each state, apply the live UI checks.

**4a. Initial State (disconnected/empty).**

1. Call `windows_focus` on the harness handle.
2. Call `windows_screenshot` — save as "01-initial-state".
3. Call `windows_snapshot` — capture the accessibility tree.

**Check against the snapshot:**

| Check ID | What to Look For | How to Detect | Severity |
|----------|-----------------|---------------|----------|
| LIVE-LAYOUT-01 | Controls don't overlap or clip | In the snapshot, check that no two sibling controls share the same bounding rectangle region. | Warning |
| LIVE-LAYOUT-02 | Reasonable spacing between controls | Check bounding rectangles — controls should have at least 4px gap between them. | Suggestion |
| LIVE-CONN-01 | Connection status visible | Look for a label or status element showing "Not connected", "Disconnected", or similar. | Warning |
| LIVE-CONN-02 | Action buttons disabled | Use `windows_find_elements` with `controlType: "Button"` and check `IsEnabled` property. Primary action buttons should be disabled before connection. | Warning |
| LIVE-ACCESS-01 | Tab order is logical | Note the order of interactive elements (buttons, textboxes, dropdowns, checkboxes) in the snapshot tree — this reflects tab order. Controls should flow top-to-bottom, left-to-right. | Suggestion |

**4b. Connected State (after harness auto-connects with mock service).**

The harness auto-connects unless `--no-autoconnect` was passed. The connection happens during harness startup, so by the time `windows_list_windows` finds the window in Step 3a, the mock service is already injected.

1. Call `windows_snapshot` — check if connection status label changed.
2. Call `windows_screenshot` — save as "02-connected-state".

**Check:**
- Connection label should now show connected org name or "Connected"
- Action buttons should now be enabled
- Status colors: look for green/gray color indicators if visible in snapshot

**4c. Populated State (after primary action).**

1. Set up any required input controls using the control inventory from Step 2d:
   - **Dropdowns** (category `input-dropdown` in the inventory): `windows_click` to expand, then `windows_snapshot` to see items, then `windows_click` on the first available item.
   - **TextBoxes** (category `input-text`): `windows_fill` with a test value appropriate to the control name (e.g., `txtFilter` → `"test"`, `txtFilePath` → a temp file path).
   - **CheckBoxes** (category `input-toggle`): Leave at defaults unless they gate the primary action.
2. Click the `primaryAction` button from the control inventory using `windows_click`.
3. Wait for completion using `windows_wait_for_element` (wait for grid to populate, status label to update, or button to re-enable).
4. Call `windows_screenshot` — save as "03-populated-state".
5. Call `windows_snapshot` — capture the populated tree.

**Check:**

| Check ID | What to Look For | How to Detect | Severity |
|----------|-----------------|---------------|----------|
| LIVE-DATA-01 | Grid headers are descriptive | Use `windows_get_table_data` on the grid. Column headers should not be "Column1", "Column2" — they should be meaningful names. | Warning |
| LIVE-DATA-02 | Data is populated | Grid should have at least one row of data (from mock data). | Warning |
| LIVE-FEED-01 | Status updated after operation | Use `windows_get_text` on status labels. They should reflect the completed operation (e.g., "Loaded 1 record"). | Suggestion |
| LIVE-FEED-02 | Button text reflects state | After loading data, check if the primary action button changed text (e.g., "Load" → "Reload"). | Suggestion |

**4d. Tab Navigation Check.**

1. Call `windows_snapshot` to get the current element tree.
2. Note the ordering of interactive elements (buttons, textboxes, dropdowns, checkboxes) in the tree — this reflects accessibility/tab order.
3. If the order is illogical (e.g., a bottom button appears before a top textbox), flag as `ACCESS-01`.

**4e. Error State (optional — if time permits).**

If the plugin has error handling paths:
1. Check `windows_list_windows` for any error dialog windows.
2. If an error dialog appeared, screenshot it and check:
   - Does it have an appropriate icon (Error, Warning)?
   - Is the message user-friendly (not a raw exception stack trace)?
   - Does it have OK/Cancel buttons as appropriate?

#### Step 5: Close and Compile Report

**5a. Close the harness.**

Call `windows_close` on the harness window handle so it writes `calls.json`.

**5b. Check SDK call recording (supplementary).**

Read `$PLUGIN_DIR/calls.json` if it exists. Note:
- Whether the plugin made SDK calls during the review (validates that mock data worked)
- Any unmatched calls that might indicate missing mock data

**5c. Generate the structured review report.**

Compile all findings from Steps 2 and 4 into this format:

```
# UI Review Report: [Plugin Name]
Date: [date]
Plugin: [plugin path]

## Summary
- Critical: [count]
- Warning: [count]
- Suggestion: [count]
- Overall Grade: [A/B/C/D/F based on criteria below]

## Critical Findings
[List each Critical finding with Check ID, description, evidence, recommendation]

## Warnings
[List each Warning finding with Check ID, description, evidence, recommendation]

## Suggestions
[List each Suggestion finding with Check ID, description, evidence, recommendation]

## Screenshots
- Initial state: [path]
- Connected state: [path]
- Populated state: [path]

## Best Practices Compliance
| Category | Status | Notes |
|----------|--------|-------|
| Layout & Responsiveness | [Pass/Partial/Fail] | ... |
| Connection Management | [Pass/Partial/Fail] | ... |
| Async Operations | [Pass/Partial/Fail] | ... |
| User Feedback | [Pass/Partial/Fail] | ... |
| Data Display | [Pass/Partial/Fail] | ... |
| Input Controls | [Pass/Partial/Fail] | ... |
| Error Handling | [Pass/Partial/Fail] | ... |
| Accessibility | [Pass/Partial/Fail] | ... |
```

**Grading criteria:**
- **A**: 0 Critical, 0-2 Warnings
- **B**: 0 Critical, 3-5 Warnings
- **C**: 1 Critical or 6+ Warnings
- **D**: 2-3 Critical findings
- **F**: 4+ Critical findings

### `checklist`

Display the UI best practices checklist without running any analysis. Print the following:

```
=== XrmToolBox Plugin UI Best Practices Checklist ===

LAYOUT & RESPONSIVENESS
  [ ] Controls use Anchor or Dock for responsive layout (not fixed positions)
  [ ] GroupBox used for logical grouping of related controls
  [ ] Consistent padding/margins (12-20px from edges)
  [ ] Controls don't overlap or clip at different window sizes
  [ ] TableLayoutPanel or FlowLayoutPanel for complex layouts

CONNECTION MANAGEMENT
  [ ] Clear connection status indicator (label showing connected/disconnected)
  [ ] Buttons disabled when not connected
  [ ] UpdateConnection override handles connection changes
  [ ] Status colors: Green for connected, Gray for disconnected, DarkOrange for warnings

ASYNC OPERATIONS
  [ ] WorkAsync used for ALL long-running operations (no UI blocking)
  [ ] ExecuteMethod wraps button handlers that need a connection
  [ ] Loading indicators shown during operations
  [ ] Progress bars for known-length operations (worker.ReportProgress)
  [ ] Cancel capability for long operations

USER FEEDBACK
  [ ] MessageBox with appropriate icons (Error, Warning, Information, Question)
  [ ] Status labels updated after operations complete
  [ ] Real-time counters during batch operations
  [ ] Button text reflects state ("Load" vs "Reload")

DATA DISPLAY
  [ ] DataGridView with sorting support
  [ ] Column headers present and descriptive (not "Column1")
  [ ] Right-click context menus for grid operations
  [ ] Preview/limit for large datasets (e.g., 100 rows)

INPUT CONTROLS
  [ ] ComboBox with AutoComplete for searchable lists
  [ ] TextBox validation for required fields
  [ ] Tooltips on controls that need explanation
  [ ] Logical default values for inputs

ERROR HANDLING
  [ ] Error dialogs shown for failures (not silent swallowing)
  [ ] PostWorkCallBack checks args.Error != null
  [ ] Validation before operations (not after failure)
  [ ] Friendly error messages (not raw exception stack traces)

ACCESSIBILITY
  [ ] Tab order is logical (top-to-bottom, left-to-right)
  [ ] Controls have meaningful names (Hungarian notation: btn, txt, cbo, etc.)
  [ ] Font sizes readable (8.25pt+ default)
  [ ] No auto-generated names (button1, textBox2)
```

### `help`

Show available commands:

```
XrmToolBox UI Review Commands:

  /xrmtoolbox:ui-review review [plugin-path]
    Run a full UI review — static source analysis + live UI inspection
    via FlaUI-MCP. Produces a structured report with findings and
    recommendations graded by severity (Critical, Warning, Suggestion).

  /xrmtoolbox:ui-review checklist
    Display the UI best practices checklist for quick reference.
    No analysis is performed.

  /xrmtoolbox:ui-review help
    Show this help message.

Prerequisites (auto-installed if missing):
  - XrmToolBox Test Harness (xrmtoolbox-testing-toolkit)
  - FlaUI-MCP (requires Claude Code restart after first install)
  - Plugin DLL (auto-built if not found)
```

## Appendix: Check ID Reference

| Check ID | Category | Severity | Source | Description |
|----------|----------|----------|--------|-------------|
| LAYOUT-01 | Layout | Warning | Static | Controls use Anchor or Dock |
| LAYOUT-02 | Layout | Suggestion | Static | GroupBox used for grouping |
| LAYOUT-03 | Layout | Suggestion | Static | Consistent margins (12px+) |
| LAYOUT-04 | Layout | Suggestion | Static | TableLayoutPanel for complex layouts |
| NAME-01 | Accessibility | Warning | Static | Hungarian notation prefixes |
| NAME-02 | Accessibility | Suggestion | Static | Meaningful control names |
| ACCESS-01 | Accessibility | Warning | Static/Live | Tab order explicitly set |
| ACCESS-02 | Accessibility | Suggestion | Static | Font size 8.25pt+ |
| TOOLTIP-01 | Input | Suggestion | Static | Tooltips on complex controls |
| ASYNC-01 | Async | Critical | Static | WorkAsync for long operations |
| ASYNC-02 | Async | Critical | Static | ExecuteMethod wraps handlers |
| CONN-01 | Connection | Warning | Static | UpdateConnection overridden |
| CONN-02 | Connection | Warning | Static | Connection status displayed |
| CONN-03 | Connection | Warning | Static | Buttons disabled when disconnected |
| ERR-01 | Error | Critical | Static | PostWorkCallBack error handling |
| ERR-02 | Error | Suggestion | Static | MessageBox icons appropriate |
| ERR-03 | Error | Warning | Static | Input validation before operations |
| FEED-01 | Feedback | Suggestion | Static | Status labels updated |
| FEED-02 | Feedback | Suggestion | Static | Progress reporting in loops |
| DATA-01 | Data | Suggestion | Static | Grid sorting support |
| DATA-02 | Data | Suggestion | Static | Context menus for grids |
| LIVE-LAYOUT-01 | Layout | Warning | Live | No overlapping controls |
| LIVE-LAYOUT-02 | Layout | Suggestion | Live | Reasonable spacing |
| LIVE-CONN-01 | Connection | Warning | Live | Status visible in UI |
| LIVE-CONN-02 | Connection | Warning | Live | Buttons disabled before connect |
| LIVE-ACCESS-01 | Accessibility | Suggestion | Live | Logical tab order |
| LIVE-DATA-01 | Data | Warning | Live | Descriptive grid headers |
| LIVE-DATA-02 | Data | Warning | Live | Grid populates with data |
| LIVE-FEED-01 | Feedback | Suggestion | Live | Status updated after action |
| LIVE-FEED-02 | Feedback | Suggestion | Live | Button text reflects state |

## Appendix: FlaUI-MCP Tools Used

| Tool | Usage in Review |
|------|----------------|
| `windows_list_windows` | Find the test harness window handle |
| `windows_focus` | Bring harness to foreground before screenshots |
| `windows_screenshot` | Capture UI state at each review phase |
| `windows_snapshot` | Get accessibility tree for element analysis |
| `windows_click` | Interact with buttons, dropdowns |
| `windows_fill` | Set input values for populated state |
| `windows_get_text` | Read status labels, connection indicators |
| `windows_get_table_data` | Read grid headers and data |
| `windows_find_elements` | Search for specific control types |
| `windows_wait_for_element` | Wait for UI changes after actions |
| `windows_close` | Close harness to trigger calls.json write |

## Appendix: Common Patterns in Well-Designed Plugins

These are examples from production XrmToolBox plugins that demonstrate best practices:

**Connection status pattern:**
```csharp
public override void UpdateConnection(IOrganizationService newService,
    ConnectionDetail detail, string actionName, object parameter)
{
    base.UpdateConnection(newService, detail, actionName, parameter);
    lblConnection.Text = detail?.OrganizationFriendlyName ?? "Not connected";
    lblConnection.ForeColor = detail != null ? Color.Green : Color.Gray;
    btnLoad.Enabled = detail != null;
}
```

**WorkAsync with progress pattern:**
```csharp
WorkAsync(new WorkAsyncInfo
{
    Message = "Loading entities...",
    Work = (worker, args) =>
    {
        var results = new List<Entity>();
        // ... load data with worker.ReportProgress(percent, message) ...
        args.Result = results;
    },
    ProgressChanged = (args) =>
    {
        SetWorkingMessage(args.UserState?.ToString() ?? $"{args.ProgressPercentage}%");
    },
    PostWorkCallBack = (args) =>
    {
        if (args.Error != null)
        {
            MessageBox.Show(args.Error.Message, "Error",
                MessageBoxButtons.OK, MessageBoxIcon.Error);
            return;
        }
        var results = (List<Entity>)args.Result;
        dgvResults.DataSource = results;
        lblStatus.Text = $"Loaded {results.Count} records";
    }
});
```

**Responsive layout pattern (Designer.cs):**
```csharp
this.dgvResults.Anchor = ((System.Windows.Forms.AnchorStyles)((((
    System.Windows.Forms.AnchorStyles.Top |
    System.Windows.Forms.AnchorStyles.Bottom) |
    System.Windows.Forms.AnchorStyles.Left) |
    System.Windows.Forms.AnchorStyles.Right)));
this.dgvResults.Location = new System.Drawing.Point(12, 180);
```

## Appendix: Control Name Prefixes (Hungarian Notation)

| Prefix | Control Type | FlaUI-MCP Tool |
|--------|-------------|---------------|
| btn | Button | `windows_click` |
| dgv | DataGridView | `windows_snapshot`, `windows_get_table_data` |
| cbo, cmb | ComboBox | `windows_click` (expand + select) |
| txt | TextBox | `windows_fill` |
| chk | CheckBox | `windows_click` (toggle) |
| nud | NumericUpDown | `windows_fill` |
| rtb | RichTextBox | `windows_fill` |
| lbl | Label | `windows_get_text` |
| grp | GroupBox | (container) |
| pnl | Panel | (container) |
| tab | TabControl | `windows_click` (tab header) |
| progress | ProgressBar | `windows_get_text` |
