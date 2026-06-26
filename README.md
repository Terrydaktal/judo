# judo

`judo` installs self-contained Linux applications from archives, package payloads, URLs, or local directories.

Current script version: `0.1.7`

It extracts or inspects the input, picks the most likely main executable, and then either installs the app under `/opt/<AppName>` for archive inputs or keeps the source tree in place for directory inputs. In both cases it creates a user desktop launcher and a user command symlink in `~/.local/bin`.

## What It Does

Given:

```bash
judo [--force] <source-archive-directory-or-url> <AppName>
judo uninstall <AppName>
```

Version:

```bash
judo --version
```

it will:

1. Resolve name conflicts for `~/.local/share/applications/<appname>.desktop` and `~/.local/bin/<appname>` in all cases, and also `/opt/<AppName>` for archive inputs.
2. If input is an `http://` / `https://` URL, download it with `curl -L` first and then continue using the downloaded archive file.
3. If input is a directory, use it directly as the source tree and skip archive extraction and `/opt` staging.
4. If input is a local archive path, or a URL that was just downloaded, prepare a clean extraction directory at `/tmp/<AppName>-extract`.
5. Extract that archive payload into the temp tree using the handler for its format (tar/zip/package payload extraction depending on format).
6. Score executable candidates and choose the best match for `<AppName>`.
7. Find an icon candidate if one exists.
8. For archive inputs, choose a staging source tree from extracted content and stage files under `/opt/.<AppName>.staging.<pid>`.
9. For archive inputs, re-score the executable from staged content and atomically replace `/opt/<AppName>`.
10. Show the top executable, desktop file, and icon candidates, auto-pick the top one for each, and ask for a final confirm/edit/cancel step.
11. If you edit, you can choose an executable, desktop file, or icon by number or by absolute path.
12. Reuse or generate a desktop file and set managed keys (`Name`, `Exec`, `Icon`, etc.).
13. Create/update `~/.local/share/applications/<appname>.desktop` and `~/.local/bin/<appname>`.
14. For `judo uninstall <AppName>`, remove the generated desktop file, launcher symlink, and any generated user `hicolor` icon for `<appname>`. Remove `/opt/<AppName>` only when that install directory exists; leave source trees outside `/opt` in place.

Staging in `/opt/.<AppName>.staging.<pid>` is intentional for archive inputs:

- It keeps staging on the same filesystem as `/opt/<AppName>`, so the final `mv` is an atomic rename instead of a cross-filesystem copy.
- It reduces half-installed states if something fails before final replacement.
- It is cleaned up automatically: on success it is renamed into place, and on failure/exit the cleanup trap removes the staging directory.

`/tmp` is still used for extraction, but not for final staging/swap. Directory inputs bypass `/opt` staging entirely.

## Requirements

- Linux with Bash
- `sudo` privileges for archive installs (writes under `/opt`)
- `tar`
- `file`
- `find`, `awk`, `grep`, `install`
- `unzip` or `bsdtar` (for `.zip`)
- `ar` (for `.deb`)
- `bsdtar` or `rpm2cpio` + `cpio` (for `.rpm`)
- `tar` with zstd support (for `.tar.zst` / `.tzst` / `.pkg.tar.zst`)

Recommended:

- `~/.local/bin` on your `PATH`

## Supported Archive Types

- `.tar`
- `.tar.gz` / `.tgz`
- `.tar.bz2` / `.tbz2`
- `.tar.xz` / `.txz`
- `.tar.zst` / `.tzst`
- `.pkg.tar.zst`
- `.zip`
- `.deb` (payload extraction mode)
- `.rpm` (payload extraction mode)
- `.AppImage` (single-file executable mode with metadata extraction)

You can pass either a local archive/AppImage path or an `http://` / `https://` URL. URL inputs are downloaded first via `curl -L`, then installed.

For package files (`.deb`, `.rpm`), `judo` extracts payload contents and installs from that tree. It does not run package-manager maintainer scripts or resolve dependencies.

For AppImage files, `judo` stages the AppImage itself as the executable under `/opt/<AppName>/<appname>.AppImage` and runs `--appimage-extract` in a temporary directory to discover bundled desktop files and icons.

## Installation

Clone/download this repository, then make script executable:

```bash
chmod +x ./judo
```

(Optional) Install globally:

```bash
sudo install -m 755 ./judo /usr/local/bin/judo
```

## Usage

```bash
judo [--force] <source-archive-directory-or-url> <AppName>
judo uninstall <AppName>
```

Examples:

```bash
judo ~/Downloads/Telegram.tar.xz Telegram
judo ~/Downloads/obsidian.tar.gz Obsidian
judo ~/Downloads/krita-5.2.9-x86_64.appimage.zip Krita
judo ~/Downloads/Slicer-linux-amd64.deb Slicer
judo ~/Downloads/example.tar.zst Example
judo ~/Downloads/example.rpm Example
judo ~/Downloads/App.AppImage App
judo ~/src/copyq copyq
judo --force ~/Downloads/Telegram.tar.xz Telegram
judo uninstall Telegram
```

## Name Collision Handling

Before installation work, `judo` checks for existing generated targets:

- `/opt/<AppName>`
- `~/.local/share/applications/<appname>.desktop`
- `~/.local/bin/<appname>` (symlink or regular file)
- `~/.local/share/icons/hicolor/{scalable,256x256}/apps/<appname>.*`

If conflicts exist:

- Interactive TTY: prompts for `yes` (overwrite), `new name`, or `cancel`
- Non-interactive: exits with error and asks you to rerun with `--force` or a different app name
- `--force`: proceeds and replaces conflicting targets

## Executable Selection Heuristics

`judo` scores executable files and prefers:

- Exact name matches to `<AppName>` (and case variants)
- Filenames containing `<AppName>`
- Non-hidden paths
- Paths under `bin/` or `app/`

It de-prioritizes likely helper binaries (`test`, `debug`, `helper`, `daemon`). Executable `.desktop` files are excluded from executable selection and shown only as desktop file candidates.

## Source Modes

After the best executable is found:

- Archive input: `judo` stages the extracted tree under `/opt/.<AppName>.staging.<pid>`, then renames it into `/opt/<AppName>`.
- Directory input: `judo` leaves the source tree where it is and only creates/replaces the desktop file and `~/.local/bin` symlink.

## Desktop File Behavior

If a vendor `.desktop` file is found:

- It is reused
- Its `Name=` is preserved for the start menu label
- These keys are replaced: `Version`, `Exec`, `Icon`, `Terminal`, `Type`, `Categories`
- Other vendor keys are preserved (for example `StartupWMClass`, `MimeType`, `Actions`)

If no usable vendor file exists, `judo` generates a minimal desktop entry. In an interactive terminal, it asks whether to use a different start menu name; otherwise it falls back to the supplied `<AppName>`.

`Name` controls the start menu label. `Exec` points to the selected executable. For archive inputs this is the executable under `/opt/<AppName>`; for directory inputs it is the executable inside the provided directory. When possible, vendor `Exec` suffix arguments/field codes are preserved. The supplied `<AppName>` still controls the symlink name, desktop filename, install directory, and generated icon name.

`judo` prints the desktop file candidates first, then the executable candidates, then the icon candidates. It auto-selects the best option in each section and then lets you confirm, edit, or cancel before it writes the final desktop file. If the selected desktop file has a usable `Icon=` entry, that icon is shown in the icon shortlist with the resolved path it came from.

## Icon Behavior

- `judo` scores image candidates (`.png`, `.svg`, `.xpm`) and prefers obvious app icons such as `logo.png`, `icon.svg`, or filenames matching the app name
- It avoids obvious test/doc/example assets like `tests/small16x16.png`, `docs/`, and screenshot thumbnails
- `judo` prints the desktop file, executable, and icon candidates in that order, auto-selects the top choice for each, and then stops for a final confirm/edit/cancel prompt
- In the edit step, you can override any of the three by number or by absolute path
- If a selected desktop file contains a resolvable `Icon=` value, that icon appears in the icon shortlist as a desktop-file-derived candidate with its resolved absolute path
- Archive input: copies the selected icon into the user `hicolor` icon theme (`~/.local/share/icons/hicolor/scalable/apps/<appname>.svg` for SVG, or `~/.local/share/icons/hicolor/256x256/apps/<appname>.<ext>` for raster icons) and the desktop file uses `Icon=<appname>`
- Directory input: the desktop file points to the icon where it already lives inside the source tree
- Vendor `Icon=<name>` entries are only preserved when that icon name resolves in the installed icon themes or pixmap directories
- If no icon is found, the desktop file falls back to `Icon=<AppName>`

## Re-running Safely

Re-running for the same app is supported. Archive installs are moved to timestamped backups under `/opt` before replacement. Directory installs simply replace the desktop file and `~/.local/bin` symlink.

## Output Paths Summary

For `judo app.tar.xz MyApp`:

- Install: `/opt/MyApp`
- Temp extract: `/tmp/MyApp-extract`
- Icon copy: `~/.local/share/icons/hicolor/scalable/apps/myapp.svg` or `~/.local/share/icons/hicolor/256x256/apps/myapp.<ext>`
- Desktop: `~/.local/share/applications/myapp.desktop`
- Symlink: `~/.local/bin/myapp`
- Backup (if replaced): `/opt/.MyApp.backup.YYYYMMDD-HHMMSS`

For `judo ~/src/MyApp MyApp`:

- Install: stays in `~/src/MyApp`
- Desktop: `~/.local/share/applications/myapp.desktop`
- Symlink: `~/.local/bin/myapp`

## Troubleshooting

- `No executable found`
  - Archive likely lacks executable bit(s) or contains only installer payloads.
- `This archive looks like a Windows-only bundle`
  - The archive contains Windows binaries like `.exe`/`.dll` but no native Linux executable. Use a Linux release, AppImage, or source archive instead.
- `Unsupported format`
  - Repackage as supported tar format.
- `Archive file type: ...`
  - `judo` prints the detected archive type before extraction. If it doesn't match the filename suffix, `judo` stops early and tells you the file is mislabeled or the wrong release artifact.
- App launches from terminal but not desktop
  - Inspect desktop file under `~/.local/share/applications/`.
- Command not found after install
  - Ensure `~/.local/bin` is in `PATH` and open a new shell.

## Security Notes

- This script installs and executes third-party binaries.
- Only use archives from trusted sources.
- Review extracted content when in doubt.

## License

No license file is currently included. Add one if you intend to distribute this project.
