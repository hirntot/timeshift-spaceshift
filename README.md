# Timeshift SpaceShift

A **cron**-driven helper for [Timeshift](https://github.com/linuxmint/timeshift) on **Btrfs**. When snapshot storage drives filesystem use past configurable thresholds, it deletes the **oldest** snapshots (down to a **minimum** count) and can **notify** the desktop user. If no graphical session is active, it only logs to a file.

**UI language** (log lines and `notify-send` text) follows `LANG` (first component: **de**, **fr**, **es**); anything else uses **English**. Under **cron**, `LANG` is often unset — the script then reads `/etc/default/locale` when present so it matches the distro default. Override explicitly in `/etc/cron.d/timeshift-spaceshift` (e.g. `LANG=de_DE.UTF-8` before the job line) if needed. To add another locale, extend the `l10n()` function in `source_files/usr/sbin/timeshift-spaceshift`.

---

## Requirements

- Debian/Ubuntu (or similar) with **Timeshift** and **Btrfs** on the relevant volumes
- Runs as **root** (the package uses **cron**)
- For **desktop notifications:** a logged-in user with an active X11/DBus session; otherwise only log output

The Debian package pulls in dependencies such as `timeshift`, `btrfs-progs`, `logrotate`, `sudo`, and `libnotify-bin`.

---

## Installation

### From a GitHub release (`.deb`)

1. Download the matching `*_all.deb` from the project’s **Releases** page.
2. Install:

```bash
sudo apt install ./timeshift-spaceshift_*_all.deb
```

Adjust the path and filename as needed.

### Build from source

From the `packages/timeshift-spaceshift` directory:

```bash
sudo apt install build-essential debhelper-compat
dpkg-buildpackage -us -uc -b
```

The built package appears one level up, for example:

```text
../timeshift-spaceshift_<version>_all.deb
```

In many parent repositories, `*.deb` is listed in `.gitignore`; ship the binary via **GitHub Releases**, not necessarily inside the git tree.

---

## Configuration

File: `/etc/timeshift/timeshift-spaceshift.conf`

| Variable | Meaning | Default |
|----------|---------|---------|
| `DISK_USAGE_WARNING_LIMIT` | Above this usage (%): warn only (notification + log) | `90` |
| `DISK_USAGE_LIMIT` | Above this: delete oldest snapshots until below limit or minimum count reached | `95` |
| `SNAPSHOT_LIMIT` | Minimum number of snapshots to keep | `3` |

Changes apply on the next cron run.

---

## Deletion Policy (Important)

- `timeshift-spaceshift` focuses on freeing disk space when usage limits are exceeded.
- It does **not** treat snapshot comments/descriptions as a protection flag.
- In other words: even commented snapshots can be removed automatically when cleanup is triggered.
- If you need to keep specific snapshots permanently, store/export them outside the normal cleanup path.

### How to keep a snapshot permanently

Keep snapshots outside the Timeshift snapshot pool that `timeshift-spaceshift` cleans:

1. **Recommended: copy to `/backups` as a btrfs snapshot** (same host, outside Timeshift path):

```bash
sudo btrfs subvolume snapshot -r "/timeshift-btrfs/snapshots/<SNAPSHOT_NAME>" "/backups/timeshift-archive/<SNAPSHOT_NAME>"
```

2. **Alternative: export with `btrfs send/receive`** (especially for another btrfs filesystem):

```bash
sudo btrfs send "/timeshift-btrfs/snapshots/<SNAPSHOT_NAME>" | sudo btrfs receive "/mnt/backup-btrfs/timeshift-archive"
```

3. **Alternative: create a file-based archive** (works on any filesystem):

```bash
sudo tar -C "/timeshift-btrfs/snapshots" -czf "/mnt/backup-disk/<SNAPSHOT_NAME>.tar.gz" "<SNAPSHOT_NAME>"
```

As long as a snapshot remains in the active Timeshift storage path, it is eligible for automatic deletion when disk usage exceeds your configured limits.

---

## Schedule (cron)

The package installs `/etc/cron.d/timeshift-spaceshift`: runs **hourly** at **minute 5** (e.g. 08:05) as **root**, offset from the top of the hour to reduce overlap with Timeshift’s own hourly jobs.

---

## Logs

- Script log: `/var/log/timeshift-spaceshift.log`
- On first run the script may create `/etc/logrotate.d/timeshift-spaceshift` (rotation, `maxage`, size).

---

## Notes for maintainers (GitHub)

1. Keep **`Homepage:`** in `debian/control` aligned with the public repository URL.
2. For each release, attach the built `timeshift-spaceshift_<version>_all.deb`. Optionally also upload a source package (`dpkg-buildpackage -S` or `debuild -S`) for PPA-style workflows.
3. Keep `LICENSE` and copyright year(s) up to date.

---

## Layout

| Path | Contents |
|------|----------|
| `source_files/usr/sbin/timeshift-spaceshift` | Main script |
| `source_files/etc/timeshift/timeshift-spaceshift.conf` | Default config (packaged) |
| `source_files/etc/cron.d/timeshift-spaceshift` | Cron entry |
| `debian/` | Packaging (`control`, `changelog`, `rules`, …) |

---

## License

This package is licensed under the **MIT License**. See `LICENSE`.
