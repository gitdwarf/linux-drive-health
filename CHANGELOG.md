# Changelog

All notable changes to Linux Drive Health Monitor, newest first.

### Version 2.3.0
- **Fixed `--test` stopping cron:** test mode deliberately makes every drive look unhealthy to exercise the alerts, but the cron-stop step was not guarded, so a test run on a healthy machine stopped the cron service. Test mode now never stops cron, and the test email says so.
- **Fixed drives being skipped on large systems:** disk discovery matched only `sda` to `sdz` and NVMe controller and namespace numbers 0 to 9, and configured drives had the same naming limit. `sdaa` and beyond, and `nvme10n1` and beyond, were never checked. Drives listed in `unmounteddrives` may now be whole disks or partitions, SATA or NVMe.
- **Internal:** stopping cron (`fn_stop_cron`) and the disk-name pattern (`disk_re`) are each defined once and shared by the code that uses them.

### Version 2.2.0
- **Fixed silent hang when a temporary unmount fails:** `umount` was attempted once with its error discarded, then the script waited forever for the mount to go away. A transient "target is busy" (reproduced on cycle 32 of a mount, TRIM, unmount test loop) could leave one run blocked for days, along with every `cron.daily` job after it. It now retries `umount` until `umount_timeout` (120 seconds by default), then emails a high priority alert with the real `umount` error plus `fuser` and `lsof` output, and exits.
- **New alert status:** `drive-health-monitor-status:UNMOUNT_FAILURE` header, with the subject `<machineID> unmount failure`.
- **Internal:** alert email sending is now one function (`fn_send_alert`), shared by the end of the run and the unmount failure path. No behaviour change for existing alerts.

### Version 2.1
- **Fixed singleton check:** the single-instance check now uses a `flock` lock file instead of searching for the script name with `pgrep`. If the lock file cannot be created, the run is treated as a drive error and alerts.

### Version 2.0
- **XFS Unicode Validation:** Pre-maintenance scanning prevents data corruption
- **UUID/LABEL Support:** Flexible drive specification
- **Improved Mount Handling:** Better conflict detection and normalization
- **Enhanced Test Mode:** Clearer distinction from live alerts
- **Better Reporting:** Per-inode unicode issue reporting
- **Conditional Safety:** TRIM and defrag skip if unicode problems are detected

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
