# 9. Your own photo library (Immich), and the backup gap

[← 8. Nextcloud](08-nextcloud.md) · [Guide](../README.md) · Next: [10. Home Assistant →](10-home-assistant.md)

## What we're building and why

**Immich** replaces the big phone-photo services: automatic backup from every phone, timeline, albums, face recognition, search by what's in the picture, sharing. Then **ImmichFrame**, a small app that turns any screen into a photo frame for one album.

And this is the chapter where I found out my backups had been quietly missing things for months. **That part applies to you even if you never install Immich.**

## Before you start

Chapters 1, 2, 5 and 7. Free RAM: Immich's defaults ask for 6 GB.

**Why its own container, not inside Nextcloud:** they're separate systems with separate databases. Sharing a container wouldn't make them share photos, just a blast radius.

## The build

### Step 1: Check headroom

```bash
free -h      # look at "available", not "free"
pct list
```

About 15 GB total, 10–11 GB available. Room for one more.

### Step 2: Create the container, deliberately

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/immich.sh)"
```

**Advanced**, every prompt read: 4 cores / 6 GB RAM (the defaults; indexing is hungry), 20 GB disk grown to 32 GB after, my existing SSH key, **TUN/TAP: yes** (needed for Step 7; this time I caught it *before* the install), machine learning **CPU only** (this box's graphics can't help), host upgrade mid-install **declined**.

```bash
pct resize <immich-ID> rootfs +12G     # you can grow a container disk any time, never shrink it
```

> 🤖 **Murderbot Check: the time zone.** The confirmation screen showed my correct time zone. Murderbot, working from an old wrong note, told me to change it, on the container *and* in Immich's settings. I corrected it and we changed both back. The useful bit: **Immich reads its own `TZ=` from its `.env` file, separately from the container clock.** Wrong photo times? Check both.

### Step 3: First login, and the iPhone that couldn't find it

`http://<immich-IP>:2283`; the first account becomes admin. The web page worked on my phone's browser. The Immich **app** said **"Server is not reachable."**

**Fix:** iPhone **Settings → Immich → Local Network → On.** iOS has a separate permission for talking to devices on your home network, and the app doesn't always ask. Not listed? Force-quit and reopen the app.

### Step 4: The backup gap (the most important step in this chapter)

Before trusting it with irreplaceable photos: where exactly will they live, and are they actually backed up?

```bash
cat /etc/pve/jobs.cfg          # the nightly "All" job from chapter 1
pct config <jellyfin-ID>       # mp0: /mnt/media,mp=/media  <- a bind mount
```

Jellyfin's nightly backup: about 1.5 GB. The library it serves: hundreds of GB. The job backs up each container's own small disk and **silently skips the mounted drive.** "All containers" never meant "all data." Fine for movies. Not fine for photos.

```bash
pvesm status                   # storage: total / used / free
```

Second problem, found in those numbers: **no retention.** A full backup of everything every night, kept forever.

**Fix #1, retention** (done before any photos went in; this one works exactly as described):

```bash
pvesm set <backup-storage-name> --prune-backups keep-last=7
```

**A dedicated mount for photos**, not the shared one:

```bash
mkdir -p /mnt/media/immich
pct set <immich-ID> -mp0 /mnt/media/immich,mp=/media,backup=1
pct reboot <immich-ID>
pct exec <immich-ID> -- id immich
chown -R 100999:100991 /mnt/media/immich      # use YOUR measured numbers (chapter 7)
chmod -R 750 /mnt/media/immich
```

> ⚠️ **Correction: that `backup=1` does nothing here.** I added it believing it pulled the photos into the nightly backup. It doesn't. Proxmox's docs: the `backup` option is "only used for volume mount points", and "the contents of bind mount points are not backed up when using vzdump." `/mnt/media/immich` is a bind mount, so the flag is silently ignored. The backup log even says so: `excluding bind mount point mp0 ('/media') from backup (not a volume)`.
>
> **The photos were never in the nightly backup.** Immich's settings and database (the container's own disk) were; the photos themselves were not.
>
> **Check your own server now:** open the latest backup task (the node → Task History) and search for "excluding". Every line is a folder that isn't in your backups.

> 🤖 **Murderbot:** `backup=1` will include it.
> **Me:** Will it?
> **Murderbot:** …it will be *noted* that you wanted it included.

**The real fix: back the photo folder up separately.** The recommended route is **restic** on the host, nightly, to the **other** drive: incremental, encrypted, and the same tool can send a copy off-site later.

```bash
apt install restic
mkdir -p /mnt/backup/restic-immich
openssl rand -base64 32 > /root/.restic-pass && chmod 600 /root/.restic-pass
restic -r /mnt/backup/restic-immich --password-file /root/.restic-pass init
restic -r /mnt/backup/restic-immich --password-file /root/.restic-pass backup /mnt/media/immich
restic -r /mnt/backup/restic-immich --password-file /root/.restic-pass snapshots
```

Keep a copy of that password file **off the server**; without it the backup can't be opened. Nightly schedule and a restore test: [`configs/restic-photos.sh.example`](../configs/restic-photos.sh.example).

> ⚠️ **Honest status: we didn't test this on the one box.** The restic fix was written up while finishing Book 1 and never applied to this server before it was retired. It's the standard restic recipe, but it's untested *here*. Test a **restore** before you trust it: a backup you've never restored isn't a backup yet. (Part 2 takes another run at where the photos live.)

### Step 5: Point Immich at the new storage

Immich had already made its upload folder on its own small disk. Moving it = update the config file **and** the database.

```bash
pct enter <immich-ID>
systemctl stop immich-web immich-ml
cp -a /opt/immich/upload/. /media/          # only if you've uploaded anything already
ln -sfn /media /opt/immich/app/upload
ln -sfn /media /opt/immich/app/machine-learning/upload
sed -i 's|^IMMICH_MEDIA_LOCATION=.*|IMMICH_MEDIA_LOCATION=/media|' /opt/immich/.env
/opt/immich/app/bin/immich-admin change-media-location     # old: /opt/immich/upload  new: /media
systemctl start immich-ml immich-web
```

`ln -sfn` makes (or replaces) a signpost link. The `sed` line rewrites the media location in `.env`.

> ⚠️ **Landmine: the tool fixes the database, not the config file.** On this build the `sed` edit got lost (it was in the same batch as the time-zone commands that were undone). The migration tool said "success" because the database really was updated, but Immich kept reading the old location from `.env`. The giveaway: the storage widget showed **31.3 GiB** (the container's own disk) instead of the 1 TB drive. **Always check:** `grep -i media /opt/immich/.env`. Re-running the `sed` and restarting fixed it: **915.8 GiB**.

### Step 6: Reading the storage numbers

The widget's "used" is the **whole drive**, Jellyfin's library and downloads included. Immich's real size, from the host: `du -sh /mnt/media/immich`.

### Step 7: Sharing with family who will never install a VPN

**Tailscale Funnel** publishes one service to the **public internet** with real HTTPS, no router port opened. It must run on the machine serving the app; that's why TUN went on in Step 2.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
tailscale funnel --bg 2283
tailscale funnel status
```

The first time, it said Funnel wasn't enabled and gave a link to turn it on in the admin console. `--bg` matters: without it, Funnel dies when you close the terminal.

> ⚠️ **Landmine: never hand out that address as "the photo library".** It's Immich's real login page, now public. What you send is a **Shared Link**: album → Share → create link, with optional **expiry**, optional **password**, download on/off. It exposes exactly one album. Tested end to end; it worked.

### Step 8: "Swap is 100% full!" (It's fine.)

Weeks later the container showed memory 14%, **swap 99.97%**. Linux parks rarely used memory in swap even with RAM free, and it stays there. The problem would be constant movement:

```bash
pct exec <immich-ID> -- vmstat 1 5     # si / so columns near zero = nothing actively swapping
```

## ImmichFrame: a photo-frame page

A small app that shows a slideshow of chosen albums on any screen with a browser. **Its own small Docker container** (8 GB disk, 1 core, 1 GB RAM; Portainer agent no, Docker TCP socket no).

**A narrow API key.** Immich → Account Settings → API Keys → New, named `immichframe`, with only `asset.read`, `asset.view`, `asset.download`, `album.read`, `person.read`. No "Select all", nothing that writes. A photo frame has no reason to change anything.

> ⚠️ **Security note, self-inflicted.** I pasted the config into a chat **with the real key in it.** Even a read-only key doesn't belong in a chat log. Fix: delete the key in Immich, make a new one with the same permissions, update the config, restart. Do this any time a key lands somewhere it shouldn't.
>
> 🤖 **Murderbot:** I would like to note that I did not ask for the key.
> **Me:** Noted.
> **Murderbot:** In the "never" voice again.

**Config:** [`configs/immichframe/`](../configs/immichframe/) (compose file + `Settings.example.yml`). Then:

```bash
docker compose up -d     # NOT "restart": you can't restart what was never created
docker ps
docker logs immichframe --tail 50     # "Application started." (DataProtection warnings are harmless noise)
```

> ⚠️ **Landmine: the broadcast address.** `ip a` shows `inet <address>/24 brd <something>.255`. I opened the `.255` one first: `ERR_ADDRESS_INVALID`. `.255` is the **broadcast** address; no device ever has it. Use the `inet` address.

> 🤖 **Murderbot Check: the setting in the wrong place.** Videos never showed. The first config had `ShowVideos: true` under `General:`. ImmichFrame's docs say it's an **account-level** setting, under `Accounts:`. In the wrong section it's silently ignored. Moved, restarted, videos played. (The example config has it in the right place.)

**Not confirmed:** whether "some photos come up more often" is just randomness (no shuffle setting is documented). And I also put Funnel on the frame container so a screen elsewhere could show the album: **set up, never tested from outside the house.** Treat it as unverified.

## Roads not taken

- **PhotoPrism, Ente, LibrePhotos, Nextcloud Memories, PiGallery2:** see Book 1 for the long version. Immich won on phone backup and sharing.
- **Immich Kiosk** instead of ImmichFrame: very configurable.
- **Backups:** a volume mount point instead of a bind mount (then vzdump really includes it); **Proxmox Backup Server** (deduplicated, incremental; Part 2 builds one on a laptop); off-site copies. The **3-2-1 rule** says three copies, two kinds of media, one off-site. On the one box, the photos had **one** copy.

## Skip ahead?

Don't skip Step 4, even if you never install Immich.
