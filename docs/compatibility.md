# Compatibility with the SolveCraft name

SolidCraft was first called SolveCraft, and builds up to 0.4 kept that name in their
identifiers (binaries, crates, file extension, folders) after the displayed name changed. Since 0.5.0 the product uses the SolidCraft name
everywhere: crates `solidcraft-*`, binaries `solidcraft` and `solidcraft-cli`, `.solidcraft`
designs, the `"format": "solidcraft/N"` tag, `SOLIDCRAFT_*` environment variables, settings
and recovery folders, the MCP server name, docs, CI and packaging.

The old name survives **only** in the shims below (plus their tests, this file, and changelog /
release notes). `git grep -i solvecraft` must list nothing else. When you add or remove a
shim, update this file in the same PR.

## 1. `.solvecraft` design files

New designs are saved as `.solidcraft`. Files with the old extension still open everywhere:

| Path in | Where |
|---|---|
| Extension → design format | `crates/io/src/lib.rs` (`LEGACY_DESIGN_EXT`, format detection) |
| Open dialog filters (desktop) | `apps/solidcraft/src/main.rs` (`services`) |
| Drag-drop, recent files, `doc.open`, MCP `open`, CLI `eval`/`run` | go through the same format detection in `solidcraft_io` |
| Browser file picker | `apps/solidcraft-web/src/files.rs` (`accept`) |
| Windows | `packaging/windows/solidcraft.wxs`: the `.solvecraft` extension on the design ProgId and in `SupportedTypes` |
| macOS | `packaging/macos/Info.plist.in`: `solvecraft` in the design document type's extensions |
| Linux / FreeBSD / Flatpak | `packaging/linux/ai.storyteller.solvecraft.mime.xml`: `*.solvecraft` glob and the `application/x-solvecraft` alias |

Recent-files entries that point at a `.solvecraft` file keep working: they are plain paths and
open through the same detection. Tests: the legacy-extension tests in `crates/io` and the CLI
test that opens a `.solvecraft` copy of a corpus design.

## 2. The `solvecraft/N` format tag

Designs record their format as `"format": "solidcraft/N"`. Files saved by builds up to 0.4 say
`"format": "solvecraft/N"`; `crates/doc/src/format.rs` (`version_of`) reads both prefixes and
upgrades them as usual (the file is re-tagged `solidcraft/N` when saved).

Not backward-compatible: builds up to 0.4 don't know the new tag and refuse files saved by
0.5.0 or later. Test: `version_of` accepts both prefixes; legacy-tagged designs load and are
re-tagged on save.

## 3. `SOLVECRAFT_*` environment variables

Every runtime variable is read through one helper, `solidcraft_geom::env` (`var`, `var_os`):
it reads `SOLIDCRAFT_<NAME>` and falls back to `SOLVECRAFT_<NAME>` when the new one is unset.
The shell entry points do the same: `packaging/linux/AppRun`
(`SOLIDCRAFT_NO_DESKTOP_INTEGRATION`) and `packaging/env.sh` (`SOLIDCRAFT_VERSION`,
`SOLIDCRAFT_BUILD_DATE`). Tests: the helper's unit tests (new wins, old is the fallback,
neither → unset).

## 4. Settings, preferences, autosave and recovery folders

New folders use the SolidCraft name:

| | Linux | macOS | Windows |
|---|---|---|---|
| Preferences, graphics settings | `~/.local/share/solidcraft` | `~/Library/Application Support/SolidCraft` | `%APPDATA%\SolidCraft\data` |
| Recovery (autosave) | `…/solidcraft/recovery` | `…/SolidCraft/recovery` | `%LOCALAPPDATA%\SolidCraft\recovery` |

On first launch, if the new folder doesn't exist but the old one (`solvecraft` / `SolveCraft`)
does, it is **copied** to the new location; the old folder is never deleted or changed. The
window and UI state that eframe kept under `ai.storyteller.solvecraft` is copied the same way.
The thumbnail cache is not migrated; it is rebuilt. The copy helper lives in `solidcraft_io`
and is used by the desktop app and `solidcraft_engine::recovery`. Tests: copy when only the
old folder exists, no copy when the new one exists, the old folder is left intact.

## 5. `solvecraft-cli` alias

A tiny `solvecraft-cli` binary (built from `apps/solidcraft-cli`) ships in every package for at
least one release (0.5.x). It prints a one-line deprecation note to stderr and runs the
`solidcraft-cli` next to it with the same arguments, stdin/stdout and exit code, so scripts
and MCP client configs that call `solvecraft-cli` keep working. Test: the alias forwards
`--version` and prints the note on stderr only (stdout stays clean for MCP).

There is no alias for the desktop binary: launchers, file associations and Start-menu entries
installed by the packages point at `solidcraft`.

## 6. MCP

`serverInfo.name` is now `solidcraft`. The name a client shows is chosen by the user when the
server is added, so an existing setup such as

```sh
claude mcp add solvecraft -- solvecraft-cli mcp
```

keeps working (through the alias above). New setups use:

```sh
claude mcp add solidcraft -- solidcraft-cli mcp
```

Switch before the alias is removed.

## Package IDs

| Package | Decision | Why |
|---|---|---|
| Flatpak app ID / Linux desktop ID `ai.storyteller.solvecraft` | **kept** | A new Flatpak ID is a different app: existing installs would stop receiving updates. The `.desktop`, metainfo and MIME file names and the Wayland `app_id` must match it, so they keep it too. |
| Flatpak remote file | `solidcraft.flatpakrepo`, plus a legacy `solvecraft.flatpakrepo` copy | Old links keep working; existing remotes point at the repo URL, which is unchanged. |
| macOS bundle ID | changed to `ai.storyteller.solidcraft` | No updater depends on it; the app is drag-installed. macOS may ask again on first launch (unsigned builds). |
| MSI `UpgradeCode` | **kept** | So the new MSI upgrades the old install in place. Product name is SolidCraft; the default folder becomes `Program Files\SolidCraft` (the major upgrade removes the old folder). A custom install folder isn't remembered across this one upgrade. |
| deb / rpm package name | renamed to `solidcraft`; deb `Replaces`/`Conflicts`/`Provides: solvecraft`, rpm `Obsoletes`/`Provides: solvecraft` | Installing the new package replaces the old one instead of installing side by side. |
| AppImage | `solidcraft-<version>-linux-<arch>.AppImage`; the release also carries a copy of its `.zsync` under the old `solvecraft-…` name | AppImages already installed look for the old-named `.zsync`, so AppImageUpdate keeps finding updates. |

## macOS

Releases before 0.4 were installed as `SolveCraft.app`. To update, quit the app, drag
`SolidCraft.app` from the DMG onto Applications, move `SolveCraft.app` to the Trash, and
replace any Dock shortcut with the old name (open `SolidCraft.app`, then
**Options > Keep in Dock**). Designs and settings are kept (see 4).

## Flatpak remotes added from the former website address

`flatpak remote-modify --user --url=https://bherbruck.github.io/solidcraft/flatpak/repo/ solvecraft`
(use your remote's name), then `flatpak update --user`.
