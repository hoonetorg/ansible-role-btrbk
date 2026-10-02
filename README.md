# ansible-role-btrbk

Installs [btrbk](https://digint.ch/btrbk/), renders `/etc/btrbk/btrbk.conf` from inventory data
and creates own systemd units (`btrbk-<name>.service`, optionally `btrbk-<name>.timer`).
The `btrbk.timer` shipped by the distribution package is masked (it runs `btrbk run` daily;
on Debian it is even enabled and started on install).

The role does **not** create filesystems, subvolumes, mounts, crypttab or fstab entries.
Volume paths and snapshot dirs must exist (use the disk role); the role asserts that they do.

On openSUSE Leap btrbk is not in the main repos; add the OBS `filesystems` repo first
(e.g. with ansible-role-repo).

## Variables

Everything lives in one dict `btrbk`, merged recursively over `btrbk_defaults`
(`defaults/main.yml`), so only differences need to be set.

| key                  | default                    | description |
|----------------------|----------------------------|-------------|
| `package`            | `btrbk`                    | package name |
| `conf_path`          | `/etc/btrbk/btrbk.conf`    | config file, also passed via `-c` to the units |
| `mask_shipped_timer` | `true`                     | mask the distribution's `btrbk.timer` |
| `config`             | see defaults               | global options (`key: value`) |
| `volumes`            | `[]`                       | volume sections |
| `units`              | `[]`                       | systemd units |
| `force_prune_units`  | `true`                     | create `btrbk-force-prune-<group>` units, see below |

Options (`config`, `volumes[].options`, `subvolumes[].options`, `targets[].options`) are
written as `key  value` lines. Booleans become `yes`/`no`, lists produce one line per element
(e.g. `group: [disk-a, offsite]`). For nested subvolumes (`home/projects`) without explicit
`snapshot_name` the role sets `snapshot_name home_projects` (btrbk's own default would be the basename
`projects`, which collides with other `.../projects` subvolumes in the same snapshot dir/target).
Validated with `btrbk -n -c <file> config print` before writing.

Default global options:

```
transaction_log /var/log/btrbk.log    snapshot_preserve_min latest
lockfile /run/btrbk.lock              snapshot_preserve 36h 14d 8w 24m 2y
timestamp_format long-iso             target_preserve_min all
snapshot_dir .snapshots               target_preserve no
snapshot_create always
incremental yes
```

### volumes

```yaml
volumes:
  - path: /mnt/btr_pool            # btrfs mount (usually top level, subvolid=5)
    options: {}                    # volume options
    targets: []                    # targets for all subvolumes of this volume
    subvolumes:
      - name: home                 # relative to the volume path
        options: { snapshot_preserve: 48h 7d }
        targets:
          - /mnt/backup-a/myhost   # string: "target <path>"
          - path: /mnt/backup-b/myhost   # dict: "target [type] <path>" + options
            type: send-receive
            options: { target_preserve: 4w }
```

### Occasionally connected backup disks

Give every target `noauto` and a group per disk, and let each backup unit filter by that group:

```yaml
volumes:
  - path: /mnt/btr_pool
    subvolumes:
      - name: home
        targets:
          - path: "/mnt/backup-a/{{ inventory_hostname_short }}"
            options: { noauto: true, group: disk-a }
      - name: data
        targets:
          - path: "/mnt/backup-b/{{ inventory_hostname_short }}"
            options: { noauto: true, group: disk-b }
units:
  - { name: backup-disk-a, command: resume, options: ["-v"], filters: [disk-a], requires_mounts_for: [/mnt/backup-a] }
  - { name: backup-disk-b, command: resume, options: ["-v"], filters: [disk-b], requires_mounts_for: [/mnt/backup-b] }
```

- runs without filter (the hourly `snapshot`) skip `noauto` targets and exit 0; snapshot cleanup of those
  subvolumes is skipped (warning) until their backup unit runs
- a filter matching the target group enables exactly those targets
- keep one target per subvolume: with several targets, snapshots are only pruned by a run that includes
  all of them

### Force-prune units

btrbk never deletes snapshots of a subvolume while one of its targets is skipped or unreachable (it cannot
know the latest snapshot/backup pair). For a disk that is lost, retired or not attached for a long time the
snapshots of its subvolumes therefore grow forever.

With `force_prune_units: true` (default) the role creates for every target group (e.g. `disk-b`):
- `/etc/btrbk/btrbk-force-prune-disk-b.conf`: btrbk.conf without the targets of group `disk-b`
- `btrbk-force-prune-disk-b.service`: `btrbk -c <that conf> -v prune`, no timer, no `[Install]`
  (listed as `static`, only runs with `systemctl start btrbk-force-prune-disk-b`)

Subvolumes whose only target is in that group are pruned by `snapshot_preserve*` (latest always kept); all
other subvolumes keep their `noauto` targets and are left untouched. The retention is the same as after a
successful backup, only the protection of the latest snapshot shared with the disk is missing. The next backup
to that disk is therefore incremental from the newest retained snapshot that was sent before (a backup sends all
missing snapshots, so retained dailies/weeklies/monthlies older than the last backup are on the disk): a larger
delta (up to the previous weekly/monthly/yearly), a full send only if no common snapshot is left.

Use it for lost, retired or never attached disks. For a disk that can be connected run its backup unit instead
(it sends and then prunes with the latest common snapshot protected).

Units/configs of groups that no longer exist are removed; `force_prune_units: false` removes all of them.

### units

Each entry creates `/etc/systemd/system/btrbk-<name>.service` (`Type=oneshot`) running
`btrbk -c <conf_path> <options...> <command> <filters...>`. With `timer` set, a
`btrbk-<name>.timer` is created, enabled and started. Units without timer are started by hand.

| key                   | description |
|-----------------------|-------------|
| `name`                | unit name suffix |
| `description`         | unit description (default `btrbk <command>`) |
| `command`             | btrbk command: `snapshot`, `resume`, `run`, `prune`, ... |
| `options`             | btrbk CLI options, e.g. `["-v"]`, `["-q"]` |
| `filters`             | btrbk filters: group names or paths, e.g. `[disk-a]` |
| `requires_mounts_for` | paths -> `RequiresMountsFor=` |
| `requires`            | units -> `Requires=` + `After=` (e.g. `mnt-backup\x2da.mount`) |
| `timer`               | `{on_calendar: hourly, accuracy_sec: 1m, persistent: true, enabled: true}` |
| `state`               | `present` (default) or `absent` (stops, disables, removes unit files) |

```yaml
units:
  - name: snapshot
    description: btrbk hourly snapshots
    command: snapshot
    options: ["-q"]
    timer: { on_calendar: hourly }
  - name: backup-disk-a
    description: btrbk backup to backup disk A
    command: resume
    options: ["-v"]
    filters: [disk-a]
    requires_mounts_for: [/mnt/backup-a]
```

## Backup disk via systemd (no script)

With crypttab (`noauto,nofail`) and an fstab line for the backup disk that has
`noauto,nofail,x-systemd.requires=systemd-cryptsetup@<name>.service`, a
`systemctl start btrbk-backup-disk-a` pulls in the mount through `RequiresMountsFor=`, which pulls
in the cryptsetup unit (asks for the passphrase), mounts, and then runs btrbk.
`systemctl stop systemd-cryptsetup@<name>.service` afterwards unmounts and closes the LUKS device.
The target directory (e.g. `/mnt/backup-a/myhost`) must exist on the backup filesystem.

## Notes

- `btrbk snapshot` does **not delete** snapshots of subvolumes whose target is not accessible
  (btrbk(1): the latest snapshot/backup pair must be determined first). With a backup disk that
  is only occasionally attached, snapshots of those subvolumes accumulate until the next
  `resume` run prunes them.
- All units share the lockfile (non-blocking): a snapshot timer firing during a long backup run
  exits with an error for that hour.

## License

Apache-2.0

Created with the help of AI
