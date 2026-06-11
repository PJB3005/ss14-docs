# Solution & project setup

Robust expects you to just use a [regular C# project structure and solution files](../game-project-structure.md), and expects its own projects to be part of that for maximal integration. That said, we do have some expectations of how exactly RT lives in your project. These are intended to avoid any future compatibility or development hazards.

## Solution contents

Robust's projects, such as `Robust.Client`, need to be in your C# solution to be built properly. We have a tool to automatically synchronize your solution's contents and include everything you'll need from Robust.

Note that your solution **must** be a `.slnx`, not a `.sln`. The newer format is easier to work with, which our tooling relies on.

In the simplest configuration, you can simply run `Tools/Robust.SolutionGen` from your game's directory in a terminal like so:

```
dotnet run --project RobustToolbox/Tools/Robust.SolutionGen update
```

```admonish failure
Note that the above command must be run from the directory containing your solution. You **cannot** just run `Robust.SolutionGen` from your IDE and expect it to work!
```

If your project is more complicated (e.g. multiple solutions), you may need to pass some arguments to `Robust.SolutionGen`. These are documented in the help text, accessible like so:

```
dotnet run --project RobustToolbox/Tools/Robust.SolutionGen -- update --help
```

When you [update your version of Robust](../versioning-compatibility.md), we may change the contents of the solution. If this happens you should re-run `Robust.SolutionGen` and commit the result. Such cases will be mentioned in the release notes, and not happen across patch versions.

### Solution features

Robust contains some features, like `Robust.Client.WebView`, that are not necessary or desired for most games. To avoid your IDE having to unecessarily load and build these projects, they are not enabled by default. Instead, you must enable a *feature* for them.

Features are defined inside your solution file, which you will have to edit manually. Open it up, and add the following, anywhere inside the root `<Solution>` element:

```xml
<Properties Name="RobustToolbox">
    <Property Name="Features" Value="WebView" />
</Properties>
```

```admonish tip "About Visual Studio"
Visual Studio does not let you edit the solution file directly while it is loaded. You first need to *unload* the solution in the right-click menu, edit it, then reload it when you're done.

Alternatively, you can just open the solution in Notepad or something.
```

Once you've filled in which features you want, all you need to do is re-run `Robust.SolutionGen` as explained earlier!

Multiple features may be enabled, each exposing additional functionality. To do this, comma-separate the feature names:

```xml
<Properties Name="RobustToolbox">
    <Property Name="Features" Value="WebView,Foobar" />
</Properties>
```

At present, the following features exist:

| **Feature name** | **Description** |
|------------------|-----------------|
| `WebView`        | Enables use of the [`Robust.Client.WebView` module](../content-loading/robust-modules.md).

## Project references

To be able to use Robust's functionality from your game code, you'll need to depend on its projects. However, to avoid relying on any fragile implementation details, you should *not* directly reference Robust's C# projects in your own. Instead, when you want to depend on certain functionality in Robust, you should import one of the MSBuild files in our `Imports/` folder. In your `.csproj` file, this means that instead of doing this:

```xml
<ItemGroup>
    <ProjectReference Include="..\RobustToolbox\Robust.Shared\Robust.Shared.csproj" />
    <ProjectReference Include="..\RobustToolbox\Robust.Client\Robust.Client.csproj" />
</ItemGroup>
```

You should do this:

```xml
<Import Project="..\RobustToolbox\Imports\Shared.props" />
<Import Project="..\RobustToolbox\Imports\Client.props" />
```

This also means you no longer have the luxury of using your IDE to directly add project references, instead needing to open it up and do it manually.

```admonish tip "About Visual Studio"
Just like with the solution file, you'll need to unload a C# project before Visual Studio allows you to edit it by hand!
```

## Additional project imports

To set up various other miscellaneous settings, analyzers, and customizations, you should import `MSBuild/Robust.Properties.targets` in your game's C# projects.

```xml
<Import Project="..\RobustToolbox\MSBuild\Robust.Properties.targets" />
```

## NuGet package management

You are expected to use [Central Package Management](https://learn.microsoft.com/en-us/nuget/consume-packages/central-package-management) to manage NuGet package versions. Additionally, to avoid version mismatches with Robust's dependencies, you should import Robust's `Directory.Packages.props` into your own:

```xml
<Import Project="RobustToolbox/Directory.Packages.props" />
```

Note that this *disables* `ManagePackageVersionsCentrally` by default due to some technical build issues. If you have C# projects that are not directly including `Robust.Properties.targets` you should re-enable this property yourself.
