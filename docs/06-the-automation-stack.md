# 6. The automation stack (the "arr" apps)

[← 5. Tailscale](05-tailscale.md) · [Guide](../README.md) · Next: [7. The permissions lesson →](07-permissions.md)

## Read this first

This chapter is about software that searches for, downloads, renames and files media automatically. It does **not** cover, recommend or name any source of copyrighted content, on purpose. Everything here works the same for legal uses: Linux images, public-domain films and old TV, Creative Commons releases, podcasts, your own discs. What you point these tools at is your responsibility.

## What we're building and why

Small single-purpose tools that hand work to each other. Most names end in "-arr", so: the **arr stack**.

| Tool | Job | Port |
|---|---|---|
| **Prowlarr** | Keeps the list of search sources ("indexers") and shares it with the others | 9696 |
| **Sonarr** | TV: finds, downloads, renames and files new episodes | 8989 |
| **Radarr** | Same, for movies | 7878 |
| **qBittorrent** | The download engine | 8090 (as installed here) |
| **FlareSolverr**, later **Byparr** | Gets past "checking your browser" pages some sources use | 8191 |

Each gets its own container. One breaks, the rest keep running.

## Before you start

Chapters 1–4. **Read [chapter 7](07-permissions.md) right after this one**: the permissions fix used here was the wrong one.

## The build

### Step 1: Prowlarr

```bash
bash -c "$(wget -qLO - https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/prowlarr.sh)"
brctl show vmbr0
pct config <prowlarr-ID> | grep net0     # the MAC, for a router reservation
```

Defaults: Debian 13, unprivileged, 4 GB disk, 2 cores, 1 GB RAM. Open `http://<prowlarr-IP>:9696`, make a login.

### Step 2: Add sources

**Indexers → Add Indexer.** Name, API key (only for sources with accounts), Categories, Sync Profile/Priority, and **Tags** (links a source to a helper, below). **Test** each. Some fail with:

```
Unable to access <source>, blocked by CloudFlare Protection.
```

### Step 3: FlareSolverr

FlareSolverr runs an invisible real browser that sits through the protection page and hands the result back.

```bash
bash -c "$(wget -qLO - https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/flaresolverr.sh)"
```

Prowlarr: **Settings → Indexers → Indexer Proxies → `+` → FlareSolverr.** Host `http://<flaresolverr-IP>:8191`, give it a **Tag**, put the same tag on sources that need it. Test, Save.

> ⚠️ **Landmine: "Disabled" doesn't always mean disabled.** The proxy card said **Disabled.** It wasn't: `pct exec <flaresolverr-ID> -- systemctl status flaresolverr` said `active (running)`, and its log showed Prowlarr's requests coming back `200 OK`. Test a real source before deciding anything's broken.

One source never cleared, ever: `Timeout after 60.0 seconds`, while every other page cleared instantly. **One source always fails, everything else works** = that site's protection is winning. I found a one-person workaround proxy and passed: unvetted code, more complexity, for one source. Dropping a stubborn source beats winning that fight.

### Step 4: Sonarr and Radarr

```bash
bash -c "$(wget -qLO - https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/sonarr.sh)"
bash -c "$(wget -qLO - https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/radarr.sh)"
```

Logins on `:8989` and `:7878`. Let Prowlarr push sources to both: **Prowlarr → Settings → Apps → `+` → Sonarr**, with Sonarr's address and API key (Sonarr: Settings → General → Security). Same for Radarr.

### Step 5: qBittorrent, and the password hunt

```bash
bash -c "$(wget -qLO - https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/qbittorrent.sh)"
```

Web page: `http://<qbittorrent-IP>:8090`, user `admin`. The password is a temporary one printed **once** in the log at first start. By the time I looked, it was gone. The hunt, honestly:

1. `cat /root/qbittorrent.creds` — empty.
2. `ournalctl …` — typo. It happens.
3. `journalctl -u qbittorrent-nox -n 50 --no-pager | grep -i password` — already scrolled out.
4. Ran `pct exec …` from *inside* the container: `pct: command not found`. `pct` only exists on the host.
5. The config file shows only a one-way hash (`WebUI\Password_PBKDF2=…`). Can't be reversed.

**The reset:** delete the hash, restart, it prints a fresh temporary password:

```bash
systemctl stop qbittorrent-nox
sed -i '/WebUI\\Password_PBKDF2/d' /root/.config/qBittorrent/qBittorrent.conf
systemctl start qbittorrent-nox
journalctl -u qbittorrent-nox -n 50 --no-pager | grep -i "temporary password"
```

Log in, set a real password right away (**Tools → Options → Web UI**).

### Step 6: Connect Sonarr and Radarr to qBittorrent

In each: **Settings → Download Clients → `+` → qBittorrent.** Host, port `8090`, `admin` + password. Category `sonarr` in Sonarr, `radarr` in Radarr, so they don't grab each other's files.

### Step 7: One drive, one set of paths, seen the same way by everyone

Skip this and finished downloads never show up in your library. qBittorrent, Sonarr and Radarr must all see **the same folder at the same path**, exactly like Jellyfin:

```bash
pct set <sonarr-ID> -mp0 /mnt/media,mp=/media
pct set <radarr-ID> -mp0 /mnt/media,mp=/media
pct set <qbittorrent-ID> -mp0 /mnt/media,mp=/media
pct reboot <sonarr-ID>; pct reboot <radarr-ID>; pct reboot <qbittorrent-ID>

mkdir -p /mnt/media/downloads/incomplete /mnt/media/downloads/complete /mnt/media/tv /mnt/media/movies
```

(I did qBittorrent first and forgot the other two. They're the ones that move files. They need it too.)

qBittorrent → **Tools → Options → Downloads**: save path `/media/downloads/complete`, incomplete in `/media/downloads/incomplete`. Leave **Torrent Management Mode** on **Manual**: in Automatic, qBittorrent moves finished files itself while Sonarr/Radarr also move them, and two programs moving the same files ends in "file not found".

> 💡 **Why one drive matters: hardlinks.** Downloads and library on the same drive, same `/media` mount everywhere = Sonarr/Radarr can **hardlink** instead of copy: the file appears in the library instantly, no extra space. Check **Settings → Media Management → Use Hardlinks instead of Copy** is on.

### Step 8: Library folders

Sonarr → Settings → Media Management → Root Folders → `/media/tv`. Radarr → `/media/movies`. And Radarr refused:

```
Folder '/media/movies/' is not writable by user 'root'
```

Root can't write? In an unprivileged container, "root" is root only inside its own box; on the real drive it's a nobody. That night I ran `chmod -R 777` on the whole media drive. It worked. It was wrong. [Chapter 7](07-permissions.md) is why.

> 🤖 **Murderbot:** That will work.
> **Me:** Will it work *well*?
> **Murderbot:** It will work.

After that, the pipeline ran end to end: request → search through Prowlarr → download → import, rename, hardlink → Jellyfin picks it up.

## What went wrong: the night the whole server stopped answering

Weeks later, sources in Prowlarr would spin forever on Test. Prowlarr's log: timeouts, every one running through the same line, `FlareSolverr.PostResponse`. One FlareSolverr problem, not many source problems.

- `journalctl` inside the FlareSolverr container: **hung**.
- `htop` on the **host**: **hung**. That was the clue. If the host can't draw a process list, the whole machine is drowning.
- `curl` to FlareSolverr's health page: connected, then nothing, ever.

> 🔍 **Diagnostic pattern: when interactive tools hang, use one-shot commands.** `uptime` and `free -h` print once and exit.

```
load average: 94.39, 92.31, 68.52
```

This CPU runs **four** things at once. A load of 94 = about twenty times more work than it can do. Memory was fine; pure CPU starvation.

**Tonight's fix:** force it off from the outside:

```bash
pct stop <flaresolverr-ID> --skiplock
pct start <flaresolverr-ID>
```

**The cause,** from FlareSolverr's own log: Prowlarr fired a burst of requests, each allowed up to **180 seconds**, at sources that were dead two ways: some **banned** the address outright (no tool on your end fixes a ban), others showed a **challenge** the helper couldn't solve and hung the full three minutes. Each attempt = a real browser running flat out. Several stacked on a four-thread CPU that also runs everything else: load 94.

**Lasting fixes:**

1. **Delete dead sources; don't just disable them.** Prowlarr re-checks them on a schedule and restarts the storm.
2. Make sure Prowlarr's auto-disable-on-failure is on.
3. Shorter timeouts.
4. A better helper: **Byparr**.

### Step 9: FlareSolverr → Byparr

Byparr does the same job a different way and speaks the same language, so Prowlarr can't tell the difference.

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/byparr.sh)"
```

Prowlarr → Indexer Proxies → edit the proxy → Host `http://<byparr-IP>:8191` → Test, Save. Old FlareSolverr container **stopped, not deleted**, as a fallback.

**Honest status:** Byparr is installed and wired in. **We never measured** whether it beats FlareSolverr on the sources that failed, and never confirmed the dead sources were all deleted. It can't do anything about a ban either way.

> 🤖 **Murderbot:** Four threads. A whole web browser per attempt. Three minutes each.
> **Me:** In my defense, I didn't know it was three minutes.
> **Murderbot:** It was in the log.
> **Me:** Everything is in the log. That's the problem with the log.

## Planned, not built: audiobooks

Researched, not built on the one box. Readarr (the "Sonarr for books") was retired in 2025; successors exist. The plan was **Audiobookshelf** as the player (no built-in remote access, so Tailscale), a phone app that talks to it, and legal sources only: DRM-free stores, your library's lending app, public-domain recordings. No DRM stripping, ever.

## Roads not taken

- **Jackett** instead of Prowlarr: older; you add each source to each app by hand.
- **No bot-wall helper at all:** if your sources aren't protected, skip it and never meet the night above.
- **Transmission / Deluge / rTorrent** instead of qBittorrent.
- **Usenet** (SABnzbd/NZBGet): paid providers, no uploading, often faster.
- **Jellyseerr:** a friendly "request a show" page for the household. First add-on I'd install next.
- **Tdarr:** converts your library to H.264 in the background, overnight. The automatic fix for [chapter 4](04-jellyfin.md)'s transcoding problem.
- **One Docker Compose file** for the whole stack: tidy; you trade away per-app snapshots and limits.

## Skip ahead?

Optional; nothing later depends on it. But if you built it, read [chapter 7](07-permissions.md) before you leave it running.
