# Linux Drive Health Monitor - Full Documentation

**Complete technical reference and advanced configuration guide**

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [How It Works](#how-it-works)
- [Configuration](#configuration)
- [Alert System](#alert-system)
- [Safety & Error Handling](#safety--error-handling)
- [Technical Details](#technical-details)
- [Advanced Usage](#advanced-usage)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)

---

## Overview

This script provides comprehensive drive health monitoring with aggressive multi-channel alerting and safe automated maintenance.

**Design Philosophy:**
1. **Safety first** - Never write to failing drives
2. **Alert everyone** - No one should miss a drive failure
3. **Graceful degradation** - Works even if optional components missing
4. **Universal compatibility** - Works on any Linux distro/desktop

---

## Features

### Health Monitoring

**Drive Coverage:**
- NVMe drives (`/dev/nvme*`)
- SATA drives (`/dev/sd*`)
- USB drives (with SMART support)

**SMART Checks:**
- Verifies SMART support before testing
- Runs self-assessment health test
- **Zero tolerance:** Any non-PASSED result triggers full alert cascade
- Gracefully skips drives without SMART (USB sticks, etc.)

### Alert System

When a drive fails SMART checks, the script alerts through **every available channel**:

**Desktop Notifications (All Logged-In Users):**
- Zenity error popups (blocking, can't be ignored)
- Critical desktop notifications (persistent)
- Works with multiple users simultaneously

**Terminal Broadcast:**
- SSH sessions (`/dev/pts/*`)
- Local terminals (`/dev/tty*`)
- tmux/screen sessions
- Virtual consoles

**Email:**
- Aggregated error report
- Includes all drive failures or xfs_fsr errors if there are any of either
- Sent only once with all issues

**Console:**
- Printed to stdout if running interactively

**Multi-Desktop Environment Support:**

*Wayland:*
- mako
- waybar-notify

*X11/XWayland:*
- xfce4-notifyd (XFCE)
- dunst (universal)
- mate-notification-daemon (MATE)
- notify-osd (Unity/Ubuntu)

Script starts all available daemons silently - whichever is installed will work.

### Automated Maintenance

**Only runs if ALL drives report healthy.**

**1. TRIM on Unmounted Drives**

For drives not normally mounted (hot spares, rescue partitions):
- Temporarily mounts to scratch location
- Waits for mount to complete (no race conditions)
- Performs TRIM
- Unmounts cleanly
- Waits for unmount to complete

**2. TRIM on All Mounted Drives**

Uses `fstrim --all` for any mounted filesystem supporting TRIM.

**3. XFS Defragmentation (HDDs Only)**

- **Auto-detects drive type** using `lsblk -d -o rota`
- Only defrags spinning hard drives (ROTA=1)
- **Automatically skips SSDs** even if listed in `xfsdrives`
- Filters noise from xfs_fsr output
- Only runs if xfs_fsr not already running
- Reports any errors via email

---

## How It Works

### Phase 1: SMART Health Check

```
For each NVMe drive (/dev/nvme0n1, nvme1n1, ...)
  ├─ Check if SMART supported -> Skip if not
  ├─ Run SMART health assessment
  └─ If anything other than "PASSED" -> Set baddrive flag + alert cascade

For each SATA/USB drive (/dev/sda, sdb, sdc, ...)
  ├─ Check if SMART supported -> Skip if not (USB sticks, etc.)
  ├─ Run SMART health assessment
  └─ If anything other than "PASSED" -> Set baddrive flag + alert cascade
```

### Phase 2: Alert Cascade (For Any and If Every Drive That Fails to PASS)

```
Detect desktop environment (Wayland vs X11)
  └─ Start all relevant notification daemons
     └─ For each logged-in user:
        ├─ Show zenity error popup
        └─ Send critical desktop notification
           └─ Broadcast to all terminals:
              ├─ SSH sessions (/dev/pts/*)
              ├─ Virtual terminals (/dev/tty*)
              ├─ Print to console
              └─ Aggregate for email
```

### Phase 3: Maintenance (Only If All Healthy)

```
If baddrive == 0:
  ├─ For each unmounted drive:
  │  ├─ Mount to scratch location
  │  ├─ Wait for mount to complete
  │  ├─ Run fstrim
  │  ├─ Unmount
  │  └─ Wait for unmount to complete
  │
  ├─ Run fstrim --all (all mounted drives)
  │
  └─ For each XFS drive:
     ├─ Check if rotational (lsblk -d -o rota)
     ├─ If SSD (ROTA=0) -> Skip with message
     ├─ If HDD (ROTA=1):
     │  ├─ If not mounted -> mount temporarily
     │  ├─ Run xfs_fsr (if not already running)
     │  ├─ Capture any errors
     │  └─ Unmount if we mounted it
     └─ Report errors via email

Send aggregated email if any errors/warnings
```

---

## Configuration

### User Variables

**`machineID`**
- Default: `"MachineID"` -> falls back to `"Your Computer"`
- Purpose: Identifies which machine in email alerts
- Used in subject: `"CRITICAL ERROR ALERT ON A DRIVE ON Your Computer"`
- Useful if managing multiple systems

**`notifyemail`**
- Default: `"YOUREMAILHERE"` -> falls back to `"root"`
- Purpose: Where to send aggregated error reports
- Requires working `mail` command (see Troubleshooting)
- Examples: `"admin@example.com"`, `"sysadmin@company.org"`

**`xfsdrives`**
- Default: `"/dev/xfshdd1 /dev/xfshdd2"` -> falls back to `""`
- Purpose: XFS-formatted drives to defrag
- **IMPORTANT: Only list spinning hard drives!**
- Script auto-detects SSDs and skips them
- Space-separated list: `"/dev/sdc1 /dev/sdd1 /dev/nvme0n1p3"`
- Can be partitions or whole drives
- Check drive type: `lsblk -d -o name,rota` (1 = HDD, 0 = SSD)

**Why SSDs are auto-skipped:**
- XFS defrag provides no benefit on SSDs
- Causes unnecessary write wear
- Script checks ROTA value before defragging
- If ROTA=0 (SSD), prints message and skips

**`unmounteddrvs`**
- Default: `"/dev/unmounteddrv1 /dev/unmounteddrv2"` -> falls back to `""`
- Purpose: Drives to temporarily mount for TRIM
- **Use cases:**
  - Hot-swap spare drives (installed but unmounted)
  - Rescue/recovery partitions
  - Secondary internal SSDs not auto-mounted
  - Emergency boot partitions
- **NOT for:** External USB drives (might be unplugged!)
- Space-separated list: `"/dev/sdb1 /dev/sdc2"`
- Example: `"/dev/sdb1"` (rescue partition)

**`mountbase`**
- Default: `"/scratch/mount/point"` -> falls back to `/tmp/drvchkmount` (auto-created)
- Purpose: Temporary mount point for unmounted drive operations
- Script checks if in use as an existing mount point before mounting
- Auto-creates temp directory if using defaults

---

## Alert System

### Desktop Notifications

**Multi-User Support:**

Instead of only alerting root, script finds all logged-in users and alerts each:

```bash
users=$(who | awk '{print $1}' | sort -u)
for user in $users; do
  runuser -u "$user" -- notify-send -u critical "HDD SMART ALERT!" "${alert}"
  runuser -u "$user" -- zenity --error --text="${alert}"
done
```

**Why this matters:**
- Family computer: Both users get alerted
- Multi-user server: everyone logged in sees the alert,
    maximises the chances that it will get noticed and reported
- SSH + desktop user: Both sessions alerted

**How it works:**
- `who` lists logged-in users
- `runuser` executes notification as that user
- Each user sees popup in their session
- Notifications appear on correct DISPLAY

### Terminal Broadcast

**SSH Sessions:**
```bash
for pts in /dev/pts/[0-9]*; do
  [ -w "$pts" ] && printf "%b\n" "${alert}" > "$pts" 2>/dev/null
done
```

**Virtual Terminals:**
```bash
for tty in /dev/tty[0-9]*; do
  [ -w "$tty" ] && printf "%b\n" "${alert}" > "$tty" 2>/dev/null
done
```

**Coverage:**
- SSH sessions (pts/0, pts/1, etc.)
- Local terminals (xterm, gnome-terminal, etc.)
- tmux/screen sessions
- Virtual consoles (Ctrl+Alt+F1-F6)

**Safety:**
- Only writes to writable terminals (`[ -w "$pts" ]`)
- Errors silently ignored (`2>/dev/null`)
- Won't break if terminal closed mid-write

### Email Aggregation

Instead of sending separate emails for each issue:

```bash
# During checks, accumulate errors:
alert_email="${alert_email} ${error_message}"
alert_emailsubject="CRITICAL ERROR ALERT ON A DRIVE ON ${machineID}"

# At end, send single email:
[ -n "${alert_emailsubject}" ] && mail -s "${alert_emailsubject}" "${notifyemail}" <<< "${alert_email}"
```

**Result:** One email with all issues, not spam.

---

## Safety & Error Handling

### What Prevents Data Loss

**1. SMART Check First**
```bash
if [ "${baddrive}" -eq 0 ]; then
  # Maintenance only runs if ALL drives healthy
fi
```

**If ANY drive fails SMART, script:**
- Triggers alerts
- Continues checking the rest of the drives
- Skips ALL maintenance
- Sends email notice
- Exits cleanly

**2. Mount Point Conflict Detection**
```bash
if ! mount | grep -q "on ${mountbase} "; then
  # Safe to mount
else
  echo "Mount point in use - skipping"
fi
```

Won't overwrite active mounts.

**3. Wait Loops (Not sleep)**

**Bad approach:**
```bash
mount /dev/sdb1 /mnt
sleep 2  # Hope it's done
fstrim /mnt  # Might not be mounted yet!
```

**Good approach:**
```bash
mount /dev/sdb1 /mnt
while ! mount | grep -q " /dev/sdb1 "; do
  sleep 0.5
done
# NOW we know it's mounted
fstrim /mnt
```

**Why:** Fast when it's fast, safe when it's slow.

**4. SSD Auto-Detection**
```bash
is_rot=$(lsblk -d -o rota -n "${drive}" | tr -d ' ')
if [ "${is_rot}" -eq 1 ]; then
  # Safe to defrag (spinning drive)
else
  echo "Drive ${drive} is SSD, skipping defrag"
fi
```

Even if you accidentally list SSDs in `xfsdrives`, they won't be defragged.

**5. Process Conflict Prevention**
```bash
if ! pgrep -i xfs_fsr >/dev/null; then
  # Safe to start defrag
fi
```

Won't start multiple concurrent defrags.

**6. Strict Error Handling**
```bash
set -euo pipefail
shopt -s nullglob
```

- `set -e`: Exit on any unhandled error
- `set -u`: Treat undefined variables as errors
- `set -o pipefail`: Pipelines fail if any command fails
- `shopt -s nullglob`: Glob patterns matching nothing expand to empty (not literal `*`)

### What Triggers Alerts?

**Always triggers alerts (critical):**
- Drive SMART failure
  - Desktop notifications (all users)
  - Terminal broadcasts (all sessions)
  - Email
  - Console output

**Sends email notices only:**
- xfs_fsr warnings/errors during defrag
- Unexpected errors during maintenance

**Never alerted (normal operation):**
- Drives without SMART support (just skipped with message)
- `xfs_fsr` noise like "start inode=0" (filtered)
- Missing optional notification daemons (silently ignored)
- SSD in xfsdrives list (skipped with message, not error)

---

## Technical Details

### Why Temporary Mounts for TRIM?

**Problem:** TRIM only works on mounted filesystems.

**Scenario:** You have a hot-spare SSD or rescue partition that's:
- Installed in the system
- Formatted and ready
- But not normally mounted (to prevent accidental writes)

**Solution:** Script temporarily mounts it, TRIMs it, unmounts it.

**Why it's safe:**
- Checks if mount point already in use
- Waits for mount completion before TRIM
- Waits for unmount completion before exiting
- Only touches drives explicitly listed in `unmounteddrvs`
- Mount point conflicts are detected and skipped

### Why Check SMART Before Maintenance?

**Philosophy:** Don't write to failing drives.

If a drive reports SMART errors:
- Might be about to fail completely
- Writing to it could accelerate failure
- TRIM/defrag might make data recovery harder
- Drive might be in read-only mode already

**Better approach:**
1. Detect problem immediately
2. Alert aggressively
3. Stop all writes to that drive ASAP
4. Let user back up data
5. Replace drive
6. Resume normal maintenance only when all healthy

### Why Multiple Notification Daemons?

**Problem:** Different desktop environments use different notification systems.

**Examples:**
- XFCE: xfce4-notifyd
- KDE: built-in notification system
- GNOME: gnome-shell handles it
- Wayland compositors: mako, waybar-notify
- Standalone: dunst

**Solution:** Try starting all of them. Whichever is installed will work.

**Implementation:**
```bash
for daemon in "${wayland_daemons[@]}"; do
  "${daemon}" &>/dev/null &
done
```

**What happens:**
- If `mako` installed and not running -> starts
- If `mako` not installed -> command fails silently (`&>/dev/null`)
- If `mako` already running -> new instance exits harmlessly
- Notification appears in whichever daemon is running

**Result:** Works on any desktop environment without user configuration.

### Why Terminal Broadcast?

**Scenario:** You're SSH'd into server managing something critical.

**Problem:** You might not check email immediately.

**Solution:** Alert appears directly in your terminal session.

**Coverage:**
- SSH sessions see alert in active terminal
- tmux/screen users see it in their session
- Multiple SSH sessions all get alerted
- Virtual console users (Ctrl+Alt+F#) see it too

**Safety:**
- Only writes to terminals that exist and are writable
- Errors silently ignored (closed terminal, no permission, etc.)
- Won't break your terminal (just prints text)

### Why Multi-User GUI Alerts?

**Scenario:** Family computer with multiple user accounts.

**Problem:** Only root would get notification.

**Solution:** Script finds all logged-in users and notifies each.

**Example:**
- Parent logged in on desktop
- Kid logged in via SSH for gaming server
- Both see alert about failing drive

**How:**
- `who` command lists all logged-in users
- `runuser -u "$user"` executes notification as that user
- Each user's notification appears in their session
- Works even if multiple users on same desktop (fast user switching)

---

## Advanced Usage

### Custom Alert Methods

Add to `fn_smartcheck()` after existing alerts:

**Slack webhook:**
```bash
curl -X POST -H 'Content-type: application/json' \
  --data "{\"text\":\"${alert_txt}\"}" \
  https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

**Discord webhook:**
```bash
curl -X POST -H 'Content-type: application/json' \
  --data "{\"content\":\"${alert_txt}\"}" \
  https://discord.com/api/webhooks/YOUR/WEBHOOK
```

**Syslog:**
```bash
logger -p daemon.crit -t drive-health "${alert_txt}"
```

**Custom script:**
```bash
/usr/local/bin/send-to-monitoring-system "${alert_txt}"
```

### Skip TRIM for Specific Drives

If certain drives shouldn't be TRIMmed:

```bash
# In the TRIM section, add filtering:
/sbin/fstrim --all --verbose --quiet 2>&1 | grep -v '/dev/sda'
```

Or exclude from mount list:
```bash
for drv in "${unmounteddrvs[@]}"; do
  # Skip /dev/sdb1
  [[ "$drv" == "/dev/sdb1" ]] && continue
  # ... rest of code
done
```

### Different Defrag Schedules

Some drives need more frequent defragging than others:

**Create two scripts:**

`/etc/cron.daily/drive-health-quick`
```bash
#!/bin/bash
xfsdrives="/dev/sdc1"  # High-activity drive
# ... rest of script
```

`/etc/cron.weekly/drive-health-full`
```bash
#!/bin/bash
xfsdrives="/dev/sdc1 /dev/sdd1 /dev/sde1"  # All drives
# ... rest of script
```

### Monitor Multiple Machines

**Centralized email:**
```bash
# Each machine sends to same email
notifyemail="alerts@company.com"

# machineID identifies sender
machineID="WebServer01"
machineID="DatabaseServer"
machineID="FileServer"
```

**Email rules:**
- Filter by subject for each machine
- Create tickets automatically
- Route to on-call staff

### Integration with Existing Monitoring

**Nagios/Icinga:**
```bash
# Add to script:
if [ "${baddrive}" -eq 1 ]; then
  # Submit passive check result
  echo "DRIVE_HEALTH CRITICAL - Drive failure detected" | \
    /usr/sbin/send_nsca -H monitoring.server.com
fi
```

**Prometheus:**
```bash
# Create metrics file
echo "drive_health_status{machine=\"${machineID}\"} ${baddrive}" > \
  /var/lib/prometheus/node-exporter/drive-health.prom
```

---

## Troubleshooting

### No desktop notifications when in Test Mode

**Check notification daemons:**
```bash
# Wayland
pgrep -a mako
pgrep -a waybar-notify

# X11
pgrep -a xfce4-notifyd
pgrep -a dunst
pgrep -a mate-notification-daemon
pgrep -a notify-osd
```

**If nothing running, install one:**
```bash
# Universal (works on most DEs)
sudo apt install dunst       # Debian/Ubuntu
sudo dnf install dunst       # Fedora
sudo pacman -S dunst         # Arch

# XFCE-specific
sudo apt install xfce4-notifyd

# Wayland
sudo pacman -S mako          # Arch
sudo apt install mako-notifier  # Debian/Ubuntu (if available)
```

**Test notifications:**
```bash
notify-send "Test" "This is a test notification"
```

### Email not sending

**Check if mail command exists:**
```bash
which mail
```

**Install mail utilities:**
```bash
# Debian/Ubuntu
sudo apt install postfix mailutils
sudo dpkg-reconfigure postfix  # Choose "Internet Site"

# Fedora/RHEL
sudo dnf install postfix mailx
sudo systemctl enable postfix
sudo systemctl start postfix

# Arch
sudo pacman -S postfix mailutils
```

**Test email:**
```bash
echo "Test message" | mail -s "Test Subject" your@email.com
```

**Check mail logs:**
```bash
sudo tail -f /var/log/mail.log  # Debian/Ubuntu
sudo journalctl -u postfix -f   # systemd systems
```

**Common issues:**
- Firewall blocking port 25
- ISP blocking SMTP
- No relay host configured
- Wrong email format

**Quick fix (relay through Gmail):**
```bash
# Configure postfix to relay through Gmail
sudo postconf -e 'relayhost = [smtp.gmail.com]:587'
# ... (see postfix + Gmail tutorials for full setup)
```

### "SMART not supported" for all drives

**Verify smartmontools installed:**
```bash
which smartctl
sudo smartctl --version
```

**Test specific drive:**
```bash
sudo smartctl -i /dev/sda
```

**If shows SMART info:** Drive supports SMART, script should work.

**If says "Unknown USB":** Drive is USB without SMART support (normal).

**If command not found:**
```bash
# Install smartmontools
sudo apt install smartmontools      # Debian/Ubuntu
sudo dnf install smartmontools      # Fedora
sudo pacman -S smartmontools        # Arch
sudo slackpkg install smartmontools # Slackware
```

### TRIM not working

**Check if filesystem supports TRIM:**
```bash
sudo fstrim -v /
```

**Expected output:**
```
/: 54.3 GiB (58291863552 bytes) trimmed
```

**If error: "the discard operation is not supported"**

This is normal for:
- Hard disk drives (HDDs don't need/support TRIM)
- Older SSDs without TRIM support
- Filesystems not mounted with discard support
- USB drives

**Check mount options:**
```bash
mount | grep /dev/sda1
```

Look for `discard` in options. If missing, filesystem supports TRIM but not enabled at mount.

**Enable TRIM at mount (add to /etc/fstab):**
```bash
/dev/sda1  /  ext4  defaults,discard  0  1
```

**Or use periodic TRIM (what this script does):**
- Don't add `discard` to fstab
- Let script run `fstrim` periodically
- Less SSD wear than continuous TRIM

### xfs_fsr errors

**Verify drive is XFS:**
```bash
df -T /dev/sdc1
```

**Expected:** `Type: xfs`

**If different filesystem:** Remove from `xfsdrives` list.

**Check if mounted:**
```bash
mount | grep /dev/sdc1
```

**Requirements for XFS defrag:**
- Must be XFS filesystem
- Either mounted OR script can temporarily mount it
- Must be rotational drive (HDD, not SSD)

**Check if rotational:**
```bash
lsblk -d -o name,rota /dev/sdc
```

If `ROTA = 0`, it's an SSD. Script will skip it automatically.

### Script exits immediately

**Check for syntax errors:**
```bash
bash -n drive-health-monitor
```

**Run with debug output:**
```bash
bash -x drive-health-monitor
```

**Common issues:**
- Missing dependencies (smartctl, etc.)
- Permission errors (not running as root)
- Invalid drive paths in config
- Syntax error from manual edits

**Strict error handling means:**
- Undefined variables cause exit
- Any unhandled command failure causes exit
- This is intentional (fail-safe behavior)

---

## FAQ

### Q: Will this work on Raspberry Pi?

**A:** Yes, if:
- smartmontools is installed
- Drives support SMART (some SD cards don't)
- Script can access drive devices

Works great for Pi with USB hard drives attached.

### Q: Can I run this on a NAS?

**A:** Yes! Perfect use case:
- Set up email alerts
- Run via cron
- Monitor all drives in RAID array
- Script checks individual drives, not RAID health (use `mdadm` for that)

### Q: Does this replace regular backups?

**A:** **NO! NEVER!**

This script:
- ✅ Alerts you when drives are failing
- ✅ Gives you early warning
- ❌ Does NOT recover data
- ❌ Does NOT replace backups

SMART failures can give you time to back up before total failure. But you still need backups!

### Q: My SSD doesn't support SMART. Is that bad?

**A:** Concerning for modern SSDs.

- Very old SSDs: might not have SMART
- Cheap/counterfeit SSDs: often lack SMART
- USB adapters: might not pass through SMART commands

**Recommendation:** If storing important data on SSD without SMART, consider replacement.

### Q: Can I run this on Windows (WSL)?

**A:** Theoretically yes, but limited:
- smartctl in WSL has restricted drive access
- Better to use native Windows tools (CrystalDiskInfo, etc.)
- Or dual-boot Linux for proper drive monitoring

### Q: What about RAID health?

**A:** This script checks individual drives.

For RAID array health:
- **mdadm (software RAID):** `mdadm --detail /dev/md0`
- **Hardware RAID:** Use vendor tools (MegaCLI, etc.)
- **ZFS:** `zpool status`
- **btrfs:** `btrfs device stats`

Can combine this script with RAID monitoring for complete solution.

### Q: Why bash and not Python/systemd/etc?

**A:** Bash is ideal for system maintenance:
- ✅ Universal (every Linux has bash)
- ✅ Zero runtime dependencies
- ✅ Easy to audit (just read the script)
- ✅ Easy to modify (just edit text)
- ✅ Runs anywhere (no virtualenv, no package conflicts)
- ✅ Direct system access (no abstraction layers)

Python would add complexity without benefits for this use case.

### Q: How often should I run this?

**Recommendations:**
- **Desktop/laptop:** Daily
- **Server:** Every 6 hours
- **Critical server:** Every hour
- **NAS:** Daily

SMART failures can happen suddenly. More frequent checks = earlier warning.

### Q: Can I disable defrag but keep SMART checks?

**A:** Yes! Just set:
```bash
xfsdrives=""
```

Script will:
- Still check SMART on all drives
- Still alert on failures
- Skip defrag (empty drive list)
- Still TRIM mounted drives

### Q: What if I have no XFS drives?

**A:** Script works fine:
- SMART checks still run
- TRIM still works
- Just skips defrag section
- No errors

Default config already handles this (empty `xfsdrives` list).

---

## Contributing

### Reporting Issues

**Good bug report includes:**
- Linux distro and version
- Desktop environment (if applicable)
- Drive types (NVMe/SATA/USB)
- Error messages (full output)
- What you expected vs what happened

### Feature Requests

**Currently considering:**
- btrfs scrub support
- ZFS pool health monitoring
- Web dashboard for multi-machine monitoring
- Prometheus metrics export
- mdadm RAID health integration
- LVM thin provisioning checks

### Pull Requests

**Welcome contributions for:**
- Bug fixes
- Documentation improvements
- New notification methods
- Support for additional filesystems
- Better error handling

**Please maintain:**
- Bash compatibility (no bashisms if possible)
- Universal Linux support
- Safety-first approach
- Graceful degradation

---

## License

MIT License

Copyright (c) 2026

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

---

**TL;DR:** Use it however you want. Just don't blame me if your drive fails anyway - though if you're getting SMART alerts and ignoring them, that's on you! 😄
