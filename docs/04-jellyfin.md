# 4. Your own streaming service (Jellyfin + a file share)

[← 3. Pi-hole](03-pihole.md) · [Guide](../README.md) · Next: [5. Tailscale →](05-tailscale.md)

## What we're building and why

**Jellyfin** is a free, open-source media server. Point it at folders of movies and shows; it fetches posters and descriptions, remembers where everyone left off, and streams to phones, TVs, sticks and browsers. No company account, no subscription.

This chapter also covers getting files onto the server (a Samba share), the biggest hardware decision in the guide (transcoding), and a disk that filled itself months later.

## Before you start

Chapters 1 and 2. The media drive at `/mnt/media`. Video you have the right to use: home videos, your own discs ripped to files, public-domain or Creative Commons films.

## The build

### Step 1: Create the container

```bash
bash -c "$(wget -qLO - https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/jellyfin.sh)"
brctl show vmbr0     # the two-second habit: is the new veth listed?
```

Basic (Default) settings.

### Step 2: Give it the media drive

A new container only sees its own small disk. A **bind mount** makes a host folder appear inside it:

```bash
pct set <jellyfin-ID> -mp0 /mnt/media,mp=/media
pct reboot <jellyfin-ID>
```

`-mp0` = mount point 0. Before the comma: the **host** path. `mp=/media`: where it shows up **inside** the container. Same files, two names, depending which side you stand on. That trips everyone once.

### Step 3: A file share, so you can drag files in

**Samba** speaks SMB, the file-sharing language Windows, Mac and Linux all understand. I installed it on the Proxmox host:

```bash
apt install samba -y
nano /etc/samba/smb.conf
```

At the bottom:

```ini
[media]
   path = /mnt/media
   browseable = yes
   read only = no
   guest ok = no
   valid users = root
```

```bash
smbpasswd -a root        # gives root a Samba password (separate from everything else)
systemctl restart smbd
```

Mac: Finder → Go → Connect to Server (`Cmd+K`) → `smb://<server-IP>`. Windows: `\\<server-IP>` in File Explorer. Copy: [`configs/smb.conf.example`](../configs/smb.conf.example).

> ⚠️ **Landmine: that's a third password.** Separate from the Proxmox root login and from Pi-hole's. Weeks later I tried the wrong two first. `smbpasswd -a root` again sets a new one.

> 🤖 **Murderbot:** A dedicated Samba user would be better practice than root.
> **Me:** Noted.
> **Murderbot:** You said "noted" in the voice that means "never".
> It's in Roads not taken. It's still root.

### Step 4: Jellyfin's setup wizard, with tight libraries

Open `http://<jellyfin-IP>:8096`, create the admin user, add libraries. Two rules, both learned the hard way:

1. **Use the container-side path.** Inside the container the drive is `/media`, not `/mnt/media`.
2. **Point each library at exactly the folder it owns.** Movies → `/media/movies`, Shows → `/media/tv`. **Not** the whole `/media`.

My first library was the whole `/media`. Fine, until other things moved onto that drive: then the download folders showed up in the browse screen (not great when you're showing the server off), and later Jellyfin threw permission errors on the private photo folder every scan. [Chapter 7](07-permissions.md) has the full story.

## The transcoding landmine: read this before building a big library

Video comes in different **codecs**. If a device can play a file's codec as-is, Jellyfin just sends it: **Direct Play**, almost free for the server. If not, Jellyfin **transcodes**: re-encodes it live while you watch.

Modern graphics chips have video hardware that makes that easy. This box's old card has none, so four CPU cores do it alone, and they can't keep up with high-quality video. **What it looks like:** plays a few seconds, then stutters and stalls. Jellyfin's Dashboard shows the session as **Transcoding**, often with a reason.

Options, cheapest first:

- **$0: keep the library in H.264.** Nearly every device can Direct Play H.264. Convert stragglers with **HandBrake** on your everyday computer, not the server. ("No Valid Source Found" in HandBrake: select the file, not the folder. Still failing? It's probably copy-protected. Watch it in its original app.)
- **About $130:** a cheap graphics card bought only for its video engine.
- **About $170–220:** a small modern mini PC whose built-in graphics handle it.

This box stayed on the $0 option. (Part 2 has some newer Intel chips with Quick Sync in it. Foreshadowing.)

## What went wrong: the disk that filled itself

Months in, Jellyfin started misbehaving, then mostly stopped. Its own 16 GB system disk: **100% full.** Nothing added on purpose.

> 🔍 **Diagnostic pattern: "a disk is full, but of what?"** Confirm it, then measure the biggest folders. Don't guess.

```bash
pct exec <jellyfin-ID> -- df -h /
pct exec <jellyfin-ID> -- bash -c "du -sh /var/* 2>/dev/null | sort -h"
```

> ⚠️ **Landmine: `pct exec` doesn't expand `*`.** It runs the program directly, not through a shell. Wrap anything with a wildcard in `bash -c "…"`.

Culprit: `/var/cache/jellyfin/transcodes`, **13 GB** of leftover pieces of live transcodes. The built-in cleanup task was running but evidently missing orphans, probably from sessions cut off during a stretch of restarts.

**Right now** (after checking nothing was playing):

```bash
pct exec <jellyfin-ID> -- bash -c "rm -rf /var/cache/jellyfin/transcodes/*"
```

Safe here (Jellyfin recreates these). Be very careful with `rm -rf` anywhere else.

**For good: move the transcode folder to the big drive.**

```bash
mkdir -p /mnt/backup/jellyfin-transcodes
setfacl -R -m u:<jellyfin-host-UID>:rwX /mnt/backup/jellyfin-transcodes
setfacl -R -d -m u:<jellyfin-host-UID>:rwX /mnt/backup/jellyfin-transcodes
pct set <jellyfin-ID> -mp1 /mnt/backup/jellyfin-transcodes,mp=/transcodes
pct reboot <jellyfin-ID>
```

`<jellyfin-host-UID>` is a number you'll learn to find in [chapter 7](07-permissions.md). Check Jellyfin's own user can write there:

```bash
pct exec <jellyfin-ID> -- su -s /bin/sh jellyfin -c "touch /transcodes/test && echo WRITE_OK"
```

Then Jellyfin: **Dashboard → Playback → Transcoding → Transcode path** = `/transcodes`. Over the next heavy viewing night, the container's disk sat flat at about 16% while the transcode folder grew and shrank on the big drive.

> ⚠️ **Trade-off I accepted.** That drive also holds the nightly backups. Don't schedule backups for movie night.

## Roads not taken

- **Plex:** most polished apps everywhere; account required, hardware transcoding and watching your own server from outside the house cost money.
- **Emby:** Jellyfin's parent; paid tier for the good stuff.
- **Client apps:** third-party players (Infuse, Swiftfin, Streamyfin) sometimes Direct Play what official apps would transcode. On weak hardware that matters more than any server setting.
- **A dedicated Samba user instead of root:** better practice; a leaked share password then isn't a root-level password. Read chapter 7 first if you do it.
- **File share:** NFS (faster between Linux machines; Part 2 uses it), or a file-server container instead of Samba on the host.

## Skip ahead?

Needed for a media library. Read the transcoding section even if you skip the rest.
