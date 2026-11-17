# Expanding & Combining Storage on an AlmaLinux VM (LVM + XFS)

A step-by-step record of how we grew the root filesystem from **16G → 56G**
by adding a second disk and merging it into the existing LVM volume group.

---

## 1. Before / After

### Before

| Mount | Size | Used | Free | Notes |
|---|---|---|---|---|
| `/` (root) | 16G | 1.4G | 15G | lived on `/dev/sda3` only |
| `/boot` | 2G | 364M | 1.6G | |
| `/boot/efi` | 599M | 9.1M | 590M | |
| `sdb` (40G) | — | — | — | **attached but unused** |

Disk layout before:

```
sda (20G)
├─sda1  600M  /boot/efi
├─sda2    2G  /boot
└─sda3 17.4G
  ├─almalinux-root 15.4G  /
  └─almalinux-swap    2G  [SWAP]

sdb (40G)      <-- doing nothing
```

### After

| Mount | Size | Used | Free | Notes |
|---|---|---|---|---|
| `/` (root) | **56G** | 2.2G | **54G** | now spans **two** disks |

Disk layout after:

```
sda (20G)
├─sda1  600M  /boot/efi
├─sda2    2G  /boot
└─sda3 17.4G ─┐
              ├─ VG "almalinux"
sdb (40G) ────┘   ├─almalinux-root  ~55G  /   (one filesystem!)
                  └─almalinux-swap    2G  [SWAP]
```

`df -h /` result:

```
Filesystem                  Size  Used Avail Use% Mounted on
/dev/mapper/almalinux-root   56G  2.2G   54G   4% /
```

---

## 2. Key Concepts (read this first)

You need four ideas to understand everything below.

### Disk → Partition → Filesystem → Mount

| Term | Meaning | Example |
|---|---|---|
| **Disk** | The raw physical/virtual device | `/dev/sda`, `/dev/sdb` |
| **Partition** | A slice of a disk | `/dev/sdb1` |
| **Filesystem** | The format that stores files (XFS, ext4) | made with `mkfs.xfs` |
| **Mount point** | Where a filesystem appears in the tree | `/`, `/mnt/storage` |

`df` only shows **mounted filesystems**. A disk that is attached but not
formatted/mounted will **not** appear in `df`. That is why the new 40G disk was
invisible at first.

### LVM (Logical Volume Manager)

LVM lets you pool multiple disks into one flexible "virtual disk". Three layers:

```
  Physical disks          Pool              Virtual disks (what you format/mount)
  ──────────────          ────              ─────────────────────────────────────
  /dev/sda3  ─┐
              ├──►  VG "almalinux"  ──►  LV "root"  ──►  XFS  ──►  /
  /dev/sdb   ─┘                          LV "swap"  ──►  swap
   (PV)                (VG)                    (LV)
```

| Layer | Name | Command to create |
|---|---|---|
| **PV** — Physical Volume | a disk/partition marked for LVM | `pvcreate` |
| **VG** — Volume Group | the pool made of one or more PVs | `vgcreate` / `vgextend` |
| **LV** — Logical Volume | a "virtual partition" carved from the VG | `lvcreate` / `lvextend` |

**The whole trick:** add the new disk as a **PV** → put it in the **same VG** →
**extend the existing LV** → **grow the filesystem** to fill it. Result: one
filesystem, bigger, no new mount points.

### XFS can grow while mounted

Our root filesystem is **XFS**. XFS can be enlarged **online** (while mounted)
with `xfs_growfs` — no reboot, no unmount. (ext4 uses `resize2fs` instead.)

### The commands you'll meet

| Command | What it answers |
|---|---|
| `df -h` | How full is each **mounted filesystem**? |
| `lsblk` | What **disks/partitions** exist and where are they mounted? |
| `blkid` | What **filesystem/UUID** is on a device? (empty = raw) |
| `free -h` | **RAM** usage — NOT disk! (common beginner mix-up) |
| `pvs` / `vgs` / `lvs` | The three LVM layers, one command each |

---

## 3. ⚠️ Warnings — read before running anything

- **`wipefs`, `mkfs`, `pvcreate` destroy data.** Only run them on a disk you
  are sure is empty or backed up.
- Combining two disks into one `/` means **if either disk fails, the whole
  root filesystem is damaged**. Fine for a lab VM; for production use RAID.
- LVM extension is **not easily reversible** — you can shrink an LV but XFS
  **cannot be shrunk**. Treat the merge as one-way.
- Always **verify with `lsblk` / `blkid`** before the destructive steps.

---

## 4. Step-by-Step

### Step 0 — Inspect the current state

```bash
df -h          # what is mounted and how full
lsblk          # what disks/partitions exist
```

Goal: identify the new disk. In our case it was **`/dev/sdb`, 40G, no
partitions, no mount point** — i.e. attached but unused.

### Step 1 — Confirm the new disk is raw and empty

```bash
sudo blkid /dev/sdb
lsblk /dev/sdb
```

- `blkid` prints **nothing** → no filesystem, safe to use.
- `lsblk` shows **no children** (no `sdb1`) → not partitioned.

> If `lsblk` shows an `sdb1` child, see **Troubleshooting §7** — that is the
> exact error we hit.

### Step 2 — If the disk had a partition, remove it first

*(Skip if Step 1 showed a raw, unpartitioned disk.)*

```bash
sudo umount /mnt/storage                 # unmount if mounted
sudo sed -i '/\/mnt\/storage/d' /etc/fstab   # remove any fstab entry
sudo wipefs -a /dev/sdb1                 # erase filesystem signature
sudo wipefs -a /dev/sdb                  # erase partition table signature
```

Verify it is now raw — `blkid` must print nothing, `lsblk` must show no children:

```bash
sudo blkid /dev/sdb
lsblk /dev/sdb
```

### Step 3 — Mark the whole disk as an LVM Physical Volume (PV)

```bash
sudo pvcreate /dev/sdb
```

`pvcreate` writes LVM metadata onto the raw disk. It does **not** create a
filesystem — it just says "LVM may use this disk".

### Step 4 — Add the new PV to the existing Volume Group (VG)

```bash
sudo vgs                       # confirm the VG name is "almalinux"
sudo vgextend almalinux /dev/sdb
```

`vgextend` grows the pool. After this the VG is ~57G (17.4G from `sda3` +
40G from `sdb`). No filesystem has changed size yet.

### Step 5 — Extend the Logical Volume (LV) to use all the new space

```bash
sudo lvextend -l +100%FREE /dev/mapper/almalinux-root
```

- `-l +100%FREE` = "give the LV **all** free extents currently in the VG".
- `+` means *add to* the current size (not set an absolute size).
- `/dev/mapper/almalinux-root` is the LV that backs `/`.

At this point the **LV** is bigger, but the **filesystem** inside it is still
the old size. That is what Step 6 fixes.

### Step 6 — Grow the XFS filesystem to fill the LV

```bash
sudo xfs_growfs /
```

Because XFS can grow online, this works while `/` is mounted. Output will show
something like:

```
data blocks changed from 4039680 to 14524416
```

That single line is proof the filesystem expanded.

> On **ext4** the equivalent command is `sudo resize2fs /dev/mapper/almalinux-root`.

### Step 7 — Verify everything

```bash
df -h /        # / should now be ~56G
sudo pvs       # both /dev/sda3 and /dev/sdb listed as PVs
sudo vgs       # VG size ~57G, VFree 0 (all allocated)
sudo lvs       # LV "root" now ~55G
```

Expected:

```
$ df -h /
Filesystem                  Size  Used Avail Use% Mounted on
/dev/mapper/almalinux-root   56G  2.2G   54G   4% /

$ sudo vgs
  VG        #PV #LV #SN Attr   VSize   VFree
  almalinux   2   2   0 wz--n- <57.41g    0

$ sudo pvs
  PV         VG        Fmt  Attr PSize   PFree
  /dev/sda3  almalinux lvm2 a--   17.41g    0
  /dev/sdb   almalinux lvm2 a--  <40.00g    0
```

**Done.** There is now a single `/` filesystem spanning both disks, and
`/var/lib/docker` (where Docker stores images/containers) automatically uses it.

### Optional cleanup

```bash
sudo rmdir /mnt/storage    # empty leftover dir, safe to remove
```

---

## 5. Command Cheat Sheet

| Command | Layer | Purpose |
|---|---|---|
| `df -h` | filesystem | free space per **mounted** filesystem |
| `lsblk` | disk | tree of disks → partitions → mount points |
| `blkid <dev>` | filesystem | filesystem type + UUID (empty = raw) |
| `wipefs -a <dev>` | disk | erase filesystem/partition signatures |
| `pvcreate <dev>` | LVM PV | mark a raw disk as an LVM physical volume |
| `vgextend <vg> <pv>` | LVM VG | add a PV to an existing volume group |
| `lvextend -l +100%FREE <lv>` | LVM LV | grow a logical volume by all free space |
| `xfs_growfs <mountpoint>` | XFS | grow XFS filesystem to fill its LV |
| `resize2fs <lv>` | ext4 | grow ext4 filesystem to fill its LV |
| `pvs` / `vgs` / `lvs` | LVM | show the three LVM layers |

### The one-line mental model

```
pvcreate  →  vgextend  →  lvextend  →  xfs_growfs
(disk)       (pool)        (volume)     (filesystem)
```

Each step feeds the next:
`pvcreate` makes a disk usable by LVM →
`vgextend` adds that disk to the pool →
`lvextend` gives the extra pool space to the LV →
`xfs_growfs` stretches the filesystem to fill the bigger LV.

Skip any step and the next one has nothing to work with.

---

## 6. Alternative: keep the new disk as a *separate* mount (not combined)

If you do **not** want to merge, you can format the disk and mount it at a path
instead. This is what we did first, before deciding to combine:

```bash
sudo parted /dev/sdb --script mklabel gpt
sudo parted /dev/sdb --script mkpart primary xfs 0% 100%
sudo mkfs.xfs /dev/sdb1

sudo mkdir -p /mnt/storage
sudo mount /dev/sdb1 /mnt/storage

# persist across reboots — add to /etc/fstab:
sudo blkid /dev/sdb1     # copy the UUID
# UUID=xxxx-...  /mnt/storage  xfs  defaults  0 0
sudo mount -a            # test fstab without rebooting
```

Result: `/` stays 16G, and `/mnt/storage` is a separate 40G filesystem.
**Two units, not one.** LVM merging (the main guide) is what gives you a single
combined filesystem.

> You can also use an existing partition (`/dev/sdb1`) as the LVM PV with
> `pvcreate /dev/sdb1` instead of wiping the whole disk. It works the same —
> the only reason we wiped was to give LVM the entire raw disk cleanly.

---

## 7. Troubleshooting

### `Cannot use /dev/sdb: device is partitioned`

**Cause:** the disk still has a partition table (`sdb1`). LVM refused it.

**Diagnose:**

```bash
lsblk /dev/sdb
sudo blkid /dev/sdb /dev/sdb1
```

**Fix** (destroys the partition and its filesystem):

```bash
sudo umount /mnt/storage
sudo sed -i '/\/mnt\/storage/d' /etc/fstab
sudo wipefs -a /dev/sdb1
sudo wipefs -a /dev/sdb
lsblk /dev/sdb          # confirm no sdb1 child remains
sudo pvcreate /dev/sdb
sudo vgextend almalinux /dev/sdb
```

### `df` doesn't show the new disk at all

`df` only lists **mounted** filesystems. Use `lsblk` to see attached disks.
A raw/unmounted disk is invisible to `df` — that is normal, not a bug.

### `df free` does nothing useful

`free` is a **separate command for RAM**, not disk. Use `df -h` for disk and
`free -h` for memory.

### `xfs_growfs: ... not a mounted XFS filesystem`

Make sure the argument is the **mount point** (`/`), not the device path.

---

## 8. Quick recap

1. New disk `sdb` was attached but invisible to `df` (not formatted/mounted).
2. It had an old partition + XFS from an earlier attempt → `vgextend` failed.
3. We unmounted, removed the fstab line, and wiped it with `wipefs`.
4. `pvcreate` → `vgextend` → `lvextend` → `xfs_growfs`.
5. `/` grew 16G → 56G, one filesystem, spanning two disks.
6. Verified with `df -h /`, `pvs`, `vgs`, `lvs`.
