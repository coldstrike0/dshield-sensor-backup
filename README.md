# Full Backup & Restore Procedure

Hash-verified backup of all attacker-generated data on a DShield/Cowrie sensor before a reboot or reinstallation.

Preserves: web logs (`/srv/log/`), Cowrie logs (`/srv/cowrie/var/log/cowrie/`), TTY sessions (`/srv/cowrie/var/lib/cowrie/tty/`), and attacker-uploaded files (`/srv/cowrie/var/lib/cowrie/downloads/`).

## Step 1 — Identify and mount the USB drive

```bash
lsblk
# connect the USB drive, run again — the new device (e.g. sda1) is your drive
lsblk

sudo mkdir -p /mnt/usb
sudo mount /dev/sda1 /mnt/usb
```

## Step 2 — Hash the live source data (relative paths, from /)

```bash
cd /
sudo find srv/log srv/cowrie/var/log/cowrie \
     srv/cowrie/var/lib/cowrie/tty srv/cowrie/var/lib/cowrie/downloads \
     -type f -exec sha256sum {} \; | sudo tee /mnt/usb/source_hashes.sha256
```

## Step 3 — Create the archive

```bash
sudo tar -czvf /mnt/usb/sensor_backup_$(date +%F).tar.gz \
     /srv/log /srv/cowrie/var/log/cowrie \
     /srv/cowrie/var/lib/cowrie/tty /srv/cowrie/var/lib/cowrie/downloads
```

## Step 4 — Hash the archive

```bash
sha256sum /mnt/usb/sensor_backup_$(date +%F).tar.gz \
     | tee /mnt/usb/sensor_backup_$(date +%F).sha256
```

## Step 5 — Verify against live data (safety gate — do not pass without all OK)

```bash
sudo mkdir -p /tmp/verify_check
sudo tar -xzf /mnt/usb/sensor_backup_$(date +%F).tar.gz -C /tmp/verify_check
cd /tmp/verify_check
sudo sha256sum -c /mnt/usb/source_hashes.sha256
```

Every line must report `OK`. If anything fails, the live data still exists. Re-run from Step 2.

## Step 6 — Clean up and unmount (now safe to reboot/reinstall)

```bash
sudo rm -rf /tmp/verify_check
sudo umount /mnt/usb
```

*Reboot or reinstall the sensor.*

## Step 7 — After rebuild: verify storage integrity

```bash
sudo mount /dev/sda1 /mnt/usb
sha256sum -c /mnt/usb/sensor_backup_2026-06-27.sha256
```

(Use the actual dated filename — `$(date +%F)` would resolve to today's date, not the backup's.)

## Step 8 — Extract and verify restore integrity

```bash
sudo mkdir -p /tmp/restore_check
sudo tar -xzf /mnt/usb/sensor_backup_2026-06-27.tar.gz -C /tmp/restore_check
cd /tmp/restore_check
sudo sha256sum -c /mnt/usb/source_hashes.sha256
```

All `OK` = the restored data is byte-for-byte identical to what was live before the wipe. Move the verified directories back into place as needed.
