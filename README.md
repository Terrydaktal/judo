# judo

`judo` installs self-contained Linux applications distributed as tar archives.

It extracts the archive, picks the most likely main executable, installs the app under `/opt/<AppName>`, creates a user desktop launcher, and creates a user command symlink in `~/.local/bin`.

## What It Does

Given:

```bash
judo [--force] <AppName> <tar-file-or-url>
```

it will:

1. Extract the archive to `/tmp/<AppName>-extract`.
2. Score executable candidates and choose the best match for `<AppName>`.
3. Stage files under `/opt/.<AppName>.staging.<pid>`.
4. Replace `/opt/<AppName>` (backing up previous install to `/opt/.<AppName>.backup.<timestamp>` when present).
5. Create/update `~/.local/share/applications/<appname>.desktop`.
6. Create/update `~/.local/bin/<appname>` symlink.

## Requirements

- Linux with Bash
- `sudo` privileges (writes under `/opt` and optionally `/usr/share/pixmaps`)
- `tar`
- `find`, `awk`, `grep`, `install`

Recommended:

- `~/.local/bin` on your `PATH`

## Supported Archive Types

- `.tar`
- `.tar.gz` / `.tgz`
- `.tar.bz2` / `.tbz2`
- `.tar.xz` / `.txz`

You can pass either a local archive path or an `http://` / `https://` URL. URL inputs are downloaded first via `curl -L`, then installed.

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
judo [--force] <AppName> <tar-file-or-url>
```

Examples:

```bash
judo Telegram ~/Downloads/Telegram.tar.xz
judo Obsidian ~/Downloads/obsidian.tar.gz
judo --force Telegram ~/Downloads/Telegram.tar.xz
```

## Name Collision Handling

Before extraction, `judo` checks for existing generated targets:

- `/opt/<AppName>`
- `~/.local/share/applications/<appname>.desktop`
- `~/.local/bin/<appname>` (symlink or regular file)

If conflicts exist:

- Interactive TTY: prompts for `new name`, `force`, or `cancel`
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

For `judo MyApp app.tar.xz`:

- Install: `/opt/MyApp`
- Temp extract: `/tmp/MyApp-extract`
- Desktop: `~/.local/share/applications/myapp.desktop`
- Symlink: `~/.local/bin/myapp`
- Backup (if replaced): `/opt/.MyApp.backup.YYYYMMDD-HHMMSS`

## Troubleshooting

- `No executable found`
  - Archive likely lacks executable bit(s) or contains only installer payloads.
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
