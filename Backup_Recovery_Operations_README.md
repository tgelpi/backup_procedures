# Backup & Recovery Operations README

**Environment:** MacBook Pro, Ubuntu `dell-pc`, Buffalo LS210D884, Buffalo LS210D42C  
**Last operational validation:** September 28, 2026

This document describes the current backup architecture, normal operating procedures, power-outage recovery, monitoring, and troubleshooting. It is intended for another administrator or support person who may need to maintain or recover the environment.

---

## 1. Architecture Overview

```text
                         BACKUP ENVIRONMENT

        MacBook Pro                              Ubuntu dell-pc
             |                                         |
        Time Machine                         systemd nightly timer
        /          \                                  |
       /            \                    backup-to-ls210d42c.service
      v              v                                |
LS210D884       Sabrent SSD                 rsync daemon protocol
TM_Backups      Mac Recovery                           |
Routine NAS     Detachable recovery                   v
                                                LS210D42C
                                              linux_backups
                                                   |
                                                dell-pc/
```

The two Buffalo NAS devices have distinct primary roles:

- **LS210D884** — primary routine Mac Time Machine destination.
- **LS210D42C** — Linux backup destination and other server/media duties.
- **Sabrent SSD / Mac Recovery** — independent, detachable Mac recovery backup.

---

## 2. MacBook Pro Backups

### 2.1 Routine Time Machine Backup

Primary routine Mac backup:

- NAS: `LS210D884`
- Time Machine share: `TM_Backups`
- Purpose: normal at-home Time Machine protection.

This existing Time Machine configuration is considered important and should not be casually modified while NAS protocol changes are being evaluated.

### 2.2 Sabrent Mac Recovery Backup

A Sabrent external SSD named **Mac Recovery** provides a second, detachable Mac backup.

Typical procedure:

1. Connect the Sabrent SSD to macbook.
2. Run on macbook:

```bash
backup-mac-recovery
```

3. Check Time Machine status on macbook:

```bash
tmutil status
```

4. When complete, safely eject the SSD.

The drive can be disconnected and retained separately or taken when traveling.

### 2.3 Pruning Mac Recovery Backups

The controlled pruning utility is on macbook under :

```bash
prune-mac-recovery 2
```

The numeric argument is the number of newest backups to preserve. The utility lists available backup timestamps, shows exactly which backups would be removed, and requires explicit `YES` confirmation before deletion.

Do not manually delete Time Machine backup structures unless there is a specific recovery reason.

---

## 3. Ubuntu `dell-pc` Backup

### 3.1 Schedule

Linux backups are scheduled every day at midnight using systemd:

```text
backup-to-ls210d42c.timer
        |
        v
backup-to-ls210d42c.service
        |
        v
/usr/local/bin/backup-to-ls210d42c
```

Check the timer:

```bash
systemctl list-timers --all | grep -i backup
```

Relevant units:

```text
backup-to-ls210d42c.timer
backup-to-ls210d42c.service
```

The timer should be `enabled`.

### 3.2 Backup Destination

Rsync destination:

```text
rsync://admin@ls210d42c.local/linux_backups/dell-pc
```

NAS filesystem location:

```text
/mnt/disk1/linux_backups/
```

Primary backup directory:

```text
/mnt/disk1/linux_backups/dell-pc/
```

### 3.3 Client Authentication

The rsync client password file on `dell-pc` is:

```text
/root/.config/rsync-ls210d42c.pass
```

Do **not** display, copy into documentation, or expose the contents of this file.

### 3.4 Backup Log

```text
/var/log/backup-to-ls210d42c.log
```

Check recent activity with:

```bash
sudo tail -50 /var/log/backup-to-ls210d42c.log
```

A successful run should end with:

```text
===== Backup completed: <date/time> =====
```

### 3.5 Data Included

The current backup script backs up:

```text
/home/tgelpi/
/home/tbear/
/etc/
/var/lib/plexmediaserver/
/opt/internet-monitor/
/var/log/internet-monitor/ 
```

The rsync options include:

```text
-a
--delete
--no-owner
--no-group
--stats
--password-file=/root/.config/rsync-ls210d42c.pass
```

For `/home/tgelpi/`, exclusions include:

```text
media/
Downloads/
.cache/
```

**Important:** `--delete` means files deleted from the source can also be deleted from the mirrored destination. Do not modify the rsync source/destination paths casually.

### 3.6 Manual Backup

To run the exact same backup path used by the timer:

```bash
sudo systemctl start backup-to-ls210d42c.service
```

Then verify:

```bash
systemctl status backup-to-ls210d42c.service --no-pager
sudo tail -20 /var/log/backup-to-ls210d42c.log
```

Because the service is `Type=oneshot`, `inactive (dead)` after successful completion is normal. Look for a successful exit and the `Backup completed` log entry.

---

## 4. LS210D42C Rsync Service

### 4.1 Important Buffalo Behavior

Rsync on LS210D42C is managed by Buffalo's **`inetd`**, not by a permanently running standalone rsync daemon.

Typical process:

```text
/usr/sbin/inetd
```

TCP port:

```text
873
```

Check:

```bash
netstat -an | grep 873
```

**DO NOT start a separate daemon with:**

```text
rsync --daemon
```

A standalone daemon will conflict with the existing inetd listener and produce an `Address already in use` error.

### 4.2 Active Rsync Configuration

Runtime configuration:

```text
/etc/rsyncd.conf
```

The required module is:

```text
linux_backups
```

The module points to:

```text
/mnt/disk1/linux_backups
```

### 4.3 Persistent Master Configuration

Buffalo may regenerate `/etc/rsyncd.conf` during boot. Therefore persistent master copies are stored on the data disk:

```text
/mnt/disk1/linux_backups/.rsync-config/rsyncd.conf
/mnt/disk1/linux_backups/.rsync-config/rsyncd.secrets
```

Both should be protected:

```text
owner: root
group: root
mode: 600
```

Check:

```bash
ls -l /mnt/disk1/linux_backups/.rsync-config/
```

Never display the contents of `rsyncd.secrets`.

### 4.4 Verify Rsync Module

From another system:

```bash
rsync rsync://ls210d42c.local/
```

Expected:

```text
linux_backups    Linux Backups
```

From `dell-pc`, authenticated listing:

```bash
sudo rsync \
  --password-file=/root/.config/rsync-ls210d42c.pass \
  rsync://admin@ls210d42c.local/linux_backups/
```

---

## 5. Buffalo Power-Outage Behavior

Testing on September 28, 2026 established that a cold boot following a power outage can reset/rebuild some Buffalo runtime/generated configuration.

Observed effects:

- NAS remains pingable.
- SSH is no longer available until SSH/SFTP support is restored and `sshd` is started.
- LS210D42C `/etc/rsyncd.conf` can revert to Buffalo's base configuration and lose the custom `[linux_backups]` module.
- Persistent NAS data survives.
- Existing account/root passwords survive.
- Persistent `.rsync-config` files under `/mnt/disk1` survive.

The likely sequence is:

```text
Power outage
     |
     v
Cold boot
     |
     v
Buffalo initialization
     |
     +--> SSH/SFTP runtime feature state reset
     |
     +--> /etc/rsyncd.conf regenerated
                |
                v
        linux_backups unavailable
```

This is a configuration recovery issue, not normally a loss of backup data.

---

## 6. Post-Power-Outage Recovery

A tested Mac-side utility is maintained:

```text
recover-buffalo-nas.sh
```

It uses:

```text
acp_commander.jar
```

ACP Commander version tested:

```text
ACP Commander v0.6 (2021)
```

Default NAS list:

```text
ls210d42c.local
ls210d884.local
```

### 6.1 Run Recovery

From the directory containing the script and `acp_commander.jar`:

```bash
./recover-buffalo-nas.sh
```

The script securely prompts for the Buffalo ACP password. An empty password aborts the operation.

### 6.2 Recovery Actions

For both NAS devices the script:

1. Pings the NAS.
2. Restores `SUPPORT_SSH=on`.
3. Restores `SUPPORT_SFTP=1`.
4. Starts `/etc/init.d/sshd.sh`.
5. Verifies TCP port 22.

For LS210D42C it additionally:

1. Verifies the persistent rsync configuration exists.
2. Verifies the persistent rsync secrets file exists.
3. Restores root ownership and mode `600` on the secrets file.
4. Copies the persistent `rsyncd.conf` contents into `/etc/rsyncd.conf`.
5. Sets active configuration ownership to `root:root` and mode `600`.
6. Verifies TCP port 873.
7. Verifies that `linux_backups` is advertised.

It does **not**:

- change NAS passwords;
- alter backup data;
- recreate the rsync password;
- start a standalone rsync daemon;
- start a Linux backup.

### 6.3 ACP Commander Limitation

ACP Commander v0.6 has an approximately **210-character remote command limit**.

Recovery operations are deliberately split into short commands. Do not combine them into a large compound command without checking this limitation.

### 6.4 Expected Recovery Result

```text
========================================
 Recovery Summary
========================================
Successful : 2
Failed     : 0
Skipped    : 0
========================================
```

This procedure was validated after an actual September 28, 2026 power outage. Both NAS SSH services were successfully restored, the LS210D42C `linux_backups` module was restored, and a subsequent Linux backup completed successfully.

---

## 7. Linux Backup Failure Monitoring

The nightly Linux backup has a systemd failure handler.

The backup service drop-in contains:

```ini
[Unit]
OnFailure=backup-to-ls210d42c-failure@%n
```

Verify:

```bash
systemctl show backup-to-ls210d42c.service -p OnFailure
```

Expected:

```text
OnFailure=backup-to-ls210d42c-failure@backup-to-ls210d42c.service
```

### 7.1 Failure Handler Unit

```text
/etc/systemd/system/backup-to-ls210d42c-failure@.service
```

It invokes:

```text
/usr/local/bin/backup-to-ls210d42c-failed
```

### 7.2 Failure Warning Files

A failed backup creates:

```text
/var/log/backup-to-ls210d42c.FAILURE
/etc/backup-warning
```

The user's `~/.bashrc` checks `/etc/backup-warning` and displays it when a shell is opened.

The warning directs support to check:

```bash
systemctl status backup-to-ls210d42c.service --no-pager
sudo tail -50 /var/log/backup-to-ls210d42c.log
```

### 7.3 Clearing Failure State

A successful run of `/usr/local/bin/backup-to-ls210d42c` removes:

```text
/var/log/backup-to-ls210d42c.FAILURE
/etc/backup-warning
```

Because the backup script uses `set -e`, a failed rsync exits before these warning files are removed.

---

## 8. Troubleshooting

### `@ERROR: Unknown module 'linux_backups'`

Check:

```bash
rsync rsync://ls210d42c.local/
```

If `linux_backups` is missing, `/etc/rsyncd.conf` has probably been regenerated/reset.

Run the tested recovery utility:

```bash
./recover-buffalo-nas.sh
```

Then verify the module again.

### `@ERROR: auth failed on module linux_backups`

On LS210D42C, inspect the rsync log messages:

```bash
tail -30 /var/log/messages | grep -i rsync
```

A previously observed cause was:

```text
secrets file must be owned by root when running as root
```

Required persistent secrets permissions:

```text
root:root
600
```

Do not expose the secrets file contents.

### `Address already in use` when starting rsync

Do not start a standalone rsync daemon. Buffalo `inetd` already owns TCP/873.

Check:

```bash
ps w | grep -Ei 'rsync|inetd|xinetd'
netstat -an | grep 873
```

### NAS ping works but SSH is refused

This is the known post-power-outage condition.

Run:

```bash
./recover-buffalo-nas.sh
```

Do not reset passwords merely because SSH is unavailable.

---

## 9. Routine Support Checklist

For a quick health check:

```bash
systemctl list-timers --all | grep -i backup
systemctl status backup-to-ls210d42c.service --no-pager
sudo tail -20 /var/log/backup-to-ls210d42c.log
```

Verify LS210D42C module availability:

```bash
rsync rsync://ls210d42c.local/
```

Expected:

```text
linux_backups    Linux Backups
```

Check for a persistent failure warning:

```bash
ls -l /etc/backup-warning 2>/dev/null
```

For the Mac, periodically confirm Time Machine is completing successfully and create/update the detachable Sabrent Mac Recovery backup as appropriate.

---

## 10. Operational Warnings

1. **Never expose rsync passwords or the contents of `rsyncd.secrets`.**
2. **Do not start `rsync --daemon` on LS210D42C.** Buffalo `inetd` manages TCP/873.
3. **Do not casually modify `/mnt/disk1/linux_backups/`.** It contains persistent backup data and recovery configuration.
4. The Linux backup uses **`--delete`**. Verify source and destination paths before modifying the backup script.
5. Do not assume an enabled systemd timer means backups are succeeding. Always inspect service status/logs or the failure warning mechanism.
6. After a Buffalo cold boot/power outage, ping success does **not** prove SSH or the custom rsync module survived.
7. Keep `recover-buffalo-nas.sh` and `acp_commander.jar` together in their established administration directory.
8. Treat the persistent `.rsync-config` directory as critical recovery infrastructure.

---

## 11. Key Files and Commands

| Purpose | Location / Command |
|---|---|
| Linux backup script | `/usr/local/bin/backup-to-ls210d42c` |
| Linux backup service | `backup-to-ls210d42c.service` |
| Linux backup timer | `backup-to-ls210d42c.timer` |
| Backup log | `/var/log/backup-to-ls210d42c.log` |
| Client password file | `/root/.config/rsync-ls210d42c.pass` |
| NAS backup root | `/mnt/disk1/linux_backups` |
| Persistent rsync config | `/mnt/disk1/linux_backups/.rsync-config/rsyncd.conf` |
| Persistent rsync secrets | `/mnt/disk1/linux_backups/.rsync-config/rsyncd.secrets` |
| Active rsync config | `/etc/rsyncd.conf` |
| Rsync module | `linux_backups` |
| Rsync TCP port | `873` |
| SSH TCP port | `22` |
| Failure handler | `/usr/local/bin/backup-to-ls210d42c-failed` |
| Failure unit | `/etc/systemd/system/backup-to-ls210d42c-failure@.service` |
| Failure record | `/var/log/backup-to-ls210d42c.FAILURE` |
| Login warning | `/etc/backup-warning` |
| NAS recovery script | `recover-buffalo-nas.sh` |
| Mac detachable backup | `backup-mac-recovery` |
| Mac recovery pruning | `prune-mac-recovery 2` |

---

## 12. Recovery Priority

If support is responding after a power event:

```text
1. Confirm both NAS units are powered and pingable.
2. Run recover-buffalo-nas.sh from the Mac.
3. Require Successful: 2 / Failed: 0 / Skipped: 0.
4. Verify SSH access if needed.
5. Verify `rsync rsync://ls210d42c.local/` shows linux_backups.
6. On dell-pc, check backup service/log status.
7. If desired, run a manual systemd backup.
8. Confirm the log ends with "Backup completed".
9. Confirm /etc/backup-warning is absent after success.
```

This sequence has been tested successfully following a real power outage.
