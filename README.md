# judo

`judo` installs self-contained Linux applications distributed as archives.

It extracts the archive, picks the most likely main executable, installs the app under `/opt/<AppName>`, creates a user desktop launcher, and creates a user command symlink in `~/.local/bin`.

## What It Does

Given:

```bash
judo [--force] <tar-file-or-url> <AppName>
```

it will:

1. Resolve name conflicts for `/opt/<AppName>`, `~/.local/share/applications/<appname>.desktop`, and `~/.local/bin/<appname>` (overwrite, new name, or cancel).
2. If input is an `http://` / `https://` URL, download it with `curl -L` first.
3. Prepare a clean extraction directory at `/tmp/<AppName>-extract`.
4. Extract the archive payload into that temp tree (tar/zip/package payload extraction depending on format).
5. Score executable candidates and choose the best match for `<AppName>`.
6. Find an icon candidate if one exists.
7. Choose the staging source tree from extracted content.
8. Stage files under `/opt/.<AppName>.staging.<pid>`.
9. Re-score executable from staged content.
10. Reuse or generate a desktop file and set managed keys (`Name`, `Exec`, `Icon`, etc.).
11. Atomically replace `/opt/<AppName>` (backing up previous install to `/opt/.<AppName>.backup.<timestamp>` when present).
12. Create/update `~/.local/share/applications/<appname>.desktop` and `~/.local/bin/<appname>`.

Staging in `/opt/.<AppName>.staging.<pid>` is intentional:

- It keeps staging on the same filesystem as `/opt/<AppName>`, so the final `mv` is an atomic rename instead of a cross-filesystem copy.
- It reduces half-installed states if something fails before final replacement.
- It is cleaned up automatically: on success it is renamed into place, and on failure/exit the cleanup trap removes the staging directory.

`/tmp` is still used for extraction, but not for final staging/swap.

## Requirements

- Linux with Bash
- `sudo` privileges (writes under `/opt` and optionally `/usr/share/pixmaps`)
- `tar`
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

You can pass either a local archive path or an `http://` / `https://` URL. URL inputs are downloaded first via `curl -L`, then installed.

For package files (`.deb`, `.rpm`), `judo` extracts payload contents and installs from that tree. It does not run package-manager maintainer scripts or resolve dependencies.

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
judo [--force] <tar-file-or-url> <AppName>
```

Examples:

```bash
judo ~/Downloads/Telegram.tar.xz Telegram
judo ~/Downloads/obsidian.tar.gz Obsidian
judo ~/Downloads/krita-5.2.9-x86_64.appimage.zip Krita
judo ~/Downloads/Slicer-linux-amd64.deb Slicer
judo ~/Downloads/example.tar.zst Example
judo ~/Downloads/example.rpm Example
judo --force ~/Downloads/Telegram.tar.xz Telegram
```

## Name Collision Handling

Before extraction, `judo` checks for existing generated targets:

- `/opt/<AppName>`
- `~/.local/share/applications/<appname>.desktop`
- `~/.local/bin/<appname>` (symlink or regular file)

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

It de-prioritizes likely helper binaries (`test`, `debug`, `helper`, `daemon`).

## Copy/Install Modes

After the best executable is found:

- Nested layout: if executable is inside a subdirectory under extraction root, `judo` copies that subdirectory contents
- Flat layout: otherwise, `judo` copies the full extracted tree

## Desktop File Behavior

If a vendor `.desktop` file is found:

- It is reused
- These keys are replaced: `Version`, `Name`, `Exec`, `Icon`, `Terminal`, `Type`, `Categories`
- Other vendor keys are preserved (for example `StartupWMClass`, `MimeType`, `Actions`)

If no usable vendor file exists, `judo` generates a minimal desktop entry.

`Exec` points to the final installed executable in `/opt/<AppName>`. When possible, vendor `Exec` suffix arguments/field codes are preserved.

## Icon Behavior

- First icon candidate (`.png`, `.svg`, `.xpm`) is selected
- Best-effort copy to `/usr/share/pixmaps/<icon>`
- Desktop entry uses icon basename when available; otherwise falls back to `Icon=<AppName>`

## Re-running Safely

Re-running for the same app is supported. Existing installs are moved to timestamped backups under `/opt` before replacement.

## Output Paths Summary

For `judo app.tar.xz MyApp`:

- Install: `/opt/MyApp`
- Temp extract: `/tmp/MyApp-extract`
- Desktop: `~/.local/share/applications/myapp.desktop`
- Symlink: `~/.local/bin/myapp`
- Backup (if replaced): `/opt/.MyApp.backup.YYYYMMDD-HHMMSS`

## Troubleshooting

- `No executable found`
  - Archive likely lacks executable bit(s) or contains only installer payloads.
- `This archive looks like a Windows-only bundle`
  - The archive contains Windows binaries like `.exe`/`.dll` but no native Linux executable. Use a Linux release, AppImage, or source archive instead.
- `Unsupported format`
  - Repackage as supported tar format.
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
