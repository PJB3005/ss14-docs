# Content modules

RobustToolbox, like most game engines, expects to be able to start up the process on its own terms before it loads your custom code. When RT is ready to run your code, it will load your game's "modules" (.NET assemblies) and execute their entry points.

## Entry points

After RT has loaded your module, it will automatically locate all types inheriting from `Robust.Shared.ContentPack.GameShared`. These types will be instantiated and have virtual functions like `Init()` called at specific points.

```admonish info
Besides `GameShared`, the types `GameClient` and `GameServer` also exist. These inherit from `GameShared` and, at the moment, add no additional functionality. Using them can make your code a bit more clear, but it is not strictly required.
```

## Assembly discovery

By default, RT will load .NET assemblies in the `/Assemblies/` resource path. For assemblies to be loaded, they must start with the name `Content`. In development builds, your game's bin directories (`bin/Content.Client` and `bin/Content.Server`) are mapped to `/Assemblies/` automatically.

When RT is used "as a library", the paths & prefixes used can all be changed via `GameControllerOptions`/`ServerOptions`. Furthermore, the assembly prefix can be changed via the [Content Manifest](./content-manifests.md) as well (but see below).

If the game modules depend on other libraries (e.g. database libraries), these will also be loaded from the `/Assemblies/` directory when .NET requests them.

```admonish failure
It is not possible to ship additional non-module libraries for launcher/hub clients, as these modules do not support sandboxing.

Furthermore, in this environment it is not allowed to change the assembly prefix/root namespace for similar reasons.
```

## Content start

While developing, you can directly run Robust's projects[^rtproj] (`Robust.Client` and `Robust.Server`) from your IDE. This will load your code from its appropriate build directory. There's just one problem though: your IDE doesn't realize that your game code should also be recompiled, and won't do it automatically. This means you may accidentally run with outdated code!

To work around this, you can make your main client and server projects an executable, and then have the entrypoint immediately call `ContentStart.Start()` in Robust. Now you can launch your content projects directly, so your IDE should make sure everything is fully built.

Client code:

```cs
using Robust.Client;
using System;

namespace Content.Client;

internal static class Program
{
    // Note: entrypoint MUST have [STAThread]!
    [STAThread]
    public static void Main(string[] args)
    {
        ContentStart.Start(args);
    }
}
```

Server code:

```cs
using Robust.Server;
using System;

namespace Content.Server;

internal static class Program
{
    public static void Main(string[] args)
    {
        ContentStart.Start(args);
    }
}
```

The code here will not be executed when running your game from Space Station 14's launcher or when properly packaged. This is purely a development aid.

[^rtproj]: This only works when running Robust's executables from **Robust's** build directory, e.g. `RobustToolbox/bin/Client/Robust.Client`. It will not work if you run `bin/Content.Client/Robust.Client` from your project, as RT will fail to load the resource paths!
