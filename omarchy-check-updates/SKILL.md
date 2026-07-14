---
name: omarchy-check-updates
description: Check for available Omarchy updates, compare versions, fetch release notes from GitHub, and assess whether the update affects the current machine. Also lists available system package updates from the official repos and the AUR, plus outdated mise-managed global dev tools. Generates a concise report with hardware impact analysis.
---

# Omarchy Update Checker

Check for available Omarchy updates and assess their impact on the current machine.

## When To Use

Use this skill when the user asks to:

- Check for Omarchy updates
- Compare the current Omarchy version to the latest version
- Check Omarchy GitHub release notes
- Determine whether Omarchy update changes affect this hardware
- Summarize whether an Omarchy update is worth applying on this machine
- List available system/package updates (official repos and AUR)

## Safety Rules

- Never run `omarchy update perform` or `omarchy update -y`. `omarchy update available` (read-only) is safe to use.
- Never change files, packages, bootloader settings, or kernel parameters.
- Prefer read-only commands and report what was checked.
- If a command would require sudo or prompts, do not run it unless the user explicitly asks.

## Required Workflow

1. Check installed version:

```bash
omarchy version
```

2. Check latest available version:

```bash
omarchy update available
```

3. If no Omarchy update is available, say so and skip the release-notes and hardware-impact steps (4–6). Still run the package-update listing (step 7) and emit the report — an up-to-date Omarchy does not end the run.

4. Fetch release notes from GitHub for the relevant versions. Prefer `gh`:

```bash
gh release view <available-tag> --repo basecamp/omarchy --json tagName,name,publishedAt,body,url
gh release view <current-tag> --repo basecamp/omarchy --json tagName,name,publishedAt,body,url
```

5. Summarize changes between installed and available versions.

6. Inspect only the hardware/configuration relevant to the release notes. Common checks:

```bash
lspci -nn | rg -i 'vga|3d|display|graphics|intel|amd|nvidia|arc|xe|wifi|bluetooth|audio|ethernet'
printf '%s\n' "$(< /proc/cmdline)"
lsmod | rg '(^xe|^i915|^amdgpu|^nvidia|^nouveau)'
test -f /etc/default/limine && rg -n '<specific-release-keyword>|KERNEL_CMDLINE' /etc/default/limine || true
test -f /etc/kernel/cmdline && rg -n '<specific-release-keyword>' /etc/kernel/cmdline || true
```

Replace `<specific-release-keyword>` with a precise term from the release notes, such as `xe\.enable_panel_replay`, a package name, service name, driver name, config key, or command name.

For package-related release notes, also check whether a specific package is installed:

```bash
pacman -Q <package>
pacman -Qs <package-or-keyword>
```

7. List available package updates (run this on every invocation, even when no Omarchy update is available):

```bash
# Official-repo updates. No sudo: checkupdates syncs a private temp copy of the DB.
# No output + exit 2 means "no repo updates available" — this is NOT an error.
checkupdates || true

# AUR updates, only if yay is available.
# No output + exit 1 means "no AUR updates available" — this is NOT an error.
command -v yay >/dev/null && (yay -Qua || true)
```

`checkupdates` (from `pacman-contrib`) is the accurate, no-sudo way to list official-repo updates. If it is not installed, fall back to `yay -Qu` or `pacman -Qu` and flag the results as possibly stale, since those compare against the local sync database without refreshing it. Do not install `checkupdates`, `paru`, or any helper just to perform the check, and never run `pacman -Sy` (requires sudo and risks a partial upgrade).

8. List outdated `mise`-managed global tools (Omarchy ships `mise` out of the box, so no guard is needed):

```bash
# Outdated tools managed by mise (dev tools installed via `mise use -g ...`).
# Empty output means everything is up to date. Columns: tool, requested, current, latest, config.
mise outdated

# mise updates itself separately from its managed tools; surface its own update if any.
mise version
```

`mise outdated` is read-only. Do not run `mise upgrade`, `mise self-update`, or install anything.

## Report Format

Keep the response concise and include:

- Current version
- Available version
- Release notes URL
- Summary of changes
- Impact assessment: affected, not affected, likely affected, or unclear
- Evidence checked, including relevant hardware, kernel command line, boot config, loaded modules, installed packages, or services
- Available package updates:
  - Official repos: count and the packages as `name old -> new` (or "none")
  - AUR: same format, listed separately (or "none")
  - If the listing fell back to `pacman -Qu`/`yay -Qu`, note that results may be stale
  - mise global tools: outdated tools as `name current -> latest` (or "none"), plus a note if `mise` itself has an update available
- Explicitly state that no update was applied and nothing was installed

## Example Assessment

If a release removes `xe.enable_panel_replay=0` from `KERNEL_CMDLINE`, check:

```bash
lspci -nn | rg -i 'vga|3d|display|graphics|intel|arc|xe'
printf '%s\n' "$(< /proc/cmdline)"
lsmod | rg '(^xe|^i915)'
rg -n 'xe\.enable_panel_replay|KERNEL_CMDLINE' /etc/default/limine /etc/kernel /boot 2>/dev/null || true
```

If the machine uses AMD or NVIDIA graphics and the kernel command line/boot config do not contain the flag, report that the update is harmless but likely irrelevant to this hardware.
