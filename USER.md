# What changed on this Mac, and how to update it

This note is for the person using this M3 Pro MacBook. The matching agent handoff is `AGENTS.md`. Both files live in the Omarchy checkout and in `~/Work`.

The desktop code Omarchy is running comes from `/home/nrd/.local/share/omarchy`, not from `/usr/share/omarchy`. That checkout holds the local fixes. A GitHub fork of the project is at https://github.com/Lance-N3rdlov3/omarchy-mac. The project itself is https://github.com/omacom/omarchy-mac.

## What was fixed

The black bar and black Super+Space menu were a Qt 6.12 name clash. Qt's own `Color` type hid Omarchy's palette. The shell now calls that palette `ShellColor`. The menu and bar use theme colors again.

The jitter, mouse trails, and smeared terminal text were the display sitting on the boot framebuffer. There was no GPU. Aurora release `2026.10.17.20` is installed and has booted. The panel is the native 3024x1964 at 120 Hz, a hardware cursor is in use, and `/dev/dri/renderD128` exists. The kernel is the Aurora `7.1.12` build from that release.

Bluetooth came up on that first boot. The ambient-light sensor did not. If the sensor is still missing, one more reboot is the usual fix. That is a known Aurora quirk, not a reason to change the kernel.

## What an update is allowed to change

`omarchy update` is the command to use.

It does two jobs:

1. It brings new Omarchy git commits from the project onto this checkout. Your local commits stay on top. If git cannot replay them cleanly, it cancels that rebase and leaves your commits exactly where they were, then continues.
2. It upgrades normal Arch packages, AUR packages that actually come from the AUR, and the edge Hyprland package when the edge database still publishes it.

It does not point the session back at `/usr/share/omarchy`. It does not remove the Aurora kernel pin. These lines in `/etc/pacman.conf` are what hold the working GPU stack, and they should stay:

```
IgnorePkg = linux-aurora linux-aurora-headers libfprint m1n1-aurora
```

The mouse setting in `~/.config/hypr/looknfeel.lua` is also outside git. It uses a hardware cursor when the GPU is present and a software cursor when it is not. Leave that file alone.

Packages that are not part of git and must not be rebuilt by an update: `mesa-m3`, `mesa-m3-native-pro`, `mesa-m3-pro-vulkan`, `aurora-touchid`, and `omarchy-mac-keyring`. The AUR updater already skips them.

## Commands that were changed

- `omarchy update` (through `omarchy-update-dev`). Fast-forward when it can. Otherwise rebase your commits onto the project's `quattro` branch. On a conflict it runs `git rebase --abort` and keeps your commits.
- `omarchy refresh pacman`. On this Mac, while the checkout is linked, this is now just `omarchy update`. It no longer switches you onto the packaged tree.
- `omarchy channel set`. While this checkout is linked, the x86 path no longer refreshes pacman or unlinks it. On this Apple Silicon Mac the command still starts a channel install first. See the unsafe list.
- `omarchy reinstall pkgs`. While this checkout is linked, it no longer starts a channel refresh. It still reinstalls the whole default package set. See the unsafe list.
- `omarchy pkg add`. Already skipped packages this Mac's repos do not have, instead of failing the whole command. That behavior was already in place and was rechecked.
- The ARM package selector. It asks pacman for an edge package only when the edge database actually has it. Right now that is Hyprland. It no longer fails the update by asking for `hyprtoolkit` and `hyprland-guiutils`, which edge stopped publishing for this architecture.

`omarchy pkg aur add` was not changed. If you name a package the AUR does not have, that command still fails. The normal update does not use it.

## Safe

- `omarchy update`
- `omarchy refresh pacman` on this Mac, as long as Omarchy is still linked to `/home/nrd/.local/share/omarchy`
- `omarchy pkg add` for a package you actually want
- Updating the fork after a clean rebase, with the commands in the next section

## Unsafe

- `omarchy dev unlink`. This one command still switches Omarchy back to `/usr/share/omarchy`. The next login would lose the ShellColor fix and the update fixes. Other commands were changed so they do not call it for you. Do not run it yourself.
- `omarchy channel set`. On this Mac it still installs the chosen channel's Omarchy packages before it agrees to keep the checkout. Use `omarchy update` instead.
- `omarchy reinstall pkgs`. It skips the channel switch, then reinstalls every default package. That is a much bigger change than an update.
- `sudo pacman -Syu` by hand. Use `omarchy update`.
- Removing the `IgnorePkg` lines, or upgrading `linux-aurora` or `m1n1-aurora`, unless you mean to change the GPU stack and you still have the bootloader backup.
- `omarchy refresh config hypr/looknfeel.lua`. That copies a stock config over the hardware-cursor rule.
- `git reset --hard` onto the project branch. That throws away the local fixes.
- Pushing this branch to the project remote. Pull requests are the way those fixes get upstream. The fork is where this machine's branch is published.

## How to take project updates without losing the fixes

In a terminal, from the checkout:

```bash
cd /home/nrd/.local/share/omarchy
git fetch origin
git rebase origin/quattro
```

If git stops and talks about a conflict, and you want your current fixes untouched:

```bash
git rebase --abort
```

If the rebase finishes cleanly, publish it only to your fork:

```bash
git push --force-with-lease fork quattro
```

`omarchy update` does that rebase for you and then updates packages. You only need the manual commands when you want to update git by itself, or when you want to publish the result to GitHub.

After one of these pull requests merges, run the fetch and rebase again. Git will drop the commits the project already has.

- https://github.com/omacom/omarchy-mac/pull/766
- https://github.com/omacom/omarchy-mac/pull/767
- https://github.com/omacom/omarchy-mac/pull/768

New work that should be reviewed starts as its own branch from the project's `quattro`, gets pushed to the fork, and opens a new pull request. Day-to-day use of this Mac can stay on the local `quattro` branch.

## If the Mac does not boot

The bootloader backup is `/boot/m1n1/boot.bin.before-7.1.12.aurora2-20261017.20`.

From macOS: hold the power button for startup options, open Terminal, mount the EFI partition whose PARTUUID is `38145d21-dc66-4c7c-823a-3ab6cc40267c`, and copy that backup over `m1n1/boot.bin` on that volume (`cp -X`).

The GRUB menu entry named "Previous kernel" loads an older Linux image. It does not repair a bootloader that never reaches GRUB.
