# Linux Drive Health Monitor - Full Documentation

**Complete technical reference and advanced configuration guide**

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [How It Works](#how-it-works)
- [Processing Flow](#processing-flow)
- [XFS Unicode Validation](#xfs-unicode-validation)
- [Alert System](#alert-system)
- [Configuration](#configuration)
- [Safety & Error Handling](#safety--error-handling)
- [Technical Details](#technical-details)
- [Advanced Usage](#advanced-usage)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)

---

## Overview

This script provides comprehensive drive health monitoring with XFS filesystem validation, aggressive multi-channel alerting, and safe automated maintenance.

**Design Philosophy:**
1. **Safety first** - Never write to failing drives or corrupted filesystems
2. **Alert everyone** - No one should miss a drive failure or data corruption risk
3. **Graceful degradation** - Works even if optional components missing
4. **Universal compatibility** - Works on any Linux distro/desktop
5. **Prevention over reaction** - Validate before maintenance, not after

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

**XFS Filesystem Validation:**
- Scans all XFS filesystems for invalid UTF-8 filenames
- Checks both mounted and unmounted XFS drives
- Reports specific inodes and full file paths
- Prevents TRIM and defrag if corruption detected

### Automated Maintenance

**TRIM Operations:**
- Unmounted drives: Mounts temporarily, TRIMs, unmounts safely
- Mounted drives: Uses `fstrim --all`
- Conditional: Skipped if SMART failures or XFS unicode issues

**XFS Defragmentation:**
- **HDD-only:** Automatically detects and skips SSDs
- Checks if `xfs_fsr` already running (prevents conflicts)
- Reports errors via email
- Conditional: Skipped if SMART failures or XFS unicode issues

### Alert System

Multi-channel notifications ensure no one misses problems:

**Desktop Notifications:**
- Zenity GUI popups
- Desktop notification daemon alerts
- Auto-start alerts on next login if user was offline

**Terminal Alerts:**
- Broadcast to all open TTYs, PTYs, screen, tmux sessions
- Persistent `/etc/profile.d` alerts for future logins

**Email Alerts:**
- Aggregated problem reports
- Custom priority headers for filtering
- Separate subjects for different alert types

---

## How It Works

### Execution Model

The script is designed to run as a **root cron job** with full system access.

**Requirements:**
- Must run as actual root (not via `sudo`)
- Single instance lock (via `pgrep`)
- Strict error handling (`set -euo pipefail`)

### Processing Flow

```
┌─────────────────────────────┐
│  Start (as root)            │
│  Parse --test flag          │
│  Set alert severity         │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│  SMART Health Check         │
│  ✓ All NVMe drives          │
│  ✓ All SATA/USB drives      │
└──────────┬──────────────────┘
           │
           ├─ FAIL ──► Alert All → Exit
           │
           ▼ PASS
┌─────────────────────────────┐
│  XFS Unicode Validation     │
│  ✓ Unmounted XFS drives     │
│  ✓ Mounted XFS drives       │
└──────────┬──────────────────┘
           │
           ├─ ISSUES ──► Alert Email → Skip Maintenance
           │
           ▼ CLEAN
┌─────────────────────────────┐
│  TRIM Operations            │
│  ✓ Unmounted drives         │
│  ✓ All mounted drives       │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│  XFS Defragmentation        │
│  ✓ Only HDDs (skip SSDs)    │
│  ✓ Check for running xfs_fsr│
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│  Send Email (if alerts)     │
│  Exit                        │
└─────────────────────────────┘
```

**Key Points:**
- **Fail-fast:** Any SMART failure stops all processing
- **Conditional execution:** XFS unicode issues prevent TRIM/defrag
- **Independent alerts:** SMART failures trigger all channels; unicode/defrag issues only send email

---

## XFS Unicode Validation

### Why It Matters

XFS allows filenames that don't conform to valid UTF-8. These can cause:
- **Data corruption** during defragmentation
- **Backup failures** when tools can't handle invalid encoding
- **TRIM errors** on some SSD firmware
- **Replication issues** in distributed filesystems

### How Detection Works

The script uses `xfs_scrub -n -v` to perform a **read-only** scan:

```bash
xfs_scrub -n -v /mountpoint
```

**Flags:**
- `-n`: Read-only, no repairs
- `-v`: Verbose output (shows inode numbers)

**What's checked:**
- Metadata integrity
- Directory tree structure
- **Unicode conformance in filenames**

### Output Processing

The script:
1. Filters out informational messages (phase progress, thread count, etc.)
2. Extracts unicode warnings: `Warning: inode 12345 has invalid Unicode name`
3. Maps inodes to actual file paths: `find /mnt -xdev -inum 12345`
4. Reports full paths: `'/dev/sdc1/path/to/bad-file.txt'`

### Alert Behavior

**If unicode issues detected:**
- Email sent with:
  - Subject: `[MachineID] Invalid unicode issue`
  - Priority: High
  - Full list of affected files with paths
- TRIM operations: **Skipped**
- Defragmentation: **Skipped**
- Exit code: 0 (not a fatal error, just a warning)

**Rationale:** TRIM and defrag can crash or corrupt data when operating on filesystems with invalid filenames. Better to alert and skip than risk corruption.

### Manual Remediation

```bash
# Find the problematic file
sudo find /mnt/data -xdev -inum 12345

# Option 1: Rename to valid UTF-8
sudo mv /mnt/data/bad€file.txt /mnt/data/badfile.txt

# Option 2: Delete if safe
sudo rm /mnt/data/bad€file.txt

# Re-run validation
sudo xfs_scrub -n -v /mnt/data
```

---

## Alert System

### Multi-Channel Design

The goal: **Every user and admin knows immediately when something's wrong.**

#### 1. Desktop Users (GUI)

**Live alerts (if user logged in):**
- `notify-send`: Desktop notification (critical urgency, or normal in test mode)
- `zenity`: Modal dialog popup (must be dismissed)

**Offline alerts (if user was not logged in):**
- Creates `/usr/local/bin/drive-health-monitor-user_alert/`
- Generates autostart .desktop file
- Symlinks to `~/.config/autostart/` for all users
- Next login: Popup appears immediately

**Wayland detection:**
- Checks `loginctl` for Wayland sessions
- Starts appropriate notification daemons (mako, waybar-notify)

**X11 support:**
- Starts appropriate daemons (dunst, xfce4-notifyd, etc.)

#### 2. Terminal Users (SSH/TTY)

**Live alerts:**
- Broadcasts to all open terminals: `/dev/pts/*`, `/dev/tty/*`
- Works in: tmux, screen, direct SSH, console TTY

**Offline alerts:**
- Creates `/etc/profile.d/drive-health-alert.sh`
- Sources on next shell login
- Prints unavoidable warning at prompt

#### 3. Email Alerts

**Headers for filtering:**

| Alert Type | Priority Header | Custom Header |
|------------|----------------|---------------|
| **SMART Failure** | `X-Priority: 1 (Highest)` | `drive-health-monitor-status: LIVE` |
| **XFS Unicode** | `X-Priority: 2 (High)` | `drive-health-monitor-status: UNICODE_ISSUE` |
| **Defrag Warning** | `X-Priority: 2 (High)` | `drive-health-monitor-status: DEFRAG_WARNING` |
| **Test Mode** | Normal | `drive-health-monitor-status: JUST A TEST` |

**Email filtering examples:**

Outlook/Thunderbird rule:
```
If header "drive-health-monitor-status" contains "LIVE"
  → Move to "Critical Alerts" folder
  → Play sound
  → Mark as important
```

Gmail filter:
```
Matches: has:drive-health-monitor-status LIVE
Do this: Star it, Apply label "CRITICAL", Never send to spam
```

### Test Mode Differences

| Feature | Live Mode | Test Mode |
|---------|-----------|-----------|
| **SMART check** | Looks for `PASSED` | Looks for `TESTING` (always true) |
| **Notification urgency** | Critical | Normal |
| **Zenity dialog** | `--warning` (red) | `--info` (blue) |
| **Terminal color** | Red (`\033[31m`) | Default (`\033[0m`) |
| **Email subject** | `CRITICAL ERROR ALERT` | `TESTING OF DRIVE ALERTS` |
| **Email priority** | Highest (1) | Normal |
| **Offline alerts** | Created | Not created |
| **Alert headers** | Prefixed with `!JUST A TEST!` | None |

**Use test mode to:**
- Verify mail delivery works
- Check desktop notifications appear
- Train users what alerts look like
- Debug notification daemons

---

## Configuration

### Drive Specification Formats

All drive configuration variables support three formats:

#### 1. Direct Device Paths
```bash
xfsdrives="/dev/sdc1 /dev/sdd1"
unmounteddrives="/dev/sdb1"
```

#### 2. UUID
```bash
xfsdrives="UUID=a1b2c3d4-e5f6-7890-abcd-ef1234567890 UUID=12345678-..."
unmounteddrives="UUID=abcd1234-..."
```

#### 3. LABEL
```bash
xfsdrives="LABEL=Data LABEL=Backup"
unmounteddrives="LABEL=Rescue"
```

**Normalization process:**

```bash
fn_normalise_drive() {
  # Strip UUID=/LABEL= prefix
  # Check /dev/disk/by-uuid/
  # Check /dev/disk/by-label/
  # Validate device path format
  # Set $mountdrive or empty if invalid
}
```

**Why normalization matters:**
- UUIDs persist across reboots (device names may change)
- LABELs are human-readable
- Script validates the drive exists before attempting operations

### Variable Details

#### `machineID`
```bash
machineID="Fileserver-01"
```
Used in:
- Email subjects: `[Fileserver-01] CRITICAL ERROR ALERT`
- Alert messages: `Drive failure on Fileserver-01`

#### `notifyemail`
```bash
notifyemail="sysadmin@example.com,backup@example.com"
```
Comma-separated list supported by most MTAs.

**Defaults to `root`** if unchanged — requires local mail delivery working.

#### `unmounteddrives`
```bash
unmounteddrives="/dev/sdb1 UUID=abc123... LABEL=HotSpare"
```

**Valid use cases:**
- Hot-swap bays with drives always installed but not mounted
- Rescue/recovery partitions on internal drives
- Secondary SSDs used for backups (mounted on-demand)

**Invalid use cases:**
- External USB drives that might be unplugged (will cause errors)
- Network shares (not block devices)

#### `mountbase`
```bash
mountbase="/mnt/drive-health-scratch"
```

**Requirements:**
- Must be a dedicated mount point (script checks for conflicts)
- Must not be in use by other processes
- Automatically created if it doesn't exist

**Default:** `/tmp/drivechkmount` (created automatically)

#### `xfsdrives`
```bash
xfsdrives="/dev/sdc1 /dev/sdd1 UUID=abc... LABEL=Data"
```

**SSD handling:**
- Script checks `lsblk -d -o rota`
- `rota=1` → HDD (defrag runs)
- `rota=0` → SSD (defrag skipped with message)

**Why:** XFS defragmentation on SSDs is unnecessary (no seek time) and potentially harmful (increases write amplification).

---

## Safety & Error Handling

### Multiple Layers of Protection

#### 1. Access Control
```bash
# Reject non-root
if [ "$(id -u)" -ne 0 ]; then
  exit 1
fi

# Reject sudo (want actual root)
if [ -n "${SUDO_USER:-}" ]; then
  exit 1
fi
```

**Rationale:** Mounting, unmounting, TRIM, and defrag require root. Running via `sudo` can cause permission issues with notification daemons and profile.d alerts.

#### 2. Single Instance Lock
```bash
for pid in $(pgrep -f "$(basename "$0")"); do
  if [ "$pid" != "$$" ]; then
    exit 1
  fi
done
```

**Prevents:**
- Multiple defrag processes (can corrupt data)
- Conflicting mount operations
- Race conditions on `/etc/profile.d` alert file

#### 3. Strict Error Handling
```bash
set -euo pipefail
```

- `set -e`: Exit on any command failure
- `set -u`: Exit on undefined variable use
- `set -o pipefail`: Detect failures in pipes

**Exception:** Explicitly allowed failures use `|| true`

#### 4. Mount Point Validation
```bash
if mountpoint -q "${mountbase}"; then
  echo "Mount point already in use"
  return
fi
```

**Prevents:**
- Overwriting active mounts
- Data loss from incorrect mount targets

#### 5. Wait Loops
```bash
mount "${drive}" "${mountbase}"
while ! mountpoint -q "${mountbase}"; do
  sleep 0.5
done
```

**Ensures:**
- Mount completes before TRIM
- Unmount completes before script exits
- No race conditions

#### 6. SSD Detection
```bash
is_rot=$(lsblk -d -o rota -n "${drive}")
if [ "${is_rot}" -eq 1 ]; then
  # Only defrag HDDs
fi
```

**Prevents:**
- Unnecessary SSD wear
- Wasted time (defrag has no benefit on SSDs)

#### 7. Process Conflict Detection
```bash
if pgrep -i xfs_fsr >/dev/null; then
  echo "xfs_fsr already running - skipping defrag"
  return
fi
```

**Prevents:**
- Multiple defrag processes fighting for I/O
- Possible filesystem corruption

---

## Technical Details

### SMART Monitoring

**Detection logic:**
```bash
if smartctl -i "${drive}" | grep -q "SMART support is: Available"; then
  if ! smartctl -H "${drive}" | grep -q "${smartctltest}"; then
    # SMART failure detected
  fi
else
  # SMART not supported, skip drive
fi
```

**Variables:**
- Live mode: `smartctltest="PASSED"`
- Test mode: `smartctltest="TESTING"`

**Why not just check exit codes?**

`smartctl` exit codes are complex:
- Bit 0: Command line parsing error
- Bit 1: Device open error
- Bit 2: SMART command failed
- Bit 3: Disk failing
- Bit 4: Prefail attribute <= threshold
- Bit 5: Previous error in SMART log
- Bit 6: Errors in selftest log
- Bit 7: New errors in selftest log

Parsing `SMART overall-health self-assessment test result` is more reliable.

### TRIM Implementation

**Two-phase approach:**

**Phase 1: Unmounted drives**
```bash
for drive in "${unmounteddrives[@]}"; do
  fn_mountdrive
  fstrim "$mountbase" --verbose --quiet
  fn_unmountdrive
done
```

**Phase 2: All mounted drives**
```bash
fstrim --all --verbose --quiet
```

**Flags:**
- `--verbose`: Print bytes trimmed
- `--quiet`: Suppress errors for filesystems that don't support TRIM

### XFS Defrag Implementation

**Safety checks before defrag:**
1. Is `xfsdrives` variable set?
2. Are there XFS unicode issues?
3. Is `xfs_fsr` already running?
4. Is the drive rotational (HDD)?
5. Is the mount point available?

**Defrag execution:**
```bash
xfs_fsr "${drive}" 2>&1 | grep -v "start inode=0"
```

**Output filtering:**
- `start inode=0`: Normal progress message (filtered out)
- Anything else: Potential error/warning (reported via email)

---

## Advanced Usage

### Email Filtering Setup

**Gmail:**
```
1. Settings → Filters and Blocked Addresses → Create new filter
2. Has the words: drive-health-monitor-status:LIVE
3. Star it, Apply label "CRITICAL-ALERTS", Never send to Spam
4. Optional: Forward to SMS gateway
```

**Postfix relay to external SMTP:**
```bash
# /etc/postfix/main.cf
relayhost = [smtp.gmail.com]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_security_level = encrypt

# /etc/postfix/sasl_passwd
[smtp.gmail.com]:587 youremail@gmail.com:app-password

# Apply
sudo postmap /etc/postfix/sasl_passwd
sudo systemctl restart postfix
```

### Multi-Machine Monitoring

**Centralized email:**
```bash
# On all servers:
notifyemail="central-monitoring@example.com"

# Set unique machineID:
machineID="WebServer-01"   # Server 1
machineID="DBServer-02"    # Server 2
machineID="FileServer-03"  # Server 3
```

**Email filtering by machine:**
```
Subject contains "WebServer-01" → Label: "Webserver Alerts"
Subject contains "DBServer-02" → Label: "Database Alerts"
```

### Webhook Integration

**Add to end of script (before email send):**

```bash
# Slack webhook
if [ -n "${alert_email}" ]; then
  curl -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"${alert_emailsubject}\n${alert_email}\"}" \
    https://hooks.slack.com/services/YOUR/WEBHOOK/URL
fi

# Discord webhook
if [ -n "${alert_email}" ]; then
  curl -H "Content-Type: application/json" \
    -d "{\"content\":\"**${alert_emailsubject}**\n${alert_email}\"}" \
    https://discord.com/api/webhooks/YOUR/WEBHOOK
fi
```

### Custom Notification Sounds

**Add to GUI alert function:**
```bash
fn_drive-health-monitor-user_alert_gui_notifications () {
  sed 's/^[[:space:]]*//' << eof >> "$offline_gui_alert_script"
  #!/bin/bash
  
  # Play alert sound
  paplay /usr/share/sounds/freedesktop/stereo/alarm-clock-elapsed.oga
  
  notify-send -i ${notify_icon} -u ${notify_urgency} "${notify_title}" "${alert_combined}" &>/dev/null &
  zenity ${zenity_urgency} --text="<span ${zenity_colour}>${alert_combined}</span>" &>/dev/null &
eof
}
```

### Prometheus Metrics Export

**Add to end of script:**
```bash
# Write metrics to node_exporter textfile directory
cat > /var/lib/node_exporter/textfile_collector/drive_health.prom << EOF
# HELP drive_health_smart_status SMART health status (1=healthy, 0=failing)
# TYPE drive_health_smart_status gauge
drive_health_smart_status{device="/dev/sda"} $([ $baddrive -eq 0 ] && echo 1 || echo 0)

# HELP drive_health_xfs_unicode XFS unicode validation status (1=clean, 0=issues)
# TYPE drive_health_xfs_unicode gauge
drive_health_xfs_unicode $([ $xfs_unicode_bad -eq 0 ] && echo 1 || echo 0)
EOF
```

---

## Troubleshooting

### Common Issues

#### "Must run as FULL ROOT"

**Problem:** Script detects `sudo` or non-root user.

**Solution:**
```bash
# Wrong:
sudo ./drive-health-monitor

# Right:
sudo -s
./drive-health-monitor

# Or:
su -
./drive-health-monitor
```

#### "Mount point already in use"

**Problem:** `$mountbase` is currently mounted by another process.

**Causes:**
- Another instance of the script running (single-instance lock should prevent this)
- Manual mount left active
- Different script using same mount point

**Solution:**
```bash
# Check what's mounted:
mount | grep drivechkmount

# Unmount if safe:
sudo umount /tmp/drivechkmount

# Or change mountbase in script:
mountbase="/mnt/drive-health-alt"
```

#### Email not sending

**Check MTA installed:**
```bash
which mail
dpkg -l | grep postfix
```

**Test mail delivery:**
```bash
echo "test" | mail -s "test" root
tail -f /var/log/mail.log
```

**Common fixes:**
```bash
# Install postfix
sudo apt install postfix mailutils

# Reconfigure
sudo dpkg-reconfigure postfix

# Start service
sudo systemctl start postfix
sudo systemctl enable postfix
```

#### Desktop notifications not appearing

**Check notification daemon:**
```bash
# Wayland
pgrep -a mako

# X11
pgrep -a dunst
pgrep -a xfce4-notifyd

# Install if missing
sudo apt install dunst
```

**Test manually:**
```bash
notify-send -u critical "Test" "Alert test"
```

#### XFS unicode check takes forever

**Expected on large filesystems.** XFS scrub reads metadata for every inode.

**Rough timing:**
- 100GB with 10K files: 30 seconds
- 1TB with 100K files: 2-3 minutes
- 10TB with 1M files: 10-15 minutes

**Not a bug.** This is proper validation.

#### Defrag fails with "device busy"

**Causes:**
- Another `xfs_fsr` process running
- Drive mounted but script trying to mount it to `$mountbase`

**Solutions:**
```bash
# Kill other xfs_fsr
pkill xfs_fsr

# Remove drive from xfsdrives if already mounted
# (script will defrag in-place)
```

---

## FAQ

### Q: Why require "full root" instead of sudo?

**A:** Notification delivery to user sessions, `/etc/profile.d` creation, and certain mount operations can fail with inconsistent UIDs when using `sudo`. Actual root ensures clean permissions.

### Q: Will this work on btrfs/ext4/ZFS?

**A:** Partially.
- **SMART checks:** Yes (works on any drive)
- **TRIM:** Yes (works on any filesystem supporting TRIM)
- **XFS unicode validation:** No (XFS-specific)
- **XFS defrag:** No (XFS-specific)

For btrfs, you'd want `btrfs scrub` instead. For ZFS, `zpool scrub`. This script focuses on XFS.

### Q: How often should I run this?

**A:** Daily is recommended for production systems. For home use, 2-3 times per week is fine.

SMART status can change suddenly. Running daily ensures you catch problems early.

### Q: Can I run this on a laptop?

**A:** Yes, but:
- Set `unmounteddrives=""` (external drives may not be present)
- Consider battery impact (defrag is I/O intensive)
- Maybe only run when plugged in (check `/sys/class/power_supply/AC/online`)

### Q: What if I have no XFS drives?

**A:** Set `xfsdrives=""` - the XFS checks will be skipped entirely.

### Q: Does test mode actually test anything?

**A:** It tests the **alert delivery system**, not drive health.

Test mode verifies:
- Email sends correctly
- Desktop notifications appear
- Terminal broadcasts work
- Users would actually see alerts

It does NOT test whether drives are actually healthy.

### Q: Why not use smartd instead?

**A:** `smartd` is great for **monitoring**, but this script adds:
- Aggressive multi-channel alerts (GUI + terminal + email)
- XFS validation before maintenance
- Automated TRIM
- Automated defrag with safety checks
- Test mode for alert verification

They're complementary. You can run both.

### Q: What's the performance impact?

**Typical run (healthy system, 4 drives, 2 XFS):**
- SMART checks: 2-3 seconds
- XFS unicode validation: 30 seconds - 3 minutes (depends on file count)
- TRIM: 1-10 seconds (depends on free space)
- Defrag: 5 minutes - 2 hours (depends on fragmentation)

Total: 6 minutes to 2+ hours.

**CPU impact:** Minimal (mostly I/O bound)  
**I/O impact:** Moderate during defrag, low otherwise

### Q: Can I customize alert messages?

**A:** Yes. Edit the functions:
- `fn_smartcheck()`: SMART failure messages
- `fn_xfs_unicode()`: Unicode issue messages
- Alert variables at top of script

### Q: Will this prevent drive failures?

**A:** No. It **detects** failures early and **prevents data corruption** from defragging damaged filesystems.

SMART alerts give you time to:
- Order replacement drives
- Schedule downtime
- Back up critical data
- Replace drives before catastrophic failure

It's a smoke alarm, not a fire suppression system.

---

## Changelog

### Version 2.0
- **XFS Unicode Validation:** Pre-maintenance scanning prevents data corruption
- **UUID/LABEL Support:** Flexible drive specification
- **Improved Mount Handling:** Better conflict detection and normalization
- **Enhanced Test Mode:** Clearer distinction from live alerts
- **Better Reporting:** Per-inode unicode issue reporting

### Version 1.5
- **Test Mode:** `--test` flag for safe alert testing
- **Wayland Support:** Detection and notification daemon handling
- **Multi-channel Alerts:** GUI + terminal + email
- **Offline Alerts:** Persistent notifications via autostart + profile.d

### Version 1.0
- Initial release
- SMART monitoring
- TRIM automation
- XFS defragmentation
- Email alerts

---

**Last Updated:** February 2026  
**Maintainer:** thedwarf  
**License:** MIT
