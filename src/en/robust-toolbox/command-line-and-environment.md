# Command line and environment variables

RobustToolbox's behavior can be changed at startup through command-line arguments and environment variables.

## Client

### Command-line arguments

| **Argument** | **Description** |
|--------------|-----------------|
| `--connect` | Makes the client automatically start connecting to a server as soon as it is done initializing. The default address to connect to is `localhost`, but this can be specified by `--connect-address`. |
| `--connect-address <address>` | The address to automatically connect to if `--connect` is passed. This must be the actual game server UDP traffic port, **not** the `ss14://` or `ss14s://` address typically shown to users. The value is parsed as URI, and the protocol must be `udp://` if specified. The default port is 1212. |
| `--cvar <key>=<value>` | Overrides a CVar value, superseding even the value in the config file (without writing it back). |
| `--headless` | Run the client in a "headless" mode. Audio, graphics, and similar systems are disabled. Other functionality (including the ability to connect to servers) remains present. |
| `--help` | Print help text and exit. |
| `--launcher` | Indicates that this client is being ran from a launcher. It implies `--connect` and sets a flag in `IGameController.LaunchState` that games should check to skip unnecessary menus. |
| `--loglevel <sawmill>=<level>` | Sets `ISawmill.Level` on the specified sawmill name to the provided value, which must be one of `null`, `Verbose`, `Debug`, `Info`, `Warning`, `Error`, or `Fatal`. |
| `--mount-dir <directory>` | Mounts the specified directory into the resource VFS. It is not possible to specify a mount point, so it is always mounted at the root. |
| `--mount-zip <archive>` | Mounts the specified zip file into the resource VFS. It is not possible to specify a mount point, so it is always mounted at the root. |
| `--self-contained` | Makes the [user data directory](saving-configuration/user-data-directory.md)stored next to the executable as `user_data/`. |
| `--ss14-address <address>` | Specifies the `ss14://` or `ss14s://` address being connected to with `--connect` and `--connect-address`. This value is not used by the engine itself, but is accessible to content code through `IGameController.LaunchState` and may be used for user-facing display purposes. |
| `--username <value>` | Specifies the username the client should request from servers. This is only useful when connecting to a server without authentication, though the value may be retrieved via `IBaseClient.PlayerNameOverride` before that happens. |

Additionally, positional arguments prefixed with `+` are intrepreted as console commands to execute after initialization has finished. If arguments need to be provided to the executed command, make sure the entire command is a single argument on the command line, e.g. `./Robust.Client.exe "+launchauth MyUsername"`

### Environment variables

*Note that Robust is a .NET application, and [the .NET runtime has its own environment variables](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-environment-variables) that may be useful for performance tuning or debugging.*

| **Variable** | **Description** |
| ------------ | --------------- |
| `ROBUST_AUTH_ALLOW_HWID` | Whether to allow sending HWID information to servers. Defaults to `1`, but can be disabled by specifying `0`. |
| `ROBUST_AUTH_PUBKEY` | Expected public key of of the game server to connect to. If not correct, the client will redial instead of connecting. |
| `ROBUST_AUTH_SERVER` | API address for the authentication server to use. |
| `ROBUST_AUTH_TOKEN` | Authentication token to authenticate with. |
| `ROBUST_AUTH_USERID` | UserID to use when authenticating. |
| `ROBUST_CVARS` | **Deprecated, prefer using `ROBUST_CVAR_` instead.** Specifies a set of [CVar overrides](./saving-configuration/configuration.md) in format `a=1;b=2;...`. It is not possible to escape the `;` delimiter.  |
| `ROBUST_CVAR_<name>` | Overrides a single CVar by name. A double `_` is replaced by `.` in the CVar name, so for example `ROBUST_CVAR_foo__bar` would set the CVar `foo.bar`. |
| `ROBUST_DISABLE_SANDBOX` | If `1`, skip [sandbox](./content-loading/sandboxing.md) checks on loaded content assemblies. |
| `ROBUST_INTEGRATED_GPU` | Unless set to `1`, the client attempts a hack to force using the dedicated GPU on Nvidia Optimus laptops. This will likely be removed in the future whenever renderer rewrite happens. |
| `ROBUST_NUMERICS_AVX` | Set to `true` to enable AVX instructions in `Robust.Shared.Maths.NumericsHelpers`, if supported by hardware. |
| `ROBUST_SOUNDFONT_OVERRIDE` | Specifies a MIDI soundfont to load after the OS soundfonts. See [Soundfont loading behavior](./audio/midi.md#soundfont-loading-behavior) for details. |
| `SS14_LAUNCHER_APPDATA_NAME` | Used for some development tools that directly interface with the launcher, see launcher documentation for the same environment variable for details. |

## Server

### Command-line arguments

| **Argument** | **Description** |
|--------------|-----------------|
| `--config-file <path>` | Specifies file path of the [configuration file](./saving-configuration/configuration.md) to use. |
| `--cvar <key>=<value>` | Overrides a CVar value, superseding even the value in the config file (without writing it back). |
| `--data-dir <path>` | Specifies path to the [user data directory](./saving-configuration/user-data-directory.md) to use. |
| `--help` | Print help text and exit. |
| `--loglevel <sawmill>=<level>` | Sets `ISawmill.Level` on the specified sawmill name to the provided value, which must be one of `null`, `Verbose`, `Debug`, `Info`, `Warning`, `Error`, or `Fatal`. |
| `--mount-dir <directory>` | Mounts the specified directory into the resource VFS. It is not possible to specify a mount point, so it is always mounted at the root. |
| `--mount-zip <archive>` | Mounts the specified zip file into the resource VFS. It is not possible to specify a mount point, so it is always mounted at the root. |

Additionally, positional arguments prefixed with `+` are intrepreted as console commands to execute after initialization has finished. If arguments need to be provided to the executed command, make sure the entire command is a single argument on the command line, e.g. `./Robust.Server.exe "+foo bar"`

### Environment variables

*Note that Robust is a .NET application, and [the .NET runtime has its own environment variables](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-environment-variables) that may be useful for performance tuning or debugging.*

| **Variable** | **Description** |
| ------------ | --------------- |
| `ROBUST_CVARS` | **Deprecated, prefer using `ROBUST_CVAR_` instead.** Specifies a set of [CVar overrides](./saving-configuration/configuration.md) in format `a=1;b=2;...`. It is not possible to escape the `;` delimiter.  |
| `ROBUST_CVAR_<name>` | Overrides a single CVar by name. A double `_` is replaced by `.` in the CVar name, so for example `ROBUST_CVAR_foo__bar` would set the CVar `foo.bar`. |
| `ROBUST_NUMERICS_AVX` | Set to `true` to enable AVX instructions in `Robust.Shared.Maths.NumericsHelpers`, if supported by hardware. |
