# Game Project Structure

RobustToolbox expects that games are using the basic directory layout specified below. 

```
project
├── Game.sln
├── Content.Client/
│   ├── Content.Client.csproj
│   ├── EntryPoint.cs
│   └── ...
├── Content.Shared/
├── Content.Server/
├── Resources/
│   ├── Textures/
│   ├── Prototypes/
│   └── ...
├── RobustToolbox/
├── bin/
    ├── Content.Client/
    ├── Content.Server/
    └── ...
├── ...
```

## C# projects

RobustToolbox relies on normal C# tooling & projects to build the engine and game code. This means games are expected to have at least one solution file (e.g. `SpaceStation14.slnx`) that can be opened in a .NET IDE, and likely many individual C# projects (`.csproj` in a subdirectory).

A minimal project is expected to have at least the following C# projects:

* `Content.Client`, containing client-only code.
* `Content.Server`, containing server-only code.
* `Content.Shared`, containing server-and-client code.
* `Content.Packaging`, containing code to package & publish your game. 

Games can have many more projects if they so desire, for a variety of reasons. The engine has no qualms loading a dozen [Content Modules](content-loading/content-modules.md) at once, if so desired.

Your "main" client and server project (`Content.Client` and `Content.Server`) are encouraged to use [Content Start](content-loading/content-modules.md#content-start) to make it easier to run them from an IDE. This is not required, however.

### Changing project prefix

It is recommended to keep all your game-loaded projects starting with the name `Content.`, both their assembly names and root namespaces. This can be changed, but **not for projects on the official SS14 launcher/hub infrastructure**[^opendream]. The following things will need to be updated if you want to change this:

* Change the actual C# project, root namespace, and output assembly name to your choosing.
* Change the assembly prefix used to discover content assemblies, **either**:
  * Change `assemblyPrefix` in your [Content Manifest](content-loading/content-manifests.md)
  * When using RT "as a library", you can change `AssemblyPrefix` in `ServerOptions`/`GameControllerOptions`.
* If you want to change the build output folder name (i.e. not using `bin/Content.Client/`), you must use RT "as a library" and change `ContentBuildDirectory` in `ServerOptions`/`GameControllerOptions`.

## `Resources/` directory

The `Resources/` directory contains all game files that aren't C# code. They are included with the game when packaged and can be accessed from code through APIs such as `IResourceManager`. Files in certain special directory names may be explicitly understood by Robust (for example `Prototypes/` is used to load prototypes from), while other files may be accessed simply by their path.

## `RobustToolbox/` directory

The source code for RobustToolbox is itself included in the repository via a [git submodule](https://git-scm.com/book/en/v2/Git-Tools-Submodules). This makes it as easy as possible to read and write engine code (whenever necessary).

Games may use a "`BuildChecker`" to ensure the submodule stays in sync for contributors. This system should be disabled if you intend to modify Robust directly, see [here](modifying/prs-with-engine-changes.md) for details.

## `bin` directory

Contains the built game files used during development. The engine will use these to load

[^opendream]: If you're wondering how OpenDream gets away with it: they have a [hardcoded exception in the engine that makes the sandboxing work](https://github.com/space-wizards/RobustToolbox/blob/899eef397c811a8ca22f079d19bbe51a3b2cdc30/Robust.Shared/ContentPack/Sandbox.yml#L15-L22). No, we aren't adding one for you.
