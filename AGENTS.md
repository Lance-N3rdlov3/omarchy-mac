# Task Guides

Deeper instructions for specific kinds of work live in `agents/skills/`. Read the
matching guide before starting:

- [`agents/skills/command-metadata.md`](agents/skills/command-metadata.md) - adding or changing commands in `bin/`
- [`agents/skills/install-scripts.md`](agents/skills/install-scripts.md) - working under `install/` or on system/user setup commands
- [`agents/skills/shell-dev.md`](agents/skills/shell-dev.md) - editing the Quickshell desktop under `shell/`
- [`agents/skills/icon-font.md`](agents/skills/icon-font.md) - adding branded glyphs to `default/fonts/omarchy/omarchy.ttf`
- [`agents/skills/acceptance-tests.md`](agents/skills/acceptance-tests.md) - writing or running graphical acceptance tests under `test/acceptance.d/`
- [`agents/skills/visual-verification.md`](agents/skills/visual-verification.md) - verifying any change with a visual effect in the running UI
- [`agents/skills/migrations.md`](agents/skills/migrations.md) - creating or changing migrations under `migrations/`

# Documentation Layout

Three documentation trees, split by genre and audience:

- `agents/skills/` - task procedure ("do this when doing X"), for anyone working on the codebase
- `docs/` - reference on how the system is shaped (file layout, update pipeline, theming, shell architecture), for anyone working on the codebase; skills link here for depth
- `manual/` - end-user documentation for using Omarchy, published; never codebase internals

# Style

- In markdown documents (`plans/`, `docs/`, `manual/`), write full lines — no hard wrapping at 80 columns; break only at structural boundaries like headings and list items
- Two spaces for indentation, no tabs
- Use bash 5 conditionals: use `[[ ]]` for string/file tests and `(( ))` for numeric tests
- In `[[ ]]`, don't quote variables, but do quote string literals when comparing values (e.g., `[[ $branch == "dev" ]]`)
- Prefer `(( ))` over numeric operators inside `[[ ]]` (e.g., `(( count < 50 ))`, not `[[ $count -lt 50 ]]`)
- Prefer a full `if`/`else` conditional for simple two-path control flow; don't rely on `exec` or `exit` in one branch to make following statements unreachable
- For strings/paths with spaces, quote them instead of escaping spaces with `\ ` (e.g., `"$APP_DIR/Disk Usage.desktop"`, not `$APP_DIR/Disk\ Usage.desktop`)
- Shebangs must use `#!/bin/bash` consistently (never `#!/usr/bin/env bash`)
- Scripts under `install/` and `migrations/` may be sourced and intentionally omit shebangs

# Command Naming

All commands start with `omarchy-`. Prefixes indicate purpose.

The authoritative list of user-facing command groups lives in `bin/omarchy` in `GROUP_DESCRIPTIONS`. Keep `GROUP_DESCRIPTIONS` updated when adding a new command prefix users are meant to browse to.

A group whose commands are all `# omarchy:hidden=true` gets no entry. That table drives the top-level group listing on its own, so an entry there advertises the group even when every command in it is hidden. `apply-` and `provision-` are deliberately absent for that reason; both still route, and `omarchy <group>` still prints a group header without one.

Common prefixes include:

- `cmd-` - check if commands exist, misc utility commands
- `capture-` - screenshots, screen recordings, and other capture tools
- `pkg-` - package management helpers
- `hw-` - hardware detection (return exit codes for use in conditionals)
- `refresh-` - copy default config to user's `~/.config/`
- `restart-` - restart a component
- `launch-` - open applications
- `install-` - install optional software
- `setup-` - interactive setup wizards
- `toggle-` - toggle features on/off
- `theme-` - theme management
- `update-` - update components

Do not maintain a second exhaustive prefix list here. Consult
`GROUP_DESCRIPTIONS` when selecting or checking a command group so this
guidance does not drift from the router.

# Runtime Environment

- `$OMARCHY_PATH` is set at the top level by the uwsm session environment and is always available to Omarchy runtime code.
- Commands in `bin/` and Quickshell QML should rely on `$OMARCHY_PATH` / `Quickshell.env("OMARCHY_PATH")`; do not derive fallback paths from `HOME`, `Quickshell.shellDir`, or re-export/default `OMARCHY_PATH` manually.

# Privileged Commands

- Follow the "Privilege Escalation" section of `default/agents/skills/omarchy/SKILL.md`. It draws the
  `sudo`/`pkexec` line by whether the caller has a terminal to enter a password in, and the repo's
  own scripts follow it.

# Git

- Commits should be atomic: include only one coherent change or fix, and do not mix unrelated work.
- Commit messages should be succinct and describe the change being made.

# Helper Commands

Use these instead of raw shell commands:

- `omarchy-cmd-missing` / `omarchy-cmd-present` - check for commands
- `omarchy-pkg-missing` / `omarchy-pkg-present` - check for packages (don't use these if you can just use `omarchy-pkg-add`/`omarchy-pkg-drop`)
- `omarchy-pkg-add` - install packages (handles both pacman and AUR)
- `omarchy-pkg-drop` - remove packages; use this instead of raw `pacman -R*`
- `omarchy-notification-send` - send desktop notifications; do not call `notify-send` directly
- `omarchy-hw-asus-rog` - detect ASUS ROG hardware (and similar `hw-*` commands)

Commands installed by Omarchy's default package set are runtime invariants. Invoke them directly; do not add defensive `omarchy-cmd-present` / `omarchy-cmd-missing` checks around them. Use command-presence helpers only for genuinely optional dependencies or code that can run before the default package set is installed.

Exceptions are allowed for migration and package-helper scripts where the helper may not be available yet, where the helper itself is being implemented, or where direct package-manager behavior is required.

# Menu

- The menu definition lives in `default/omarchy/omarchy-menu.jsonc`;
  [`docs/menu.md`](docs/menu.md) covers the schema, guards, and providers.
- Do not add `aliases` to new menu entries. Aliases are reserved for
  established alternate names users already type, kept for compatibility.

# Config Structure

- `config/` - default configs copied to `~/.config/`
- `default/themed/*.tpl` - templates with `{{ variable }}` placeholders for theme colors
- `themes/*/colors.toml` - theme color definitions (accent, background, foreground, red/green/yellow/blue/magenta/cyan and bright_* variants)

# Tests

Run focused automated tests for the area you changed;
[`docs/testing.md`](docs/testing.md) covers how the suites are shaped. Current
test entry points:

- `./test/all` - aggregate runner for CLI and shell tests; it intentionally does not run graphical acceptance tests
- `./test/cli` - CLI routing, command metadata, theme helpers, and safe dispatch coverage
- `./test/shell` - all Omarchy shell tests under `test/shell.d/`

New Omarchy shell tests should live in `test/shell.d/*-test.sh` so `./test/shell` picks them up automatically. Source `test/shell.d/base-test.sh` for shared root-path discovery, assertions, and Node test helpers.

The graphical acceptance suite runs in a disposable VM, not in the active
development session; see [`agents/skills/acceptance-tests.md`](agents/skills/acceptance-tests.md).

Visual changes must be verified in the running UI in addition to automated
tests; follow [`agents/skills/visual-verification.md`](agents/skills/visual-verification.md).

# Refresh Pattern

To copy a default config to user config with automatic backup:

```bash
omarchy-refresh-config hypr/hyprland.lua
```

This copies `$OMARCHY_PATH/config/hypr/hyprland.lua` to `~/.config/hypr/hyprland.lua`. The argument
is interpolated into both paths and only checked with `[[ -e ]]`, so pass a plain relative path: a
name containing `..` resolves and copies, landing outside `~/.config` rather than being rejected.

# Multi-agent loop

For substantive work on this repo (bindings, install, menus, migrations, security), use a **SWE → QA → Review** loop. The main session is the orchestrator.

## Roles

| Role | Responsibility | Edits? |
|------|----------------|--------|
| Orchestrator | Scope, handoffs, PR/user communication | Yes if loop stuck |
| SWE | Implement on `dev` (or a feature branch) | Yes |
| QA | Run automated suite; report only | No |
| Reviewer | Diff review vs `origin/main`; bugs block merge | No |

## Rules

- One writer at a time on a shared branch (or use a git worktree for parallel SWE tasks)
- QA never "fixes" — only commands, output, and FAIL reasons
- Reviewer: severity `bug` blocks; `suggestion` / `nit` are optional unless requested
- Max three SWE↔QA rounds, then surface remaining issues to the user
- Do not force-push `main`; push `dev` only when asked or policy allows
- Intentional Mac divergences are not bugs: notch-height bar sizing, `hid_apple fnmode=1` media keys, Shift+brightness for keyboard backlight, wf-recorder capture on the Asahi GPU, Spotify webapp, Codeberg update remote

## Standard QA commands

Run from the repo root before calling work done:

```bash
./test/all
bin/omarchy commands --check
for f in bin/omarchy-*; do
  if head -1 "$f" | grep -q python; then
    python3 -c 'import ast,sys; ast.parse(open(sys.argv[1]).read())' "$f" || exit 1
  else
    bash -n "$f" || exit 1
  fi
done
```

Not every command in `bin/` is a shell script, so the syntax check picks its
parser from the shebang. `bash -n` on the Python ones reports syntax errors for
perfectly valid files.

`./test/all` aggregates `./test/cli` and `./test/shell`; run those individually
when iterating on one area. It deliberately skips the graphical acceptance
suite, which belongs in a disposable VM (see
[`agents/skills/acceptance-tests.md`](agents/skills/acceptance-tests.md)).

QA should end with `VERDICT: PASS` or `VERDICT: FAIL`.

## Flow

1. Plan if the approach is ambiguous
2. SWE implements
3. QA always runs before "done"
4. Reviewer runs for non-trivial diffs (roughly >50 LOC, or install/bindings/security)
5. Bugs → SWE → QA again; then orchestrator reports and updates the PR if needed

---

# Omarchy Mac M3 Pro handoff

This file is a pickup point for a later agent. It records what was changed on this machine on 2026-10-24, including the Aurora boot, the shell and update fixes, the fork, and the pull requests. The live tree is the user's checkout at `/home/nrd/.local/share/omarchy`, not the packaged copy in `/usr/share/omarchy`.

The human-facing version of the update rules is `USER.md` in this checkout and in `/home/nrd/Work/USER.md`.

## Machine

- MacBook Pro 14-inch, M3 Pro (November 2023). Device tree `apple,j514s` / `apple,t6030` / `apple,arm-platform`.
- User `nrd`, uid 1001, in `wheel` and `render` (`render:x:987:nrd`). The session that installed Aurora did not see `render` until the reboot. The post-boot session does.
- Linux root is LUKS btrfs `asahi-root` on `nvme0n1p5` (110.3G). Filesystem UUID `725346d2-f127-47bc-b464-9dd46155e8d6`. LUKS UUID `f3c4b703-1cb4-4863-a725-93533757ecea`.
- EFI is `nvme0n1p4`, label `EFI - OMAC`, mounted at `/boot`. PARTUUID `38145d21-dc66-4c7c-823a-3ab6cc40267c`. Filesystem UUID `1494-A8B6`.
- macOS APFS is still on the disk (`nvme0n1p2` is about 309.6G, plus the other APFS slices). Recovery is from macOS.
- Firmware seen before install: stage-1 m1n1 `v1.6.1` (clean, not dirty), stage-2 `v1.9.9`, os-fw `14.7`, iBoot `iBoot-10151.140.19.700.2`. Stub product version `14.8.3`, device class `j514sap`, board id `0x04`, chip id `0x6030`. Do not copy `/boot/asahi/stub_info.json` into commits, reports, or chat. It contains a person's name and a user UUID.
- `$OMARCHY_PATH` is `/home/nrd/.local/share/omarchy`. `/etc/omarchy.conf` exports that path. Do not edit `/usr/share/omarchy`. The packaged helper there is older and still prints `omarchy/hyprland extra/hyprtoolkit extra/hyprland-guiutils`.

## Repo state

- `origin` is `https://github.com/omarchy-mac/omarchy-mac.git`. GitHub's parent repo for the fork is `omacom/omarchy-mac`. Default branch is `quattro`.
- `fork` is `https://github.com/Lance-N3rdlov3/omarchy-mac.git`. Local `quattro` still tracks `origin/quattro`.
- `quattro` was pushed to `fork` after the rebase commit. Do not push `quattro` to `origin`. Do not force-push `origin`. Publishing a rewritten local `quattro` uses `git push --force-with-lease fork quattro` only.
- Pull requests opened against `omacom/omarchy-mac` base `quattro`, from the fork:
  - https://github.com/omacom/omarchy-mac/pull/766 ShellColor (`shellcolor-qt612`)
  - https://github.com/omacom/omarchy-mac/pull/767 edge selection, rebase, and keeping a linked checkout (`aarch64-update-rebase`)
  - https://github.com/omacom/omarchy-mac/pull/768 M3 Pro GPU cost note (`m3-pro-gpu-cost`)
- Those PR branches were cut from `origin/quattro` (`ba546a61f`). The arm cherry-pick conflicted on `install/helpers/arm-package-sources.sh` because an earlier local commit had hardcoded `extra/hyprtoolkit` and `extra/hyprland-guiutils`. The resolution kept the dynamic selector from `52250a68c`. The PR branches do not contain that superseded hardcode commit.

Local `quattro` history above `origin/quattro`, oldest first:

- `7a8cf030b` Hardcoded `extra/hyprtoolkit` and `extra/hyprland-guiutils`. Superseded by the next arm commit. It remains in this branch's history and on `fork/quattro`. It is not in PR 767.
- `95268c62d` Qualify the shell palette as ShellColor under Qt 6.12. `shell/Commons/qmldir` has `singleton ShellColor 1.0 Color.qml`. The filename stays `Color.qml`. `Color.<prop>` under `shell/` was renamed to `ShellColor.<prop>`.
- `52250a68c` Select `omarchy/<name>` only when the synced edge database lists it, and do not abort the update when a fast-forward fails.
- `e126f25ef` Add `plans/arm64-repackage-assessment.md`. Written before the Aurora install. Two sentences in it still describe the pre-install kernel. The cost figures still stand.
- `d36e827d5` Rebase a dev checkout onto upstream and keep it linked. `omarchy-update-dev` fast-forwards, then `git rebase --autostash @{upstream}`, then `git rebase --abort` if a rebase is in progress. Channel refresh, channel set, and package reinstall skip unlink while `OMARCHY_PATH` is a git checkout other than `/usr/share/omarchy`.

`~/.config/hypr/looknfeel.lua` is outside the repo. It forces `cursor.no_hardware_cursors` only when `ls /dev/dri/renderD*` is empty. Do not put an unconditional software-cursor block back. `omarchy-refresh-config hypr/looknfeel.lua` would copy the packaged default over this file.

## Why the screen was wrong

Qt 6.12 adds `QtQuick.Color`, which shadowed `qs.Commons.Color`. The bar and Super+Space launcher drew black. The ShellColor rename fixes the shipped shell on Arch `quickshell 0.3.2-1`. Do not roll Qt back to 6.11 (`omawrite` 0.7 needs Qt 6.12). Plugins that still say unqualified `Color` stay broken. `~/.config/omarchy/plugins` was empty.

The jitter, cursor trails, and smeared terminal text came from the missing M3 display controller. Before Aurora, `/dev/dri` had only `simpledrm` `card0` and the panel was 3024x1890. Do not `modprobe appledrm`. Do not set `debug:damage_tracking = 0`.

## Update behavior now

`omarchy-update-dev` (`bin/omarchy-update-dev`):

- No-ops when `OMARCHY_PATH` is `/usr/share/omarchy`.
- Fails if the path is not a git checkout.
- Skips when the branch has no upstream.
- Runs `git pull --ff-only`.
- If that fails and `@{upstream}` exists, runs `git rebase --autostash @{upstream}`.
- If that rebase is still in progress, runs `git rebase --abort` and continues the package update with the local commits unchanged.
- The upstream of this checkout is `origin/quattro`, so the rebase pulls the project, not the fork.

`omarchy_arm_package_targets` prints `omarchy/<name>` only when `pacman -Sl omarchy` lists that package. On this machine that is `omarchy/hyprland` only. Do not add `omarchy/omarchy` to that list.

Checked without installing, against the databases already on disk:

- `omarchy-pkg-add` skips an unknown repo name and no-ops when the package is already installed.
- `pacman -Sup --needed --ignore hyprland omarchy/hyprland` resolved only `hyprland` 0.56.2-7. That package was not installed by the check.
- A sysupgrade print with no explicit targets was empty. Every `pacman -Qu` row was `[ignored]`.
- `omarchy-update-aur-pkgs` exited 0. Yay left `aurora-touchid`, `mesa-m3`, `mesa-m3-native-pro`, `mesa-m3-pro-vulkan`, and `omarchy-mac-keyring` alone because they are not AUR packages.
- `omarchy-pkg-aur-add` still exits 1 when the name is not in the AUR. The update path uses `yay -Sua`, not that command. One migration calls `omarchy-pkg-aur-add brave-origin-bin` with no fallback, and only if `brave-origin-beta-bin` is installed.

`[omarchy]` is `Usage = Sync`, so a normal sysupgrade does not replace the `omarchy` package. `/etc/pacman.conf` has:

```
IgnorePkg = linux-aurora linux-aurora-headers libfprint m1n1-aurora
```

The omarchy database also has `linux-aurora` 7.2.11.aurora3-1. The pin keeps the booted `7.1.12.aurora2-20261017.20` stack. Leave the pin until a watched kernel upgrade.

Tests that passed after the rebase change: `update-dev-test.sh`, `update-sequence-test.sh`, `channel-test.sh`, `arm-channel-test.sh`, `arm-package-sources-test.sh`. `./test/all` was not run. `arm-package-recovery-test.sh` still fails on this machine because `/usr/share/omarchy` supplies an older helper. That failure is environmental.

## Commands

Safe on this linked checkout:

- `omarchy update`. It rebases onto `origin/quattro`, aborts a conflicted rebase, then upgrades packages. It does not unlink the checkout. It can still install edge `hyprland` and ordinary Arch upgrades. The Aurora `IgnorePkg` lines keep the kernel and m1n1 pin.
- `omarchy refresh pacman` on aarch64 while this checkout is linked. It execs `omarchy update -y` and does not set a channel.
- `omarchy pkg add`. Unknown repo names are skipped.

Unsafe, or safe only with the limits below:

- `omarchy dev unlink` still rewrites `/etc/omarchy.conf` to `/usr/share/omarchy` and removes the dev sudoers drop-in. Callers skip it while a checkout is linked. Running the command itself still drops the checkout after the next login.
- `omarchy channel set` on aarch64 still runs `omarchy-update` with `OMARCHY_UPDATE_CHANNEL` set. The linked checkout is kept afterwards, but the channel transaction still installs that lane's `omarchy` and `omarchy-settings` pair and can change the pacman lane. Do not run it to "update" this machine.
- `omarchy reinstall pkgs` skips `omarchy-refresh-pacman` while linked, then still runs `pacman -Suu` and installs `install/omarchy-base.packages`. That is a wide transaction. Do not use it as an updater.
- `omarchy refresh pacman` when `OMARCHY_PATH` is `/usr/share/omarchy` still starts a stable channel update and unlinks.
- A bare `sudo pacman -Syu` is not the Omarchy update path. The update guard expects `OMARCHY_UPDATE_PACMAN=1`. It also does not pass the explicit edge Hyprland target.
- `omarchy refresh config hypr/looknfeel.lua` replaces the conditional hardware-cursor block.
- Deleting the Aurora `IgnorePkg` lines, or upgrading `linux-aurora` / `m1n1-aurora` by hand, is a separate reboot with the `boot.bin.before-*` backup still required.
- `git reset --hard origin/quattro` discards the local fixes. `git push origin` publishes them without a pull request.

## Aurora install (booted)

Release `aurora-2026.10.17.20`, kernel `7.1.12.aurora2-20261017.20`, booted as `7.1.12-2-20261017.20-sep-ARCH`. Installer: `/tmp/install-aurora-sep.local.sh` (patched), sha256 of the unpatched script `0151f93666ec867d28cc6ada7edab3be96e6f70fa092e94c851cebc51cdb90d5`. Two patches: `M3_STAGE1_VERSIONS` includes clean `v1.6.1`, and every `[ignored]` pacman row is skipped. Command: `pkexec bash /tmp/install-aurora-sep.local.sh --m3-handoff`. Then `pkexec gpasswd -a nrd render`.

Handoff switches in `/etc/m1n1.conf` and `/boot/m1n1/boot.bin`: `chosen.asahi,t6030-gpu=1`, `chosen.asahi,t6030-dcp=1`, `chosen.asahi,t6030-dcpext=1`. Backup: `/boot/m1n1/boot.bin.before-7.1.12.aurora2-20261017.20`. m1n1 sha256 `396cf3ec1774ca9d15ca1b6f71b5e8a3639b27f80b95d530722b35a5ea35141d`.

First boot around 12:06 local time on 2026-10-24 passed these checks:

- `/dev/dri/renderD128` exists, group `render`. `card1` is the GPU, `card2` is the display subsystem.
- Hyprland `eDP-1` is `3024x1964@120Hz`, scale 2, `hardwareCursorsInUse` true.
- `/run/user/1001/mesa-m3-session.state` said `decision=active` for T6030.
- Quickshell was running from this checkout.
- Bluetooth `hci0` was powered on. No ambient-light device was present under `/sys/bus/iio/devices` (linux-aurora#33). One later reboot is the usual fix. Bluetooth-at-boot misses are linux-aurora#129.

Do not reboot just to repeat the handoff. Do not port the driver into stock `linux-asahi`. If m1n1 stops ("No valid payload found", no boot menu, or a black screen for more than about two minutes), recover from macOS:

- Mount EFI PARTUUID `38145d21-dc66-4c7c-823a-3ab6cc40267c`.
- `cp -X '<mount>/m1n1/boot.bin.before-7.1.12.aurora2-20261017.20' '<mount>/m1n1/boot.bin'`.

From a working Linux shell the same backup can be copied over `/boot/m1n1/boot.bin` via a `boot.bin.new` temp file and `sync`. Then:

```bash
echo M1N1_UPDATE_DISABLED=1 | sudo tee -a /etc/default/update-m1n1
echo 396cf3ec1774ca9d15ca1b6f71b5e8a3639b27f80b95d530722b35a5ea35141d | sudo tee -a /var/lib/aurora-sep/m1n1-failed
```

If the desktop fails and Linux boots: Ctrl+Alt+F3, `mkdir -p ~/.config/mesa-m3 && touch ~/.config/mesa-m3/disable`, reboot. Undo with `rm ~/.config/mesa-m3/disable`. A failed GPU login sticks on software until `rm ~/.local/state/mesa-m3/attempt` and a new login.

The GRUB "Previous kernel" entry loads the copied `7.2.12` image. It does not undo m1n1. A bad m1n1 never reaches GRUB.

## Keeping the fork current

```bash
git fetch origin
git rebase origin/quattro
```

On a conflict, `git rebase --abort` restores the pre-rebase commits. After a clean rebase, `git push --force-with-lease fork quattro`. When a pull request merges, fetch and rebase again so those commits drop out. New reviewable work starts from `origin/quattro`, is pushed to `fork`, and opens another pull request. This checkout can stay on local `quattro` until that work is merged.

`omarchy update` performs the same rebase against `origin/quattro` and aborts on conflict before package updates continue.

## Pickup checklist

1. The GPU handoff already succeeded. Do not reinstall Aurora or port the driver.
2. Read `USER.md` before running an update. Use `omarchy update`. Do not run `omarchy channel set`, `omarchy dev unlink`, or `omarchy reinstall pkgs`.
3. Do not delete `/boot/m1n1/boot.bin.before-7.1.12.aurora2-20261017.20` or the Aurora `IgnorePkg` lines.
4. Do not push to `origin` unless the user asks. The fork remote is the publish target.
5. Do not copy stub identity fields into git or chat.
