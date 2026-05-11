---
name: omarchy-check-updates
description: Check for available Omarchy updates, compare versions, fetch release notes from GitHub, and assess whether the update affects the current machine. Generates a concise report with hardware impact analysis.
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

## Safety Rules

- Never run `omarchy update`, `omarchy update perform`, or `omarchy update -y`.
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
omarchy-update-available --version
```

3. If no update is available, report that and stop unless the user asked for release details anyway.

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

7. For package-related release notes, use read-only package queries:

```bash
pacman -Q <package>
pacman -Qs <package-or-keyword>
pacman -Qu
yay -Qua
```

Only run `yay -Qua` if `yay` is available. Do not install `checkupdates`, `paru`, or any helper just to perform the check.

## Report Format

Keep the response concise and include:

- Current version
- Available version
- Release notes URL
- Summary of changes
- Impact assessment: affected, not affected, likely affected, or unclear
- Evidence checked, including relevant hardware, kernel command line, boot config, loaded modules, installed packages, or services
- Explicitly state that no update was applied

## Example Assessment

If a release removes `xe.enable_panel_replay=0` from `KERNEL_CMDLINE`, check:

```bash
lspci -nn | rg -i 'vga|3d|display|graphics|intel|arc|xe'
printf '%s\n' "$(< /proc/cmdline)"
lsmod | rg '(^xe|^i915)'
rg -n 'xe\.enable_panel_replay|KERNEL_CMDLINE' /etc/default/limine /etc/kernel /boot 2>/dev/null || true
```

If the machine uses AMD or NVIDIA graphics and the kernel command line/boot config do not contain the flag, report that the update is harmless but likely irrelevant to this hardware.
