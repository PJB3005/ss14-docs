# Configuration system

Robust features a powerful and flexible configurtaion system for storing user settings, server configuration, development overrides, etc.

The primary API is accessible through the `IConfigurationManager` [service](../code-structure/ioc.md).

## The basics of a CVar

At its core, the configuration system stores a flat dictionary of *CVars*. Each CVar has a name, type, etc, and can be manipulated through `IConfigurationManager`:

```cs
var val = cfg.GetCVar<int>("foo.bar");
val += 1;
cfg.SetCVar("foo.bar", val);

cfg.SaveToFile();
```

CVars must be *registered* before they can be used. This is typically done by putting a `CVarDef<T>` in a static class with the `[CVarDefs]` attribute, like so:

```cs
[CVarDefs]
public static class MyCVars {
    public static readonly CVarDef<int> FooBar = CVarDef.Create("foo.bar", defaultValue: 20, CVar.ARCHIVE);
}
```

`CVarDef`s can be used as a type-safe alternative to passing raw strings into CVar names, allowing us to rewrite the earlier code sample like so:

```cs
var val = cfg.GetCVar(MyCVars.FooBar);
val += 1;
cfg.SetCVar(MyCVars.FooBar, val);
```

## Configuration file format

CVars are typically (though not exclusively) stored in a configuration file in [TOML](https://toml.io/en/) format. Periods in CVar names allow grouping into tables. For example:

```toml
[net]
# Sets net.tickrate and net.port
tickrate = 30
port = 1212

[game]
# Sets game.desc
desc = "My own server!"
```

On the client, the configuration file is stored at `client_config.toml` in the [User Data Directory](./user-data-directory.md). On the server, the configuration file is stored next to the executable as `server_config.toml` by default, but a different path can be specified by the `--config-file` [command-line argument](../command-line-and-environment.md).

## CVar registrations in detail

Registering a CVar specifies its name, type, default value, and flags.

The **name** of a CVar is used primarily in config files, the debug console, and so on. You should ideally not be typing out the literal string anywhere else in the code, instead preferring to use `CVarDef<T>` to refer to the CVar in a type-safe manner. 

The **type** of a CVar determines what type of data is being stored. Only simple types can be used for CVars: `bool`, `string`, `int`, `float`, enums, and similar. You cannot natively have a "list" value or similar in a CVar.

The **default value** is the value that gets read when the CVar has not been modified, be that through config file, overrides, or just code setting the value. This value can be replaced later by calling `IConfigurationManager.OverrideDefault()`.

**Flags** are additional important additional settings for the CVar. These flags can be combined with the `|` operator, and are as follows[^additionalflags]:
* `CVar.ARCHIVE`: Indicates that the server should be stored to config file if it is saved. This should be used for user preferences on the client.[^serverarchive]
* `CVar.CLIENTONLY` and `CVar.SERVERONLY`: Indicates that the CVar should only exist on either the client or server. This is useful when a CVar is in a `[CVarDefs]` in shared code, where only one of the two sides should actually register it.
* `CVar.REPLICATED` along with either `CVar.CLIENT` or `CVar.SERVER`: Makes the server automatically synchronized over the network, and gives either the client or the server control. Unlike `CLIENTONLY` and `SERVERONLY`, both sides are *aware* of the CVar. See below for details.
* `CVar.NOT_CONNECTED`: Only allows the CVar to be changed while the client is not connected to a server. This does *not* prevent a client from setting a value and *then* connecting to a server!
* `CVar.CONFIDENTIAL`: Indicates that this CVar contains something important like a password or an API key, and makes various dev tools not show previews of the value to avoid accidental leaks.

CVars are usually registered with the `[CVarDefs]` attribute, as shown in an earlier example. Another, more old-fashioned option is to use `IConfigurationManager.RegisterCVar()` manually. This has the ability to register CVars dynamically based on custom string names, but is of course less convenient.

Registrations does not need to happen before the configuration file is loaded: the engine internally tracks unknown CVars, but these cannot be accessed until they are registered.

## Getting notified when a CVar changes

Using `IConfigurationManager.OnValueChanged()` you can register a callback to be fired whenever a CVar is changed. This is highly recommended if practical, as it makes live modification and testing much easier. It can also be used to easily cache the value if very frequently accessed.

The passed callback receives the new value of the CVar. Optionally, you can specify `invokeImmediately: true` to immediately run the callback once with the latest value, making typical caching scenarios more concise:

```cs
private int _cache;

public void Initialize()
{
    _cfg.OnValueChanged(MyCVar, val => _cache = val, invokeImmediately: true);
}
```

The above example is acceptable for global services that last until the end of the program, but would memory leak for more shorter lived examples like a UI control. In these cases, you should use `UnsubValueChanged()` when you are done caring about the CVar. Remember that C# lambda expressions cannot easily be used for this purpose, so use named methods instead:

```cs
public void Initialize()
{
    _cfg.OnValueChanged(MyCVar, OnMyCVarChanged);
}

public void Shutdown()
{
    _cfg.UnsubValueChanged(MyCVar, OnMyCVarChanged);
}

private void OnMyCVarChanged(int value)
{
    // ...
}
```

When in an `EntitySystem`, you can use `Subs.CVar()` instead. This will automatically call `UnsubValueChanged()` when the entity system shuts down:

```cs
public sealed partial class MyEntitySystem : EntitySystem
{
    [Dependency]
    private IConfigurationManager _cfg = null!;

    public override void Initialize()
    {
        base.Initialize();

        Subs.CVar(_cfg, CVars.MyCVar, val => { /* ... */ });
    }
}
```

## CVar replication

CVars can be automatically replicated across the network by giving them the flag `CVar.REPLICATED` and one of `CVar.CLIENT` or `CVar.SERVER`. This can be extremely convenient for synchronizing server configuration to the client, or client preferences to the server.

When `CVar.SERVER | CVar.REPLICATED` is set, the server is in control of the CVar, and automatically sends it to all connected clients. On clients, the current CVar value can be retrieved by simply calling `GetCVar()` on the appropriate CVar. See also `IClientNetConfigurationManager.ReceivedInitialNwVars` to receive an event when the first set of replicated CVars arrive on the client.

When `CVar.CLIENT | CVar.REPLICATED` is set, each client is in control of its own CVar. Because you'll likely have many clients connected to a server at once, you must use `INetConfigurationManager.GetClientCVar()` to get the CVar of a specific client instead of the regular `GetCVar()`. This API is also accessible to shared code: if called on the client, the local client's value is always returned.

CVar replication happens automatically, and no additional functions have to be called after modifying a CVar.

Note that CVar replication is synchronized with simulation timing: if a replicated CVar is changed on tick X on the server, the client will (ideally) try to apply the CVar change on the same tick.

## CVar rollback

To allow safely handling "risky" options, the configuration system can be told to "snapshot" a set of CVars. These can later be rolled back on command or if the engine is restarted without explicit confirmation.

```admonish example
You know when you change display settings in a game, and it gives you a 30 second timer to confirm they're correct? Yeah, that.
```

To use, call `IConfigurationManager.MarkForRollback()` **before** modifying a CVar. The CVar can then be rolled back by restarting the application or calling `ApplyRollback()`. To commit the CVar changes, call `UnmarkForRollback()` and save the configuration file.

```cs
cfg.MarkForRollback(CVars.DisplayUIScale);
cfg.SetCVar(CVars.DisplayUIScale, value);

if (await ShowConfirmation())
{
    cfg.UnmarkForRollback(CVars.DisplayUIScale);
    cfg.SaveToFile();
}
else
{
    cfg.ApplyRollback();
}
```

```admonish question "Just don't save the config file until you know it's fine!"
The configuration system is a shared singleton for *every* system in the game. Having to explicitly save is a *performance optimization*, not a commitment point. It must be assumed that at any point, any other part of the game may trigger a save.
```

## CVar overrides

Via [startup parameters](../command-line-and-environment.md), CVar values can be "overriden". This takes priority over the default CVar value and the configuration file, but the overriden value will *not* get saved to the configuration file.

If you call `SetCVar()` on a CVar that is being overriden, the override gets removed in favor of the assigned value.

## Debug console

The in-game debug console can read and write CVars via the `cvar` command. Please see the command's help documentation for syntax.

Remember that the `cvar` command exists both on the client and the server; if you want to edit a server CVar from a (privileged) client, you need to prefix it with `>` or `sudo`, e.g. `sudo cvar game.desc "And now for something different"`

## Thread safety

Accessing the configuration is thread safe: it is legal to have concurrent readers and writers from any thread.

Note however that that change callbacks are always fired on the thread setting a CVar, so modifying CVars on non-main threads may cause thread-unsafe callbacks to break. They are also not guaranteed to be fired 

[^additionalflags]: There are additional flags defined in the `CVar` type such as `CHEAT` and `NOTIFY`, but these do not do anything at the moment.
[^serverarchive]: While you can save the config file on the server too, that's very poor hygiene and you probably shouldn't.
