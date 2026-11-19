# 8. Your own cloud (Nextcloud AIO behind Tailscale)

[← 7. Permissions](07-permissions.md) · [Guide](../README.md) · Next: [9. Immich and the backup gap →](09-immich-and-the-backup-gap.md)

## What we're building and why

**Nextcloud** replaces the big commercial cloud suites: file sync, calendar, contacts, share links, an app store. This chapter builds it the way Nextcloud's own developers recommend, **Nextcloud All-in-One (AIO)**, reachable from anywhere through Tailscale with a real HTTPS certificate and no ports open.

It holds personal files, so I wrote a checklist **before** building and held the build to it:

1. Remote access through Tailscale only. No port forwarding.
2. HTTPS with a real, trusted certificate.
3. Two-factor authentication.
4. Brute-force protection confirmed on.
5. Backups that actually cover the data.
6. Kept updated more carefully than the media apps.

(Spoiler: number 5 didn't make it on the one box. See the end.)

## Before you start

Chapters 1, 2, 5 and 7. A home for the data: I used `/mnt/backup/nextcloud-data` on the backup drive. (That choice has an honest problem; see What's still open.)

## Choosing how to install it

There's **no community LXC script for Nextcloud itself**: the collection offers a TurnKey Linux VM, or NextcloudPi. I read the TurnKey VM's notes and passed: manual disk partitioning and boot-loader steps at first boot, a known keyboard-layout bug that can silently scramble passwords with `@`, `=` or `\`, and VM storage instead of the simple bind mounts everything else uses.

**Decision:** a plain Debian container with Docker, running **Nextcloud AIO**. One "master" container installs and manages the rest (web server, database, cache, the app) and gives you one admin page. Containers inside a container sounds odd; it's normal and well supported.

## The build

### Step 1: A Docker container, Advanced settings

```bash
bash -c "$(wget -qLO - https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/docker.sh)"
```

The defaults (2 CPU, 2 GB RAM, 4 GB disk) are too small once AIO pulls images. **Advanced**:

| Prompt | Choice | Why |
|---|---|---|
| Type | Unprivileged | Safer; the script supports Docker unprivileged |
| CPU | 2 | A soft cap, not a reservation |
| RAM | 4096 MB | Database, cache, app |
| Disk | 20 GB | System and images only; files live on the mounted drive |
| Root SSH | No | `pct enter` is enough; one less login door |
| FUSE / extra mounts | No / none | Storage gets attached from the host afterward |

The review screen showed Nesting and Keyctl **on** (Docker needs both) and **TUN: no**.

> ⚠️ **Landmine: TUN, again.** Tailscale needs TUN ([chapter 5](05-tailscale.md)). The script never asked; it only showed on the review screen. I noticed, tried to go back, and the wizard wouldn't go back that far. Fixable (Step 2). Next time, read the TUN line before confirming.

After the build it asked two more things: **Portainer Agent → N** (needs a separate Portainer server; AIO has its own page), and **Expose Docker TCP socket → n** (that hands root-level control of every container to anyone who can reach it).

### Step 2: Add TUN by hand

Same two lines as chapter 5, at the bottom of `/etc/pve/lxc/<nextcloud-ID>.conf`:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Then `pct reboot <nextcloud-ID>`.

### Step 3: Mount the data folder and set its owner

```bash
mkdir -p /mnt/backup/nextcloud-data
pct set <nextcloud-ID> -mp0 /mnt/backup/nextcloud-data,mp=/ncdata
pct reboot <nextcloud-ID>
chown -R 100033:100033 /mnt/backup/nextcloud-data
chmod -R 750 /mnt/backup/nextcloud-data
```

A dedicated subfolder, not the whole backup drive, so Nextcloud never sees the Proxmox backup files. Owner 100033 = `www-data` (chapter 7, round 2).

### Step 4: First start of AIO (the version I'd redo)

Inside the container, AIO's standard quick start: ports 80, 8080 and 8443, `NEXTCLOUD_DATADIR="/ncdata"`. It came up on `https://<nextcloud-container-IP>:8080` and showed an eight-word **passphrase** for the AIO admin page.

> ⚠️ **Landmine: two different logins.** The AIO passphrase opens AIO's management page. The Nextcloud **admin password** (later) logs into Nextcloud. Password manager, both. Never in a chat or a document.

### Step 5: The wall: AIO wants a public domain

Next screen: a **domain name** pointing at this server from the public internet, with port 443 reachable. That's AIO's default "normal mode", and it assumes port forwarding. The opposite of rule 1.

Quick option: skip domain checks, self-signed certificate, "not secure" warning forever. **Better option:** let **Tailscale** supply the name and a real certificate. More work, and it finishes the remote-access goal at the same time. I read AIO's reverse-proxy docs before touching anything.

> 🤖 **Murderbot:** Option one is faster.
> **Me:** Option one has a "not secure" warning on my own files forever.
> **Murderbot:** Faster, I said. Not better.

### Step 6: Tailscale in the container, with `serve`

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
tailscale status
```

In the **Tailscale admin console** (a web page): **MagicDNS** on (it was), **HTTPS Certificates** on (it was off). The container's name became `<container-name>.<your-tailnet>.ts.net`. That's Nextcloud's domain.

```bash
tailscale serve --bg http://127.0.0.1:11000
```

`serve` = tailnet-only HTTPS. `--bg` keeps it running across reboots. `127.0.0.1` = the container talking to itself, so Nextcloud's web server isn't reachable from the house network at all, only through Tailscale. Port 11000 just has to match the next step.

### Step 7: Recreate AIO in reverse-proxy mode

```bash
docker stop nextcloud-aio-mastercontainer
docker rm nextcloud-aio-mastercontainer
```

Then run it again with the settings in [`configs/nextcloud-aio-tailscale.sh.example`](../configs/nextcloud-aio-tailscale.sh.example). What changed:

- **Dropped ports 80 and 8443.** Tailscale handles incoming HTTPS now.
- **`APACHE_PORT=11000`, `APACHE_IP_BINDING=127.0.0.1`**: Nextcloud listens only on localhost:11000, exactly where `serve` sends traffic.
- **`SKIP_DOMAIN_VALIDATION=true`**: skips AIO's "is this reachable from the internet?" self-test, which can't understand a tailnet-only name. Not a security downgrade; Tailscale decides who connects.
- Kept 8080 for the AIO admin page on the local network.

> 💡 **Why deleting it lost nothing.** AIO's settings (and the passphrase) live in the **named volume** `nextcloud_aio_mastercontainer`, which survives `docker rm`.

Back on the AIO page, the tailnet name was accepted as the domain.

### Step 8: Install, with almost every extra off

Current stable release. **All optional containers unchecked** (antivirus, full-text search, Talk, office, and the rest): AIO wants 40 GB+ of disk once extras are on, and this container has 20. Talk would also need router ports forwarded. Time zone: set it (it's Nextcloud's own, separate from the container clock). Community containers: skipped for a first install holding personal files.

Five to ten minutes of image downloads, then every container running and the Nextcloud **admin** password shown **once**.

### Step 9: The proof

`https://<container-name>.<your-tailnet>.ts.net/` from any tailnet device. **No certificate warning**, unlike every other admin page in this build. And files writing to `/ncdata` with no errors confirmed the permissions.

### Step 10: Two-factor authentication

> ⚠️ **Landmine: the wrong settings page.** **Administration → Security → Two-Factor Authentication** is the policy for *enforcing* 2FA on groups. Not what you want first.

The right path: avatar → **Personal settings → Security → Two-Factor Authentication → TOTP** → scan the QR code. No authenticator app needed on an iPhone: the built-in **Passwords** app → `+` → **Set Up Verification Code**. Enter the code, then **save the backup codes** somewhere safe.

### Step 11: Brute-force protection

**Administration → Security → Brute-force settings.** On by default; confirm it. The allow-list page showed my address as a Tailscale `100.x` address: proof I was coming in over the tailnet. I didn't allow-list anything; Nextcloud's own advice is as few as possible, ideally none.

Checklist: Tailscale only ✅ · real HTTPS ✅ · 2FA ✅ · brute-force ✅.

### Step 12: Phones

Tailscale must be **connected on the phone** (the name isn't a public domain). In the Nextcloud app, enter the `https://…ts.net` address; it logs in through the web page with 2FA and gives the app its own token, so the phone never stores your real password.

> 📌 **Pinned:** I logged the phone in as `admin`. Better: a personal account for daily files, `admin` for admin only. Not done.

## What's still open

- **The data lives on the backup drive, and nothing backs it up.** Proxmox's backup job never includes bind-mounted folders ([chapter 9](09-immich-and-the-backup-gap.md)). If that drive dies, the files are gone. Fixes: restic to the *other* drive (chapter 9), or AIO's built-in backup pointed at the other drive. **Neither was done on the one box.** Checklist item 5: ❌.
- **Sharing a calendar with someone else in the house:** planned (second user, invited to the tailnet, calendar shared by username). Not done.

## Roads not taken

- **TurnKey VM / NextcloudPi / Snap / manual install:** see "Choosing how to install it".
- **ownCloud Infinite Scale / OpenCloud:** fast Go rewrites, fewer apps.
- **Seafile:** very fast sync; stores files in its own format.
- **Syncthing:** device-to-device sync, no server, no calendar.
- **Radicale or Baïkal:** if you only want calendar and contacts.
- **Funnel instead of serve:** would put the login page on the public internet. For personal files, tailnet-only is the right default.

## Skip ahead?

Standalone.
