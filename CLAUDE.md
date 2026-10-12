# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A flat index of independent xCAD.NET sample add-ins, demonstrating how to use the [xCAD.NET](https://www.codestack.net/xcad/) framework (a .NET wrapper around the SOLIDWORKS COM API, by Xarial). The root `README.md` is the table of contents — each top-level folder is one self-contained example.

There is no top-level `.sln`, no shared library, no test project, no CI/linter/formatter, and no `.editorconfig`. Every example is fully self-contained and built/opened independently in Visual Studio.

## Repository layout

```
<repo>/
├── README.md                          # Index linking to each example
├── AddInApiNet6/cs/                   # .NET 6 add-in + public COM API + VBA caller
├── CommandGroupIcons/{cs,vb}/         # PNG icons of multiple sizes
├── CoordinateSystemMacroFeature/cs/   # Custom macro feature with PMP
├── DimensionWatcher/cs/               # Geometry-less macro feature
├── DocumentFeatureTabs/cs/            # Feature manager tab attachments
├── DynamicPMPageControls/cs/          # PMP controls added at runtime
├── EventsHandler/vb/                  # Document Manager event handlers
├── FormsAndWpfControls/{cs,vb}/       # WinForms + WPF embedded in SW panels
├── FurnitureConfigurator/cs/          # Data-model-driven PMP configurator
├── IssuesManager/cs/                  # Library + add-in split
├── ModelGeneratorConsole/cs/          # Standalone .NET console using xCAD
├── ParametricBox/cs/                  # Parametric macro feature
├── PMPage/cs/                         # Full PMP control showcase
├── PMPageToggleBitmapButtons/cs/      # Checkable bitmap buttons in PMP
├── ProgressBar/cs/                    # PMP progress control
├── PropertiesReader/cs/               # Multi-project: API vs Document Manager
├── PropertyAsFileName/cs/             # Hook Save-As filename
├── SqlDbEfNetCore/cs/                 # Entity Framework inside .NET Core add-in
├── SweepEdges/vb/                     # VB macro feature example
├── SweepSketch/cs/                    # Macro feature for swept bodies
├── TargetFrameworks/cs/               # SampleAddIn.sln — same code, 3 TFMs (NetFx/Core/Net6)
├── TaskPaneAddIn/cs/                  # WPF in Task Pane + WiX MSI installer
├── ToggleCommand/cs/                  # Toggle button
├── TopTenList2020/{cs,vb}/            # 3 enhancements (hide/show bodies, zoom, chamfer)
├── WeldmentProfilesSelector/cs/       # Cascading combo boxes
```

## Two project lineages

Pick the right template before scaffolding a new example; mixing them in one `.sln` is not done here.

**Legacy .NET Framework 4.6.1 (old-SDK csproj + `packages.config`):**
- PMPage, CommandGroupIcons, FormsAndWpfControls, SweepEdges, TopTenList2020, CoordinateSystemMacroFeature, DimensionWatcher, DynamicPMPageControls, FurnitureConfigurator, IssuesManager, ParametricBox, ProgressBar, PropertiesReader, PropertyAsFileName, SqlDbEfNetCore, TaskPaneAddIn, ToggleCommand, WeldmentProfilesSelector, DocumentFeatureTabs
- References SOLIDWORKS COM interops directly via `<HintPath>packages\...Xarial.XCad.SolidWorks.Interops...</HintPath>` (e.g. version 30.0.0.5041).
- Imports `packages\Xarial.XCad.SolidWorks.*\build\Xarial.XCad.SolidWorks.targets` via `<Import>`.
- `packages.config` enumerates transitive NuGet refs.
- `RootNamespace` is usually `Xarial.XCad.Examples.<Sample>.CSharp|VB`.

**Modern SDK-style .NET 6.0-windows / .NET Core (PackageReference):**
- AddInApiNet6 (net6.0-windows), TargetFrameworks/SampleAddIn1Net6 (net6.0-windows), TargetFrameworks/SampleAddIn1Core & 2Core (netcoreapp3.1), ModelGeneratorConsole (net6.0-windows)
- Always sets:
  - `<EnableComHosting>true</EnableComHosting>`
  - `<CopyLocalLockFileAssemblies>true</CopyLocalLockFileAssemblies>`
  - `<TargetFramework>net6.0-windows</TargetFramework>`
  - Single `<PackageReference Include="Xarial.XCad.SolidWorks" Version="0.8.0-2022xxxx.x" />` (or `0.8.0-preview.4296`, or older `0.7.12`).

## NuGet feed (critical for restore)

The xCAD preview packages are hosted on Azure Artifacts, **not** on nuget.org. The feed is declared in `AddInApiNet6/cs/NuGet.Config` and `CoordinateSystemMacroFeature/cs/NuGet.Config`:

```xml
<add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />
<add key="xCAD"      value="https://pkgs.dev.azure.com/xarial/xcad/_packaging/xcad-preview/nuget/v3/index.json" />
```

For .NET Framework 4.6.1 projects that only use a `packages.config`, the xCAD feed is **not** configured — Visual Studio / `nuget restore` will fail unless either the feed is added globally (`%AppData%\NuGet\NuGet.Config`) or the referenced xCAD version is already in the local `packages/` folder.

## Common xCAD add-in entry-point pattern

All add-ins follow the same skeleton (see `AddInApiNet6/cs/AddInApiNet6/SwAddInApiExample.cs` and `PMPage/cs/PMPageAddIn.cs`):

1. Inherit from `Xarial.XCad.SolidWorks.SwAddInEx`.
2. Decorate the class with `[ComVisible(true)]` and a `[Guid(...)]`.
3. Override `OnConnect()`; register commands via `this.CommandManager.AddCommandGroup<Commands_e>().CommandClick += OnCommandClick;`.
4. Declare commands as a **nested enum** with `[Title("...")]` and optional `[Icon(typeof(Resources), nameof(Resources.<name>))]` / `[CommandItemInfo(WorkspaceTypes_e.AllDocuments)]`.
5. Use `Application.ShowMessageBox(...)` for simple UI output; reach into `Application.Documents.Active` for the active model.
6. For PMP: `this.CreatePage<TDataModel>(...)` → `IXPropertyPage<TDataModel>` → `.Show(m_Data)`.

When adding a new sample, copy the closest existing `SwAddIn.cs` / `SwAddIn.vb` file and adjust the namespace, GUID, command enum, and `OnConnect` body. The `[ComVisible(true)]` and `[Guid]` attributes must be present, or SOLIDWORKS will not load the add-in.

## Macro feature pattern

Used by `SweepSketch`, `CoordinateSystemMacroFeature`, `DimensionWatcher`, `ParametricBox`, `TopTenList2020/ShaftChamfer`, and `SweepEdges/vb`. The triad is:

- **AddIn file** (`*AddIn.cs`) — owns the `SwAddInEx`, registers the macro feature with `CadExt`/`Document`.
- **`*MacroFeatureDefinition.cs`** — derives from `MacroFeatureDefinition<TData, TParams>` (or a base variant), implements `OnMakeCustomData`/`OnEdit`/`OnUpdate`. Holds the regeneration logic.
- **`*Data.cs`** — POCO carrying user-editable parameters; usually decorated with `[Title]` and `[Description]` so xCAD exposes them in the Feature Manager and edit dialog.

## Property Manager Page (PMP) pattern

Used by `PMPage`, `PMPageToggleBitmapButtons`, `WeldmentProfilesSelector`, `DynamicPMPageControls`, `FurnitureConfigurator`, `CoordinateSystemMacroFeature`, `SweepSketch`, `TopTenList2020`. Conventions seen in `PMPage/cs/`:

- A single root `m_Data` POCO whose properties are typed groups/tabs (e.g. `ControlsTab`, `SimpleControls`).
- Sub-folders `Page/Tabs/*.cs` and `Page/Groups/*.cs` model tab/group structure; each tab/class is a POCO of controls.
- A `PMPageDataModel` POCO (or per-tab POCOs) holds field values; attribute decorators on properties (`[Title]`, `[Description]`, `[NumberBoxUnit]`, `[Icon(...)]`, etc.) drive rendering.
- The add-in calls `this.CreatePage<TDataModel>(...)` with a `CreateDynamicControls` callback for runtime control creation.
- Page lifecycle events: `m_Page.Closing += OnPageClosing;` (use `arg.Cancel = true` + `arg.ErrorTitle` / `arg.ErrorMessage` to block close) and `m_Page.Closed += OnPageClosed;`.

## Cross-language (C# ↔ VB) parity

Some examples are intentionally implemented twice to compare idioms: `CommandGroupIcons/{cs,vb}`, `FormsAndWpfControls/{cs,vb}`, `TopTenList2020/{cs,vb}`, `EventsHandler/vb` (VB only). When changing one, check the other.

## Public COM API exposure

`AddInApiNet6/cs` is the only sample that exposes an out-of-process API. Pattern:

1. Define a `[ComVisible(true)] [Guid("...")]` interface (e.g. `ISwAddInApiExampleApi`) with the methods you want to be callable.
2. The add-in class implements that interface.
3. `Macro.vba` shows the consumer side: `CreateObject("ProgId").SayHello("name")` from a VB macro.
4. Required csproj flags: `<EnableComHosting>true</EnableComHosting>` and `<CopyLocalLockFileAssemblies>true</CopyLocalLockFileAssemblies>`.

## Multi-target-framework pattern

`TargetFrameworks/cs/SampleAddIn.sln` is the reference for cross-TFM packaging:

- Solution folders `.Net Framework`, `.NET Core`, `.NET6` group the per-TFM projects.
- A single `SharedLib` (`net461`, `VersionPrefix=2.0.0`, `OutDir=bin\$(Configuration)\$(VersionPrefix)`) is consumed by the add-in projects.
- TFM matrix: `SampleAddIn1{Framework,Core,Net6}` and `SampleAddIn2{Framework,Core,Net6}` — same add-in source, different `<TargetFramework>` and xCAD package version.
- xCAD preview feed is required for the `Core`/`Net6` projects; the `Framework` project uses `Xarial.XCad.SolidWorks 0.7.12` (already on nuget.org).

## Build / run cheatsheet

There is no solution-wide build. Per example, in Visual Studio: open `<example>/cs|vb/<example>.sln` (or `.sln` next to it) and press F5 with SOLIDWORKS installed locally — the add-in DLL is registered via xCAD's COM registration.

From the command line (modern examples):

```bash
# from <example>/cs
dotnet restore --configfile NuGet.Config   # only AddInApiNet6 & CoordinateSystemMacroFeature ship one
dotnet build <Example>.sln -c Debug
```

For `packages.config`-style projects, use `nuget restore <Example>.sln` then `msbuild <Example>.sln /p:Configuration=Debug`. The xCAD preview feed must be reachable, or the pre-existing local `packages/` folder must contain the referenced xCAD version.

There are **no unit tests** in this repository.

## Files to read first when changing an example

| To change… | Read first |
| --- | --- |
| A specific add-in's commands | `SwAddIn.cs` / `SwAddIn.vb` / `*AddIn.cs` (entry), then `Properties/Resources.resx` for icons |
| PMP control rendering | `PMPageAddIn.cs` + `Page/PMPageDataModel.cs` + `Page/Tabs/*.cs` + `Page/Groups/*.cs` |
| Macro feature behavior | `*MacroFeatureDefinition.cs` (regen logic) + `*Data.cs` (parameters) |
| Cross-TFM packaging | `TargetFrameworks/cs/SampleAddIn.sln` and the three per-TFM `*AddIn.cs` files |
| COM API to outside callers | `AddInApiNet6/cs/AddInApiNet6/SwAddInApiExample.cs` (interface + class) and `AddInApiNet6/cs/Macro.vba` (consumer) |
| Library/add-in split | `IssuesManager/cs/IssuesManager.sln` (two projects in one solution) |
