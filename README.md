# Linux Drive Health Monitor

**Universal drive health monitoring and maintenance for Linux systems**

Automatically checks all drives (NVMe, SATA, USB) for SMART failures, validates XFS filesystem integrity, sends aggressive multi-channel alerts, and performs safe automated maintenance (TRIM + XFS defrag on HDDs only) when drives are healthy.

**Works everywhere:** systemd, SysVinit, elogind, Wayland, X11, any major distro.

---

## Quick Start

### 1. Install Dependencies

**Required:**
```bash
# Debian/Ubuntu
sudo apt install smartmontools

# Fedora/RHEL
sudo dnf install smartmontools

# Arch
sudo pacman -S smartmontools

# Slackware
sudo slackpkg install smartmontools
```

**Optional (for full functionality):**
```bash
# Debian/Ubuntu
sudo apt install util-linux xfsprogs zenity libnotify-bin

# Fedora/RHEL
sudo dnf install util-linux xfsprogs zenity libnotify

# Arch
sudo pacman -S util-linux xfsprogs zenity libnotify
```

### 2. Download & Configure

```bash
wget https://raw.githubusercontent.com/gitdwarf/linux-drive-health/main/drive-health-monitor
chmod +x drive-health-monitor
```

Edit the top of the script:
```bash
machineID="HomeServer"                    # Your machine name (for alerts)
notifyemail="admin@example.com"           # Where to send email alerts
xfsdrives="/dev/sdc1 /dev/sdd1"           # XFS HDDs to defrag (SSDs auto-skipped)
unmounteddrives="/dev/sdb1"               # Unmounted drives to TRIM (rescue partitions, hot spares)
mountbase="/mnt/scratch"                  # Temporary mount point
```

**Drive specification supports:**
- Direct device paths: `/dev/sdc1`
- UUID identifiers: `UUID=a1b2c3d4-...`
- LABEL identifiers: `LABEL=MyDrive`

### 3. Test Run

**Test mode (safe, non-critical alerts):**
```bash
sudo -s
./drive-health-monitor --test
```

Expected output if all healthy:
```
Checking SMART status on drives
All SMART supported drives report they PASS SMART check
Performing unicode check on all compatible drives
All XFS drives checked clean of any invalid unicode issues
Performing TRIM on all compatible drives
/mnt/mountpoint : 232.9 GiB (250058113024 bytes) trimmed on /dev/sdc1
Performing defrag on all defined drives
Defrag completed for drives: /dev/sdc1 /dev/sdd1
```

**Production run:**
```bash
sudo -s
./drive-health-monitor
```

### 4. Automate (Recommended)

**Daily cron job:**
```bash
sudo cp drive-health-monitor /etc/cron.daily/250-drive-health-monitor
sudo chmod +x /etc/cron.daily/250-drive-health-monitor
```

**Or custom schedule:**
```bash
sudo crontab -e
# Run every 6 hours:
0 */6 * * * /path/to/drive-health-monitor
```

---

## What It Does

### Processing Flow

The script runs in this order:

1. **SMART Health Check** (all NVMe + SATA/USB drives)
   - ✅ Checks all drives for SMART errors
   - ✅ Gracefully skips drives without SMART support
   - ✅ **Zero false negatives:** Any non-PASSED result triggers full alert cascade
   - ❌ **If ANY drive fails:** Skip all maintenance, send alerts, exit

2. **XFS Unicode Validation** (all XFS drives)
   - ✅ Scans XFS filesystems for invalid UTF-8 filenames
   - ✅ Checks both mounted and unmounted XFS drives
   - ✅ Reports specific inodes and file paths with problems
   - ❌ **If unicode issues found:** Skip TRIM and defrag, send alert email

3. **TRIM Operations** (all compatible drives)
   - ✅ TRIM unmounted drives (mounts temporarily, TRIMs, unmounts)
   - ✅ TRIM all mounted drives (`fstrim --all`)
   - ⏭️  **Skipped if:** SMART failures or XFS unicode issues detected

4. **XFS Defragmentation** (XFS HDDs only)
   - ✅ **Automatically skips SSDs** (checks ROTA value)
   - ✅ Only defrags spinning hard drives with XFS
   - ✅ Reports any defrag errors via email
   - ⏭️  **Skipped if:** SMART failures, XFS unicode issues, or `xfs_fsr` already running

### Alert System (When Problems Found)

**Everyone gets notified - no one misses a problem:**

- **Desktop users** (all logged-in users):
  - Zenity error popup (or info popup in test mode)
  - Critical desktop notification (or normal in test mode)
  
- **Terminal users** (SSH, tmux, screen, TTY):
  - Alert broadcast to all open terminals
  - Persistent alerts on next login (via `/etc/profile.d`)
  
- **Email:**
  - Aggregated report of all errors
  - Priority headers (Critical for SMART failures, High for unicode/defrag issues)
  - Custom headers for email filtering

- **Console:**
  - Printed to stdout if running interactively

**Test mode alerts:**
- Normal urgency (not critical)
- "JUST A TEST!" headers
- Email subject: "TESTING OF DRIVE ALERTS"
- GUI alerts show as info dialogs (not warnings)
- No persistent offline alerts created

---

## Safety Features

**Why it's safe to automate:**

1. **SMART check first** - If ANY drive fails, all maintenance skipped
2. **XFS unicode validation** - Invalid filenames prevent TRIM/defrag
3. **Mount point conflict detection** - Won't overwrite active mounts
4. **Wait loops** - Ensures mounts complete before operations
5. **SSD auto-detection** - Won't defrag SSDs even if listed in `xfsdrives`
6. **Process conflict prevention** - Won't start defrag if `xfs_fsr` already running
7. **Single instance lock** - Only one copy can run at a time
8. **Strict error handling** - Script exits on unhandled errors
9. **Full root required** - Rejects `sudo` to avoid permission issues

---

## Compatibility

**Works on:**
- ✅ Any major Linux distro (Ubuntu, Debian, Fedora, Arch, Slackware, etc.)
- ✅ Systemd OR non-systemd (uses elogind for Wayland detection)
- ✅ Wayland OR X11 (auto-detects desktop environment)
- ✅ Desktop OR server (works headless with email-only)
- ✅ NVMe + SATA + USB drives

**Notification daemon support:**
- Wayland: mako, waybar-notify
- X11: xfce4-notifyd, dunst, mate-notification-daemon, notify-osd

Script gracefully degrades if optional components missing.

---

## Configuration Details

### `machineID`
Used in email subject lines and alerts to identify which machine has the problem.

### `notifyemail`
Email address for all alerts (SMART failures, unicode issues, defrag warnings).  
Requires working `mail` command. Defaults to `root` if not changed.

### `xfsdrives`
**Space-separated list of XFS-formatted drives to defragment.**

**IMPORTANT:** Only spinning HDDs are defragged - SSDs are automatically detected and skipped.

Supports multiple formats:
```bash
xfsdrives="/dev/sdc1 /dev/sdd1"              # Direct device paths
xfsdrives="UUID=abc123... UUID=def456..."    # UUIDs
xfsdrives="LABEL=Data LABEL=Backup"          # Labels
```

The script checks drive type: `lsblk -d -o rota` (1 = HDD, 0 = SSD)

### `unmounteddrives`
**Space-separated list of unmounted drives to TRIM.**

Use cases:
- Hot-swap spare drives (installed but unmounted)
- Rescue/recovery partitions
- Secondary internal SSDs not auto-mounted

**NOT for:** External drives that might be unplugged!

Supports same formats as `xfsdrives` (device paths, UUIDs, LABELs).

### `mountbase`
Temporary mount point for unmounted drive operations.  
Auto-creates `/tmp/drivechkmount` if you don't change the defaults.

---

## Test Mode

**Purpose:** Verify the alert system works without sending critical alerts.

**Usage:**
```bash
sudo -s
./drive-health-monitor --test
```

**What's different in test mode:**

| Aspect | Live Mode | Test Mode |
|--------|-----------|-----------|
| **SMART check** | `PASSED` required | `TESTING` required (always passes) |
| **Alert urgency** | Critical | Normal (info) |
| **Email subject** | `CRITICAL ERROR ALERT` | `TESTING OF DRIVE ALERTS` |
| **Email priority** | Highest (1) | Normal |
| **GUI popup** | Red warning dialog | Blue info dialog |
| **Terminal color** | Red | Default |
| **Alert headers** | `!JUST A TEST!` prefix | None |
| **Offline alerts** | Created in `/etc/profile.d` | Not created |

**Perfect for:**
- Testing notification delivery
- Verifying email configuration
- Checking desktop notifications work
- Training users what alerts look like
- Cron job dry runs

---

## Troubleshooting

### No desktop notifications?

Install a notification daemon:
```bash
# Any of these will work:
sudo apt install dunst          # Universal
sudo apt install xfce4-notifyd  # XFCE
sudo dnf install mako           # Wayland
```

### Email not sending?

Configure an MTA (Mail Transfer Agent):
```bash
sudo apt install postfix mailutils
sudo dpkg-reconfigure postfix  # Follow prompts
```

Test: `echo "test" | mail -s "test" your@email.com`

### "SMART not supported" for all drives?

Ensure smartmontools installed:
```bash
sudo smartctl -i /dev/sda  # Should show SMART info
```

### TRIM not working?

Check if your filesystem supports it:
```bash
sudo fstrim -v /  # Should show bytes trimmed
```

If error: Drive/filesystem doesn't support TRIM (normal for spinning HDDs).

### XFS unicode warnings?

The script will report specific file paths with invalid UTF-8 filenames.  
**Fix:** Rename or delete the problematic files, then re-run:
```bash
# Find the file by inode:
sudo find /mountpoint -xdev -inum 12345

# Rename to valid UTF-8:
sudo mv /path/to/bad-file /path/to/good-file
```

### Script requires "full root"?

This is intentional. Don't use `sudo ./script` - instead:
```bash
sudo -s                    # Become root
./drive-health-monitor     # Run as actual root
```

Or:
```bash
su -                       # Become root
./drive-health-monitor
```

Why: Ensures proper permissions for mounting/unmounting, TRIM, defrag operations.

---

## Examples

**Check specific drive manually:**
```bash
sudo smartctl -H /dev/sda
```

**See what drives will be processed:**
```bash
ls /dev/nvme?n?  # NVMe drives
ls /dev/sd?      # SATA/USB drives
```

**Test TRIM on specific mount:**
```bash
sudo fstrim -v /mnt/backup
```

**Check XFS filesystem for unicode issues manually:**
```bash
sudo xfs_scrub -n -v /mnt/data
```

**See if a drive is rotational (HDD) or SSD:**
```bash
lsblk -d -o name,rota /dev/sdc
# Output: 1 = HDD, 0 = SSD
```

---

## Advanced Usage

See [DOCUMENTATION.md](DOCUMENTATION.md) for:
- Technical details on alert system implementation
- Customizing notification methods
- Adding Slack/Discord webhooks
- Multi-machine monitoring setup
- Email filtering rules
- btrfs/ZFS integration ideas

---

## What's New

**Version 2.1:**
- **Fixed singleton check:**
- **XFS Unicode Validation:** Pre-maintenance scanning prevents data corruption from invalid filenames
- **UUID/LABEL Support:** Specify drives by UUID or LABEL, not just device paths
- **Test Mode Improvements:** Clearer separation between test and live alerts
- **Better Mount Handling:** Proper normalization and conflict detection
- **Enhanced Reporting:** Per-inode, per-drive reporting of unicode issues
- **Conditional Safety:** TRIM and defrag now skip if unicode problems detected

---

## Contributing

Found a bug? Have a feature request? PRs welcome!

**Potential future features:**
- Support for btrfs scrub
- Support for ZFS pool health
- Prometheus metrics export
- Web dashboard for multi-machine monitoring
- Configurable alert thresholds

---

## License

MIT License - Free to use, modify, and distribute.

**TL;DR:** Use it however you want. Just don't blame me if your drive fails anyway - though if you're getting SMART alerts and ignoring them, that's on you! 😄

---

## Credits

Built for real-world sysadmin use. Tested on:
- Slackware (SysVinit + elogind)
- Ubuntu (systemd)
- Multiple desktop environments
- Both Wayland and X11 sessions

**Aggressive alerting philosophy:** If a drive is dying or has filesystem corruption, EVERYONE should know immediately.

**Prevention over reaction:** Unicode validation prevents data corruption before it happens.
