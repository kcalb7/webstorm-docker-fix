# webstorm-docker-fix

Makes the Docker integration of **WebStorm 2023.1** work again with **Docker Engine 29+**.

After upgrading Docker, the Services tool window fails to connect with:

```
Status 400: client version 1.24 is too old. Minimum supported API version is 1.40, please upgrade your client to a newer version
```

JetBrains fixed this in 2025.2.5 and 2025.3 ([IJPL-217878](https://youtrack.jetbrains.com/issue/IJPL-217878)), and no fix has been announced for older versions. If you can upgrade the IDE, do that instead. This script is for setups that need to stay on 2023.1 (for example, a perpetual fallback license).

## Why it happens

The bundled Docker plugin opens the connection with a client pinned to API **1.24**, only to ask the daemon for its version. After that it switches to the negotiated version, capped at **1.40**. Docker Engine 29 rejects any request below its minimum API version, so the very first call fails and the plugin never gets past the handshake.

## What the script does

1. Reads the bundled plugin jar (`plugins/clouds-docker-impl/lib/clouds-docker-impl.jar`) from your own installation.
2. Patches `DockerAgentImpl$connect$2.class` so the handshake references `RemoteApiVersion.VERSION_1_40` instead of `VERSION_1_24`. The two names have the same length, so this is a 2-byte change in the class constant pool, and the script checks that the constant is used only as that field reference before touching it.
3. Bumps `<version>` in `plugin.xml` (for example `231.9423.2` to `231.9423.2.1`) so the IDE prefers the patched copy over the bundled one.
4. Rewrites the jar without `__index__`. JetBrains jars ship a precomputed index with fixed entry offsets, so any rewrite invalidates it and the IDE rejects the jar with "contains invalid plugin descriptor". Third-party plugins don't have this file either.
5. Opens the result with the IDE's own zip reader (`ImmutableZipFile`) to validate it, keeps a backup with a checksum, and installs it into the user plugins directory. The IDE installation itself is never modified.

No JetBrains code is included in this repository. The patch is built locally from the copy you already have.

## Requirements

- Linux
- WebStorm 2023.1.x (tested with 2023.1.6, build 231.9423.2, installed via snap)
- `python3`

## Usage

```bash
install -m 755 webstorm-docker-fix ~/.local/bin/
webstorm-docker-fix
```

Then restart WebStorm. Under Settings → Plugins, Docker should show version `231.9423.2.1`.

| Command | What it does |
| --- | --- |
| `webstorm-docker-fix` / `apply` | Restores the backup if it matches the current build, otherwise builds the patch from the bundled jar |
| `webstorm-docker-fix rebuild` | Builds the patch again from the bundled jar, ignoring the backup |
| `webstorm-docker-fix status` | Shows the current build, the installed plugin and the backups |
| `webstorm-docker-fix undo` | Removes the patched copy, so the IDE goes back to the original plugin |

Backups are kept in `~/.local/share/webstorm-docker-fix/`, one per IDE build.

For a Toolbox or tarball installation, point `IDE_HOME` at the IDE directory, the one that contains `product-info.json`:

```bash
IDE_HOME=/path/to/WebStorm-231.9423.2 webstorm-docker-fix
```

If the IDE updates to a different build, the patched copy no longer matches and the IDE falls back to the original plugin. Run the script again and restart.

## Alternative without patching

You can instead lower the daemon's minimum API version by setting `DOCKER_MIN_API_VERSION=1.24` in the `dockerd` environment (for example with a systemd override). This is the workaround suggested in [IJPL-217878](https://youtrack.jetbrains.com/issue/IJPL-217878), but it relaxes the requirement for every client of the daemon, not only the IDE.

## Disclaimer

Not affiliated with JetBrains or Docker. You modify your local plugin copy at your own risk. `webstorm-docker-fix undo` reverts it.

The MIT license covers only this script. Your use of WebStorm and its bundled plugins, including the patched copy, remains governed by your own JetBrains license agreement.
