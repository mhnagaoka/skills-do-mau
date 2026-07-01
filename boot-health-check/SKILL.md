---
name: boot-health-check
description: Inspect the current boot for hidden problems on an Arch Linux / Omarchy machine. Detects failed systemd units, kernel and journal errors and warnings, module ABI mismatches from partial upgrades, world-accessible boot files, and misconfigured system units. Use when the user asks to check boot health, look for hidden issues since last boot, verify system is clean after an upgrade, or diagnose strange behavior after rebooting.
---

# Boot Health Check

Inspect the current boot for hidden problems on an Arch Linux / Omarchy machine and categorize findings by severity.

## When To Use

Use this skill when the user asks to:

- Check if the system is healthy after a boot
- Look for hidden issues since the last boot / reboot
- Verify everything is clean after an upgrade or restore
- Diagnose strange behavior that appeared after rebooting
- Audit a snapshot restore (e.g. limine snapshot) for inconsistencies

## Safety Rules

- **Never** run `systemctl reboot`, `systemctl poweroff`, `systemctl rescue`, `systemctl emergency`, or any state-changing systemd verb.
- **Never** install packages, edit `/etc/`, change file permissions, or modify the bootloader.
- Prefer read-only commands and report findings; suggest fixes as text, do not apply them unless the user explicitly asks.
- `dmesg` typically needs root on Arch. If sudo is available and the user has consented (or the agent can run passwordless sudo), use it. Otherwise skip `dmesg` and note the gap — `journalctl -b -k` is a non-sudo fallback for kernel messages and is preferred when available.

## Required Workflow

Run the following in parallel where possible. Then categorize and report.

1. **Boot identity and uptime:**

```bash
uname -a
uptime -s    # boot timestamp
uptime -p    # current uptime
cat /proc/version 2>/dev/null
```

2. **Failed systemd units (system and user):**

```bash
systemctl --failed --no-pager --all
systemctl --user --failed --no-pager --all 2>/dev/null || true
```

3. **Journal errors and warnings for the current boot:**

```bash
journalctl -b -p err --no-pager
journalctl -b -p warning --no-pager
```

If too noisy, narrow to relevant units: `journalctl -b -u <unit> --no-pager`.

4. **Kernel ring buffer (if sudo available, otherwise skip):**

```bash
sudo dmesg -l err,crit,alert,emerg 2>/dev/null
sudo dmesg -l warning 2>/dev/null | tail -50
```

Non-sudo fallback:

```bash
journalctl -b -k --no-pager | tail -80
```

5. **Module ABI sanity (catch partial upgrades):** For each wireless/GPU/dkms driver family actually present on this hardware, verify `vermagic` matches the running kernel and `srcversion` is consistent across dependent modules.

```bash
UNAME_R=$(uname -r)
for m in mac80211 cfg80211 iwlwifi iwlmvm amdgpu radeon nvidia nvidia_uvm nouveau xe i915; do
  modprobe --show-exports "$m" >/dev/null 2>&1 || continue
  modinfo "$m" 2>/dev/null | rg '^(filename|vermagic|srcversion):' || true
done
echo "running kernel: ${UNAME_R}"
```

If a module's `vermagic` differs from `uname -r`, or a dependent module (e.g. `iwlmvm`) reports `Unknown symbol` in dmesg/journal, mark it as **Action required — partial upgrade suspected**. Suggest re-running the upgrade (`omarchy update` or `pacman -Syu`) before further diagnosis.

6. **Optional follow-up checks — only invoke if a relevant keyword appeared in steps 3–5:**

- Failed `set-wireless-regdom`? → check `iw reg get` and `/etc/conf.d/wireless-regdom`.
- `nvidia_uvm` failed to load? → `lsmod | rg nvidia` to confirm no NVIDIA card is active (often harmless on dual-GPU laptops).
- `bootctl` security warnings? → `stat -c '%a %n' /boot /boot/loader/random-seed` (no sudo needed).
- `Unknown key` in systemd conf.d? → `rg -n . /etc/systemd/system.conf.d/ /etc/systemd/user.conf.d/` (read-only).
- Service marked executable warning? → `stat -c '%a %n' <path>` to confirm.
- Mako/dbus name mismatch? → cosmetic, no further action.

Keep these checks one-deep; do not recurse into a full audit.

## Categorization

Sort every finding into exactly one bucket:

- **Expected / informational** — firmware quirks (ACPI `_DSM`, nvme `SUBNQN`), benign blacklist duplicates, modules for absent hardware (e.g. `nvidia_uvm` with no NVIDIA GPU), UFW/laptop-mode blocks working as intended.
- **Cosmetic, no action needed** — startup races that self-resolve (gkr-pam keyring unlock when not using GNOME Keyring, bluetooth device temporarily down, UPower-not-yet-registered during wireplumber init), wireplumber/early-desktop noise.
- **Worth a one-line cleanup** — small safe fixes like world-readable boot files, systemd directive typos, executable service files, missing `WIRELESS_REGDOM`. Suggest the exact command; ask before running.
- **Action required** — failed systemd units, kernel panics / OOM / BUG in dmesg, module ABI mismatch (`Unknown symbol`, divergent `vermagic`), repeated hardware errors affecting current session (not historical firmware quirks).

## Report Format

Keep the response concise and structured:

1. **Boot summary** — boot timestamp, uptime, kernel version, OS (Detect `Omarchy` via `/etc/os-release` or `omarchy version`).
2. **Failed units** — count and list, or "0 failed units".
3. **Findings by category**, in priority order: Action required → Worth cleanup → Expected/informational → Cosmetic. Skip empty categories.
4. **Suggested fixes** — one line each, exact commands. Mark optional ones with "(optional)".
5. **Evidence checked** — short list (failed units, journal err+warning, dmesg, modinfo vermagic, lspci/lsmod cross-checks).
6. Explicitly state: **No changes were applied.** Read-only inspection only.

## Example Finding Wording

- **Action required** — `iwlmvm: Unknown symbol ieee80211_* (err -2)` in dmesg; `modinfo mac80211` shows `vermagic 7.0.9-arch2-1` but `uname -r` is `7.0.10-arch1-1`. → Likely partial upgrade left mismatched `/lib/modules`. Re-run the upgrade before further diagnosis.
- **Worth cleanup** (optional) — `bootctl` reports `/boot` world-accessible. Suggested: `sudo chmod 700 /boot`.
- **Expected/informational** — nvidia_uvm module not found; no NVIDIA card loaded (`lsmod | rg nvidia` empty). Harmless on this AMD-GPU system.

## What This Skill Does NOT Do

- Monitoring (no historical baseline).
- Root cause analysis for failures introduced in earlier boots (use `journalctl --list-boots` to read previous boots if asked).
- Anything requiring network access (no upstream issue search).