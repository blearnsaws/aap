# Mount Data Disks (Ansible)

Ansible playbook that discovers 12 unformatted block devices, creates ext4 filesystems, mounts them at `/data/1` through `/data/12`, and optionally emails a summary.

## Overview

- **Discovers** block devices with no filesystem (`lsblk -p -P`, `FSTYPE=""`).
- **Excludes** system devices: `nvme0n1p1`, `/dev/sda`, `/dev/sda1`, etc.
- **Fails** if exactly 12 such devices are not found.
- **Creates** directories `/data/1` … `/data/12` (owner `root`, mode `0755`).
- **Formats** each device as ext4 and adds **fstab** entries: `UUID=... /data/N ext4 defaults,noatime 0 0`.
- **Mounts** with `mount -a`.
- **Sends** an email listing the mounted filesystems under `/data/`.

## Requirements

- Target hosts: Linux with `lsblk`, `blkid`, `mkfs.ext4`, `mount`.
- Ansible control node: Ansible 2.9+.
- For email: `community.general` collection (provides `community.general.mail`). If the playbook uses the built-in `mail` module name, ensure the correct collection is installed and that the module resolves (e.g. `community.general.mail`).

## Variables

| Variable     | Default  | Description                          |
|-------------|----------|--------------------------------------|
| `mount_base`| `/data`  | Base path for mount points           |
| `num_disks` | `12`     | Required number of data disks        |
| `fs_type`   | `ext4`   | Filesystem type to create            |
| `email_dl`  | `admin@example.com` | Email recipient for mount summary |

Override via inventory, group_vars, or `-e`:

```bash
-e mount_base=/mnt/data -e num_disks=12 -e email_dl=ops@example.com
```

## Usage

**Limit to a single host:**
```bash
ansible-playbook mount_data_disks.yml -l dappa01.ghar.com
```

**Override variables:**
```bash
ansible-playbook mount_data_disks.yml -l myhost -e email_dl=team@example.com
```

**Check mode (no changes):**
```bash
ansible-playbook mount_data_disks.yml -l myhost --check
```

## Disk requirements

- Exactly **12** block devices must appear as having **no filesystem** (e.g. after `pvremove` or `wipefs -a` on former LVM/FS devices).
- Devices matching `nvme0n1p1`, `/dev/sda`, or `/dev/sda*` are excluded from the count.
- If you see **LVM2_member** or any FSTYPE on disks you want to use, clear them first, e.g.:
  ```bash
  pvremove -f /dev/sdb /dev/sdc ...
  # or
  wipefs -a /dev/sdb /dev/sdc ...
  ```

## Email

The playbook sends one email per host with the subject **"data disks mounted on &lt;hostname&gt;"** and a body listing the mounted lines under `mount_base`. Set `email_dl` to a valid address (or distribution list). If the `mail` module is not available or SMTP is not configured, that task may fail; install the required collection and configure SMTP as needed for your environment.

## Idempotency

- Creating dirs, adding fstab lines, and running `mount -a` are idempotent.
- **Creating the filesystem is not:** `filesystem` will not re-format an existing ext4 device, but if you change `num_disks` or device order, ensure your inventory and disk layout match to avoid wrong device-to-mount mapping.
- Run once per host when the 12 data disks are clean and ready; re-runs are safe for mounts and fstab.

## File

- **Playbook:** `mount_data_disks.yml`
