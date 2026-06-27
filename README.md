# DayZ Server — Cherna-Demo (KRONJON)

A modded DayZ server running the vanilla Chernarus mission (`dayzOffline.chernarusplus`)
with the DayZ-Expansion suite. Mods are auto-updated from the Steam Workshop on every
restart via SteamCMD.

## Mods

All mods live in `@<Name>` folders in the server root. The launcher builds the `-mod`
list automatically from every folder starting with `@`, so adding a mod is just a matter
of dropping its folder in place (the auto-updater reads the Workshop ID from each mod's
`meta.cpp`).

| Folder | Mod | Workshop ID |
| --- | --- | --- |
| `@CF` | Community Framework (CF) — shared dependency | [1559212036](https://steamcommunity.com/sharedfiles/filedetails/?id=1559212036) |
| `@Community-Online-Tools` | Community Online Tools (admin/teleport/ESP tooling) | [1564026768](https://steamcommunity.com/sharedfiles/filedetails/?id=1564026768) |
| `@Dabs-Framework` | Dabs Framework — shared dependency | [2545327648](https://steamcommunity.com/sharedfiles/filedetails/?id=2545327648) |
| `@DayZ-Expansion-Animations` | DayZ-Expansion Animations | [2793893086](https://steamcommunity.com/sharedfiles/filedetails/?id=2793893086) |
| `@DayZ-Expansion-Bundle` | DayZ-Expansion Bundle (core features) | [2572331007](https://steamcommunity.com/sharedfiles/filedetails/?id=2572331007) |
| `@DayZ-Expansion-Licensed` | DayZ-Expansion Licensed | [2116157322](https://steamcommunity.com/sharedfiles/filedetails/?id=2116157322) |

> Each mod's `.bikey` is expected in `keys/` so clients can connect with signature
> verification on (`verifySignatures = 2` in `serverDZ.cfg`).

Workshop content folders are kept out of Git — only each mod's `meta.cpp` scaffold is
tracked (see `.gitignore`). The actual mod files come down via SteamCMD.

## Starting the server — `start_autoMods.bat`

Double-click or run `start_autoMods.bat`. It:

1. **Loads `.env`** from the script directory (if present), importing any
   `NAME=value` lines as environment variables.
2. **Builds the mod list** by scanning for `@*` folders and joining them with `;`.
3. **Enters a restart loop:**
   - If updates are enabled, it updates all mods first (see below).
   - Launches `DayZServer_x64.exe` and **waits** for it to exit (`/wait`).
   - On exit, waits 3 seconds and starts again — so a crash or a scheduled restart
     brings the server right back up.

Fixed launch settings (edit at the top of the `.bat` if needed):

| Setting | Value |
| --- | --- |
| Server name (window title) | `KRONJON-Cherna-Demo` |
| Port | `2302` |
| Config | `serverDZ.cfg` |
| Profiles folder | `config` |
| CPU cores | `2` |
| Launch flags | `-adminlog -netlog -freezecheck` |

To stop the loop, close the batch window (not just the server window), or the loop will
relaunch the server.

## `.env` configuration

`.env` is optional — the script has sane defaults for everything. Create a `.env` file in
the server root to override machine-specific settings or Steam credentials. **`.env` is
git-ignored; never commit credentials.**

Variables honored from `.env`:

| Variable | Default | Purpose |
| --- | --- | --- |
| `ENABLE_WORKSHOP_UPDATES` | `1` | Master switch. Set `0` to skip all mod updates entirely. |
| `UPDATE_ON_RESTART` | `1` | Update mods before each launch in the restart loop. |
| `USE_STEAMCMD` | `1` | `1` = download via SteamCMD; `0` = copy from a local Workshop cache (`WORKSHOP_PATH`). |
| `WORKSHOP_PATH` | `E:\SteamLibrary\steamapps\workshop\content\221100` | Local Workshop cache used when `USE_STEAMCMD=0`. |
| `SKIP_MODS` | `_@Heatmap` | Space-separated folder names to skip during updates. |
| `SKIP_MOD_IDS` | `2854246756` | Space-separated Workshop IDs to skip during updates. |
| `USERNAME` | (anonymous) | Steam account name for SteamCMD login. If unset, logs in anonymously. |
| `PASSWORD` | (empty) | Steam password for that account. See Steam Guard note below. |

Example `.env`:

```dotenv
# Use a Steam account so non-anonymous workshop downloads work
USERNAME=my_steam_account
PASSWORD=my_steam_password

# Point at this machine's local workshop cache (only used if USE_STEAMCMD=0)
WORKSHOP_PATH=E:\SteamLibrary\steamapps\workshop\content\221100

# Don't touch these mods on update
SKIP_MODS=_@Heatmap
SKIP_MOD_IDS=2854246756
```

> **Note:** `STEAMCMD` (path to `steamcmd.exe`, defaults to `C:\steamcmd\steamcmd.exe`)
> and `WORKSHOP_APPID` (`221100`, DayZ) are set inside the `.bat` itself rather than read
> from `.env`. Edit the script directly if SteamCMD lives elsewhere.

## How SteamCMD updates work

When `ENABLE_WORKSHOP_UPDATES=1` and `UPDATE_ON_RESTART=1`, the loop refreshes mods before
each server launch.

**SteamCMD mode (`USE_STEAMCMD=1`, default):**

1. Verifies `steamcmd.exe` exists (skips updating with a warning if not).
2. Reads each `@*` / `_@*` folder's `meta.cpp` to get its `publishedid`, skipping anything
   in `SKIP_MODS` or `SKIP_MOD_IDS`.
3. Writes a single SteamCMD runscript that logs in and queues a
   `workshop_download_item 221100 <id> validate` for every mod, then runs **one** SteamCMD
   session for all of them.
4. Checks for Steam Guard sentry / `config.vdf` token and warns if missing.
5. **Mirrors** each downloaded mod from SteamCMD's
   `steamapps\workshop\content\221100\<id>` into the server's `@<Mod>` folder using
   `robocopy /MIR` (so removed files are pruned too).

**Workshop-cache mode (`USE_STEAMCMD=0`):**

Skips downloading and just mirrors mods from an existing local Workshop cache
(`WORKSHOP_PATH`) into the `@<Mod>` folders. Useful when another Steam client on the box
already keeps the mods updated.

**Steam Guard / login:** Login is anonymous unless `USERNAME` is set. For a real account,
SteamCMD relies on a cached login token (`config\config.vdf`) and Steam Guard sentry file
(`ssfn*`) next to `steamcmd.exe`. Run SteamCMD manually once to log in and answer the Steam
Guard prompt so these get cached — after that the automated updates run unattended.

## Layout

```
.
├── start_autoMods.bat          # Launcher + restart loop + auto-updater
├── serverDZ.cfg                # Core server config
├── .env                        # Local overrides / credentials (git-ignored)
├── @CF, @Community-Online-Tools, @Dabs-Framework, @DayZ-Expansion-*   # Mods
├── keys/                       # Mod .bikey files for signature verification
└── mpmissions/dayzOffline.chernarusplus/   # Mission (loot, events, types, globals…)
```

Live save data under `mpmissions/**/storage_1/` is game-generated and git-ignored.
