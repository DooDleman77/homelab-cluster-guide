# 7. Interlude: the permissions lesson

[← 6. The automation stack](06-the-automation-stack.md) · [Guide](../README.md) · Next: [8. Nextcloud →](08-nextcloud.md)

Not a new service. A mistake, the rule that came out of it, and four rounds of getting file permissions right once several containers shared one drive. If you read one chapter besides chapter 1, read this one.

## The thirty-second background

Every Linux file has an **owner** (a user number, **UID**), a **group** (**GID**), and permissions for owner / group / everyone else: **r**ead = 4, **w**rite = 2, e**x**ecute = 1 (for a folder, "execute" = allowed to go inside). `750` = owner everything, group read + enter, everyone else nothing.

The twist: Proxmox containers are **unprivileged** by default. "Root" (UID 0) inside a container is mapped to a harmless high-numbered user on the real host:

```bash
cat /etc/subuid
# root:100000:65536
```

**Container UID N = host UID 100000 + N.** Same for groups (`/etc/subgid`). So root inside Radarr is user 100000 on the real drive: a stranger who owns nothing there.

## Round 1: the arr stack, and the lazy fix

[Chapter 6](06-the-automation-stack.md) ended with `Folder '/media/movies/' is not writable by user 'root'`. That night:

```bash
chmod -R 777 /mnt/media
```

Read/write/enter for owner, group **and everyone**, on every file on the drive, in one shot. It worked instantly. I moved on.

The next day I reread it and I wasn't okay with it. I wrote it down, and I'll repeat it here:

> **There will be no more `chmod -R 777` anywhere in this build. It's a lazy way to fix something, and it opens our progress up to the world.**

`777` doesn't say "these three programs may write here". It says "**every** process that can see this drive may change or delete **anything** on it", including everything in every other container that mounts it. It throws away the whole reason you put services in separate boxes.

> 🤖 **Murderbot:** It's a home network behind a router. The risk is low.
> **Me:** Then why are we bothering with containers at all?
> **Murderbot:** …I'll get the ACL commands.

## Round 2: Nextcloud, the first precise fix

Mounting a folder into Nextcloud ([chapter 8](08-nextcloud.md)) hit the same wall: inside, the folder showed as `nobody:nogroup` (an owner that maps to no one in the container). This time: give it to **exactly** the user that needs it. Nextcloud's images run as `www-data`, **UID 33** inside. 100000 + 33 = **100033**:

```bash
chown -R 100033:100033 /mnt/backup/nextcloud-data
chmod -R 750 /mnt/backup/nextcloud-data
```

Inside, it now showed `www-data www-data`. Nextcloud installed with zero permission errors. Honest note: 33 was an informed guess from Nextcloud's docs, not measured. It happened to be right. Measuring is better.

## Round 3: Immich, measure instead of guess

For the photo library ([chapter 9](09-immich-and-the-backup-gap.md)), inside the Immich container:

```bash
id immich
# uid=999(immich) gid=991(immich) ...
```

Mapped: 100999 and 100991.

```bash
chown -R 100999:100991 /mnt/media/immich
chmod -R 750 /mnt/media/immich
```

Fine for a folder with one owner. But `/mnt/media` has **several** services needing different access, and `chown` names only one owner.

## Round 4: undoing the 777, properly

**The need:** Jellyfin reads the library and writes artwork next to it. Sonarr, Radarr and qBittorrent write downloads and move files. Immich's photo folder (same drive): Jellyfin read-only at most, download tools no access.

**The tool: ACLs** (Access Control Lists): extra named entries on top of owner/group/everyone, like "user 100107 may read and write here", without changing the owner and without opening anything to everyone.

### Step 1: Measure each service's real host UID

Don't trust the math; prove it. Create a file **as the service's own user**, then look at it from the host:

```bash
pct exec <jellyfin-ID> -- su -s /bin/sh jellyfin -c "touch /media/uid-test-jellyfin"
ls -ln /mnt/media/uid-test-jellyfin      # -n = numbers instead of names
```

Jellyfin (UID 107 inside) showed up as **100107** on the host. Sonarr, Radarr and qBittorrent all run as **root** inside their containers (their scripts never made a service user), so all three are **100000**. Your numbers may differ: measure yours. Delete the test files.

### Step 2: Take away the 777

```bash
find /mnt/media -type d -exec chmod 750 {} \;     # folders
find /mnt/media -type f -exec chmod 640 {} \;     # files
```

Right now no service can write anything. Expected.

### Step 3: Grant exactly what's needed

```bash
setfacl -R -m u:100107:rwX /mnt/media
setfacl -R -m u:100000:rwX /mnt/media
setfacl -R -d -m u:100107:rwX /mnt/media
setfacl -R -d -m u:100000:rwX /mnt/media
getfacl /mnt/media/movies          # check: both users, plus matching default: lines
```

`-m` adds an entry. Capital `X` = enter folders without marking every movie as a program. `-d` sets a **default** that new files inherit; without it, every new download is born without the grant and you're back here next week.

### Step 4: Lock the photo folder back down

```bash
setfacl -R -x u:100000 /mnt/media/immich
setfacl -R -d -x u:100000 /mnt/media/immich
setfacl -R -m u:100107:rX /mnt/media/immich
setfacl -R -d -m u:100107:rX /mnt/media/immich
```

`-x` removes an entry. Immich's own ownership from round 3 stays as it was. The whole recipe: [`configs/acl-shared-media.sh.example`](../configs/acl-shared-media.sh.example).

> 💡 **Why ACLs and not a shared group?** A shared group means making the *same* group number exist and map correctly in four containers. Fiddly with unprivileged remapping. ACLs let two unrelated users have the same access without knowing about each other.

## Three more bugs hiding behind the first

What forced the cleanup: Jellyfin's posters kept disappearing, for weeks. After the ACL fix, the scan logs went perfectly clean.

> 🤖 **Murderbot Check.** Murderbot declared it fixed: clean logs. I sent screenshots of a library still full of blank placeholder art. Clean logs meant one check passed, not that the feature worked. There were three more causes, each hiding behind the last.

**Bug 1: the library watched folders it had no business touching.** One library pointed at all of `/media` kept failing on `Permission error for Directory watcher: /media/immich`. **Fix:** libraries at `/media/movies` and `/media/tv` only ([chapter 4](04-jellyfin.md)).

**Bug 2: a stuck database.** Every scan crashed on `UNIQUE constraint failed: UserData.ItemId, UserData.UserId, UserData.CustomDataKey`. Earlier, while permissions were broken, a delete had failed halfway and left orphaned rows. **Fix:** stop Jellyfin, **back up the database first**, delete only the orphaned rows, start Jellyfin:

```bash
pct enter <jellyfin-ID>
systemctl stop jellyfin
cp -a /var/lib/jellyfin/data/jellyfin.db /var/lib/jellyfin/data/jellyfin.db.bak-$(date +%F)
apt install -y sqlite3
sqlite3 /var/lib/jellyfin/data/jellyfin.db "DELETE FROM UserData WHERE ItemId = '00000000-0000-0000-0000-000000000001';"
systemctl start jellyfin
```

Your error differs? **Don't copy this `DELETE`.** Find your own stuck rows first. Never edit an app's database without a backup and with the app running.

**Bug 3: folders made over the file share were silently capped.** Folders I created from my Mac over Samba threw `UnauthorizedAccessException` when Jellyfin saved artwork, though the ACLs looked perfect. The answer, easy to skim past:

```
user:100107:rwx          #effective:r-x
mask::r-x
```

An ACL has a **mask**, a ceiling on every named entry. Samba created the folder with a mask of `r-x`, so `rwx` was *effectively* read-only. **Fix,** per folder:

```bash
setfacl -m mask::rwx "/mnt/media/movies/<folder-made-over-SMB>"
```

> ⚠️ **This one can come back.** I fixed the folders, not Samba. The likely permanent fix is `inherit acls = yes` in the `[media]` share. **We didn't apply or test that on the one box.** Until you do, spot-check new folders with `getfacl <folder> | grep effective`.

**Bug 4: one broken metadata source jammed the queue.** One online metadata provider returned malformed data for at least one episode (`'i' is invalid after a value`). Jellyfin retried it every refresh, and me impatiently re-clicking "Refresh metadata" before a run ever finished made it look stuck for weeks. **Fix:** turn that provider off for episodes (Dashboard → Libraries → Shows → Manage), then start **one** refresh and leave it alone.

After all four fixes and one patient refresh: real posters across the whole library, for the first time in weeks.

> 💡 **The lesson under the lesson.** A clean log means one thing stopped failing. Go look at the feature with your own eyes. And after fixes that "stuck" once and quietly reverted, the real test is: still working a day later, with nobody touching it?

## Roads not taken

| Approach | Verdict |
|---|---|
| `chmod -R 777` | **Never again** |
| `chown` + `750` | Right for a single-owner folder (Nextcloud, Immich) |
| **ACLs** *(chosen)* | Right for folders several services share |
| Shared group + setgid | The textbook answer; fiddly with unprivileged containers |
| Privileged containers | Every problem here disappears, and so does the protection |
| `lxc.idmap` | Powerful, easy to get wrong |
| Docker `PUID`/`PGID` | Common in Docker stacks; less separation |

## Skip ahead?

Don't, if more than one container touches the same drive.
