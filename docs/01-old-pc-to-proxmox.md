# 1. Turning an old PC into a server (Proxmox VE)

[← Back to the guide](../README.md) · Next: [2. Network first →](02-network-first.md)

## What we're building and why

A computer that's always on, sits in a corner, and runs a pile of small services for the house, each one walled off from the others so one breaking doesn't take the rest down. The software that does the walling-off is a **hypervisor**.

This guide uses **Proxmox VE** (Virtual Environment): free, built on Debian Linux, and it runs two kinds of boxes side by side:

- **LXC containers** ("Linux Containers"): light boxes that share the host's kernel. Start in seconds, use very little RAM. Almost everything in this guide runs in one.
- **Virtual machines (VMs)**: full pretend computers with their own kernel. Heavier, but they run anything. Home Assistant ([chapter 10](10-home-assistant.md)) is the one VM in this build.

> 🤖 **Murderbot:** You could also just pay for the subscriptions.
> **Me:** Where's the fun in that.
> **Murderbot:** I have a list of where the fun is. It's mostly shows.

## Before you start

**You need:** the old PC; a USB stick of 8 GB or more you don't mind erasing; a second computer to make the installer stick; a monitor and keyboard for the server during setup; a network cable from the server to your router.

**Check three things:**

1. **Virtualization in the CPU.** Almost every desktop chip from the last fifteen years has it (Intel: VT-x, AMD: AMD-V), but it's often switched **off** in the BIOS from the factory. You'll turn it on in Step 3.
2. **VT-d / AMD-Vi ("IOMMU").** Lets you hand a whole PCI card to a VM. Some older chips don't have it. **Nothing in this guide needs it**, including passing a USB stick to a VM in chapter 10. Don't let its absence stop you.
3. **Drives.** One for Proxmox (an SSD is ideal), at least one more for data. This build: a 500 GB SSD plus two 1 TB hard drives, kept separate on purpose: one for media, one for backups.

## The build

### Step 1: Download the installer

Proxmox VE ships as one **ISO** file (a disk image), about 1.4 GB, from the Proxmox website's download page. This build ran **Proxmox VE 9** (built on Debian 13, "trixie").

### Step 2: Write it to the USB stick

An ISO has to be written in a bootable layout; dragging it onto the stick doesn't work. Free tools:

- **Rufus** (Windows): pick the stick, pick the ISO, Start. If it asks "ISO image mode" or "DD image mode", pick **DD** (what Proxmox's own docs recommend).
- **balenaEtcher** (Windows, Mac, Linux): image → drive → Flash.

Everything on the stick gets erased.

### Step 3: BIOS, then boot from the stick

Plug in the stick, power on, tap the setup key (often `Del`, `F2`, `F10` or `F12`; the boot screen usually says which).

- Find the virtualization setting (often **Advanced → CPU Configuration**, called "Intel Virtualization Technology", "VT-x" or "SVM Mode") → **Enabled**.
- Save and exit, then open the one-time **boot menu** (often `F11` or `F12`) and pick the USB stick.

### Step 4: Run the installer

Four screens matter:

- **Target disk.** Pick the SSD. **This erases that disk completely.** Check the size so you don't pick a data drive. (Better: unplug the data drives first.)
- **Country, time zone, keyboard.** Get the time zone right now. A time-zone mix-up shows up again in chapter 9 and it's annoying.
- **Root password and email.** `root` is the all-powerful admin account. There's no "forgot password" link on a server in a closet. Put it in a password manager.
- **Network.** The installer fills in the address it got. Accept it for now, but **read [chapter 2](02-network-first.md) before you treat it as permanent.** The worst outage in this whole build came from this screen.

Remove the stick when it says to reboot.

### Step 5: First login

The server's screen now shows:

```
https://<server-IP>:8006/
```

Open that in a browser on another computer on the same network. Port **8006** is the Proxmox web page. Your browser warns that the certificate isn't trusted; that's normal for a self-signed certificate on a home server. Click through. Log in as `root`, realm **Linux PAM**.

A **"No valid subscription"** popup appears. That's an ad for paid support. Nothing is disabled on the free version. Click OK. (It comes back every login.)

### Step 6: Fix the update sources (the "401 Unauthorized" error)

A fresh install points at the **enterprise** repository, which only serves paid subscribers. Your first update fails with `401 Unauthorized`. Nothing is broken.

Open a shell. I used the server's own keyboard; the web page has a **Shell** button, but the browser console can be flaky (see What went wrong).

```bash
ls /etc/apt/sources.list.d/
```

`ls` = list. That folder holds APT's download sources (APT is Debian's software installer).

Newer Proxmox uses `.sources` files, older uses `.list`. Disable the enterprise one:

```bash
# If it's pve-enterprise.sources:
rm /etc/apt/sources.list.d/pve-enterprise.sources

# If it's pve-enterprise.list:
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list
```

`rm` = remove. `sed -i 's/^deb/#deb/'` edits the file in place: every line starting with `deb` gets a `#` in front, which turns it into a comment.

> ⚠️ **Landmine: the second 401.** There may also be a **Ceph** file (`ceph.list` or `ceph.sources`) throwing the same `401`. Ceph is clustered storage for multi-server setups; one box doesn't need it. Remove or comment it out the same way.

Add the free repository. `<codename>` is `trixie` for Proxmox 9, `bookworm` for Proxmox 8:

```bash
echo "deb http://download.proxmox.com/debian/pve <codename> pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
apt update && apt full-upgrade -y
```

`>` writes the text into a file. `&&` runs the second command only if the first worked. `full-upgrade -y` installs every update and answers "yes". If a new kernel came in, `reboot`.

There's a copy of this in [`configs/pve-no-subscription.list.example`](../configs/pve-no-subscription.list.example).

### Step 7: Prepare the data drives

Two drives, two clearly named folders: `/mnt/media` for the library, `/mnt/backup` for backups.

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
```

`lsblk` lists drives and partitions. Your data drives show up as something like `sdb` and `sdc`.

> ⚠️ **Landmine: formatting erases everything.** Only format a drive that's empty or backed up. If it already has a Linux filesystem with files you want, skip to mounting.

```bash
mkfs.ext4 /dev/sdb1               # format partition sdb1 as ext4 (ERASES it)
mkdir -p /mnt/media /mnt/backup   # make the mount folders
blkid /dev/sdb1 /dev/sdc1         # print each partition's UUID
```

Use the **UUID** (a permanent ID) instead of `sdb1`, because drive letters can shuffle between boots. Add one line per drive to `/etc/fstab` (the list Linux reads at boot):

```bash
nano /etc/fstab
```

```
UUID=<uuid-of-media-drive>   /mnt/media   ext4   defaults,nofail   0 2
UUID=<uuid-of-backup-drive>  /mnt/backup  ext4   defaults,nofail   0 2
```

`nofail` = keep booting even if the drive is missing (a loose cable shouldn't brick the server). In `nano`: `Ctrl+O`, `Enter` to save, `Ctrl+X` to exit. Test without rebooting:

```bash
mount -a
df -h /mnt/media /mnt/backup
```

Both drives should show their full size. Copy: [`configs/fstab-local-drives.example`](../configs/fstab-local-drives.example).

### Step 8: Tell Proxmox about the backup drive, schedule backups

Web page: **Datacenter → Storage → Add → Directory.** ID: any name. Directory: `/mnt/backup`. Content: **VZDump backup file** (`vzdump` is Proxmox's backup tool).

Then **Datacenter → Backup → Add**: storage = the backup drive, schedule = nightly at a quiet hour, selection = **All**, mode = **Snapshot**.

> ⚠️ **Landmine: "All" doesn't mean all.** This job looks complete. It isn't. It never includes folders you bind-mount into containers later, and with no retention setting it keeps every nightly backup forever. I didn't find either problem until [chapter 9](09-immich-and-the-backup-gap.md), and I got the first fix wrong. Read it.

## What went wrong

- **`401 Unauthorized` on the first update.** The enterprise repository, plus the Ceph file. Step 6.
- **The browser console wouldn't connect.** "Failed to connect to server", especially from Safari on a Mac. A real monitor and keyboard is more reliable during setup; after [chapter 5](05-tailscale.md) you'll mostly use SSH.
- **"Command not found" right after installing something.** Run `hash -r`. Still missing? `find / -name <program> -type f 2>/dev/null` and run it by its full path.
- **The picture was cut off on the TV I used as a monitor.** That's overscan. In the TV's picture menu: "Just Scan", "Screen Fit" or "Overscan: Off".
- **A password with `!` broke a command** ("event not found"). Bash treats `!` as special. Wrap passwords in single quotes: `'MyPass123!'`.

> 💡 **A habit worth starting now.** Every community install script in this guide eventually offers to upgrade the whole host in the middle of building something ("Run host upgrade now"). I always chose **Ignore**. A host upgrade, and maybe a reboot, is a job for a quiet evening, not a side effect.

## Roads not taken (short version)

- **VMware ESXi:** a free edition came back in 2025, but registration, VMs only, picky about hardware.
- **TrueNAS / Unraid:** great if storage is the main job. Unraid costs money; this budget said $0.
- **Plain Debian + Docker:** lighter and totally valid. You lose per-service snapshots and an easy full VM, and this build needed one for Home Assistant.
- **The community "post-install" script:** does the repository fix and kills the subscription popup for you. I did it by hand to understand it. Fine to use.
- **ZFS on the boot drive:** checksums and snapshots, more RAM. One SSD and 16 GB to share: I kept the default.

## Skip ahead?

Required if you're starting from scratch. Already running Proxmox? Read Step 8's warning, then go straight to [chapter 2](02-network-first.md). It's the one I wish I'd read first.
