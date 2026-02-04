# Linux Drive Health Monitor

**Universal drive health monitoring and maintenance for Linux systems**

Automatically monitors all drives (NVMe, SATA, USB) for SMART failures, sends aggressive multi-channel alerts, and performs safe automated maintenance (TRIM + XFS defrag on HDDs only) when drives are healthy.

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
machineID="HomeServer"           # Your machine name (for alerts)
notifyemail="admin@example.com"  # Where to send email alerts
xfsdrives="/dev/sdc1 /dev/sdd1"  # XFS HDDs to defrag (SSDs auto-skipped)
unmounteddrvs="/dev/sdb1"        # Unmounted drives to TRIM (rescue partitions, hot spares)
mountbase="/mnt/scratch"         # Temporary mount point
```

### 3. Test Run

```bash
sudo ./drive-health-monitor
```

Expected output if all healthy:
```
Checking SMART status on drives
Skipping /dev/sda - SMART not supported
/mnt/mountpoint : 232.9 GiB (250058113024 bytes) trimmed on /dev/sdc1
Defrag completed for drives: /dev/sdc1 /dev/sdd1
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

### Health Monitoring
- ✅ Checks **all NVMe and SATA/USB drives** for SMART errors
- ✅ Gracefully skips drives without SMART (USB sticks, etc.)
- ✅ **Zero false negatives:** Any non-PASSED result triggers alerts

### Alert System (When Drive Fails)
**Everyone gets notified - no one misses a failing drive:**

- **Desktop users** (all logged-in users):
  - Zenity error popup
  - Critical desktop notification
  
- **Terminal users** (SSH, tmux, screen, TTY):
  - Alert broadcast to all open terminals
  
- **Email:**
  - Aggregated report of all errors

- **Console:**
  - Printed to stdout if running interactively

### Automated Maintenance (Only When All Drives Healthy)

**If ANY drive fails SMART, no maintenance runs.**

When all drives healthy:

1. **TRIM unmounted drives**
   - Temporarily mounts defined unmounted drives (hot spares, rescue partitions, and the like)
   - Performs TRIM
   - Safely unmounts

2. **TRIM all mounted drives**
   - Uses `fstrim --all`

3. **XFS defragmentation (Spnning HDDs only)**
   - **Automatically skips SSDs** (checks ROTA value)
   - Only defrags spinning hard drives partitioned and formatted with XFS
   - Reports any defrag errors via email

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

## Safety Features

**Why it's safe to automate:**

1. **SMART check first** - If ANY drive fails, maintenance skipped
2. **Mount point conflict detection** - Won't overwrite active mounts
3. **Wait loops** - Ensures mounts complete before operations
4. **SSD auto-detection** - Won't defrag SSDs even if listed
5. **Process conflict prevention** - Won't start defrag if already running
6. **Strict error handling** - Script exits on unhandled errors

---

## Configuration Details

### `machineID`
Used in email subject lines to identify which machine sent the alert.

### `notifyemail`
Email address for error reports. Requires working `mail` command.
Defaults to `root` if not changed.

### `xfsdrives`
**IMPORTANT: Only list XFS-formatted SPINNING HARD DRIVES!**

- Script auto-detects SSDs and skips defrag for them
- Still list them if you want - they'll be safely ignored
- Example: `xfsdrives="/dev/sdc1 /dev/sdd1"`
- Checks drive type: `lsblk -d -o name,rota` (1 = HDD, 0 = SSD)

### `unmounteddrvs`
Drives to temporarily mount for TRIM. Use cases:
- Hot-swap spare drives (installed but unmounted)
- Rescue/recovery partitions
- Secondary internal SSDs not auto-mounted

**NOT for:** External drives that might be unplugged!

### `mountbase`
Temporary mount point for unmounted drive operations.
Auto-creates `/tmp/drvchkmount` if you don't change defaults.

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

---

## Advanced Usage

See [DOCUMENTATION.md](DOCUMENTATION.md) for:
- Technical details on alert system
- Customizing notification methods
- Adding Slack/Discord webhooks
- Multi-machine monitoring setup
- btrfs/ZFS integration ideas

---

## Contributing

Found a bug? Have a feature request? PRs welcome!

**Requested features:**
- Support for btrfs scrub
- Support for ZFS pool health
- Prometheus metrics export
- Web dashboard for multi-machine monitoring

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

**Aggressive alerting philosophy:** If a drive is dying, EVERYONE should know immediately.
