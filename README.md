# Spare Parts Server, Part 1: One Box From Junk

**A free, step-by-step homelab guide: one old desktop PC that was headed for the curb, turned into a server that blocks ads for the whole house, streams a movie library, syncs files, backs up phone photos and runs the smart home.**

This is the free guide version of *Spare Parts Server*, Book 1. Same story, same order, same mistakes. The commands are the ones that actually ran. Where something was planned but never done, or done but never tested, the guide says so.

> 🤖 **Murderbot:** For the record, I said this would take a weekend.
> **Me:** It took months.
> **Murderbot:** I said *a* weekend. I didn't say which one.

---

## Who's talking

**Me:** DooDleman77. I build the thing, I type every command, I hold the screwdriver.

**Murderbot:** my AI co-pilot. It dug up commands, walked me through the landmines, stepped on a few of its own, and kept dragging me back to the plan every time I wandered off after something shiny. It would rather be watching its shows. It is not in the server: nothing on the box runs AI, and every command here was typed by a human. When Murderbot got something wrong, it stays in, in a box marked **🤖 Murderbot Check**. We argue. The arguments are in here too, kept short.

## The box: CommandCenter

One ordinary tower PC from the early 2010s, named **CommandCenter**. Nothing special about it, and that's the point. It keeps the name until Part 2 retires it.

| Part | What it is | What it means for this guide |
|---|---|---|
| CPU | A 2012-era quad-core Intel desktop chip, 4 threads | Plenty for a dozen light services. Chokes if you ask for several heavy things at once ([chapter 6](docs/06-the-automation-stack.md) shows exactly what that looks like) |
| RAM | 16 GB | The real budget. By the end it's mostly spoken for |
| GPU | An old graphics card with no hardware video encoding | Can't help with video transcoding. [Chapter 4](docs/04-jellyfin.md) is about living with that |
| Boot drive | 500 GB SATA SSD | Proxmox, plus every container's own small disk |
| Data drive | 1 TB hard drive, found in a drawer | `/mnt/media`: movies, TV, downloads, photos |
| Backup drive | 1 TB hard drive, also found | `/mnt/backup`: nightly backups, Nextcloud's files |

**Hardware cost: $0.** Everything was already lying around.

## What ends up running on it

| Service | Type | Chapter | Job |
|---|---|---|---|
| Proxmox VE | The host | [1](docs/01-old-pc-to-proxmox.md) | Turns one PC into many walled-off boxes |
| Pi-hole | LXC container | [3](docs/03-pihole.md) | Blocks ads and trackers for every device in the house |
| Jellyfin | LXC container | [4](docs/04-jellyfin.md) | Your own streaming service |
| Tailscale | In several boxes + the host | [5](docs/05-tailscale.md) | Reach it all from anywhere, no ports opened |
| Prowlarr, Sonarr, Radarr, qBittorrent, Byparr | LXC containers | [6](docs/06-the-automation-stack.md) | Media automation (legal sources only) |
| Nextcloud All-in-One | Docker inside an LXC | [8](docs/08-nextcloud.md) | Files, calendar, contacts |
| Immich + ImmichFrame | LXC containers | [9](docs/09-immich-and-the-backup-gap.md) | Phone photo backup, and a photo-frame page |
| Home Assistant OS | The one full VM | [10](docs/10-home-assistant.md) | Smart-home hub with a Zigbee radio |

## The guide

Follow the chapters in order. Each one has: what we're building and why, what you need, the real steps with every command explained, what went wrong, and what's still open.

| # | Chapter | Time it took |
|---|---|---|
| ⚠ | [**Landmines we stepped on**](docs/gotchas.md): read this first | 10 min |
| 1 | [Turning an old PC into a server (Proxmox VE)](docs/01-old-pc-to-proxmox.md) | an evening |
| 2 | [Getting the network right before anything else](docs/02-network-first.md) | 30 min (it cost me two evenings) |
| 3 | [Blocking ads for the whole house (Pi-hole)](docs/03-pihole.md) | 30 min |
| 4 | [Your own streaming service (Jellyfin + a file share)](docs/04-jellyfin.md) | an evening |
| 5 | [Reaching your server from anywhere (Tailscale)](docs/05-tailscale.md) | an hour |
| 6 | [The automation stack (the "arr" apps)](docs/06-the-automation-stack.md) | an evening, plus one bad night |
| 7 | [Interlude: the permissions lesson](docs/07-permissions.md) | one very long night |
| 8 | [Your own cloud (Nextcloud AIO behind Tailscale)](docs/08-nextcloud.md) | an evening |
| 9 | [Your own photo library (Immich), and the backup gap](docs/09-immich-and-the-backup-gap.md) | an evening |
| 10 | [Bringing in Home Assistant (and Zigbee)](docs/10-home-assistant.md) | two tries |
| ★ | [Side quests, and what's coming in Part 2](docs/coming-in-part-2.md) | 5 min |
| ☐ | [What's still open](docs/whats-still-open.md) | the honest list |

Example configs are in [`configs/`](configs/). Every value in them is a placeholder.

## Rules this guide follows

1. **Before any installer: unplug every drive whose contents matter.** Installers erase things.
2. **Fix the network before you build on it** ([chapter 2](docs/02-network-first.md)). Never trust an address you haven't re-checked since the network last changed.
3. **No `chmod -R 777`. Ever.** It's a lazy fix and it opens everything to everything. [Chapter 7](docs/07-permissions.md) is the whole reason this rule exists.
4. **"All" in a backup job doesn't mean all your data.** Check what's actually in it ([chapter 9](docs/09-immich-and-the-backup-gap.md)).
5. **No ports opened to the internet.** Remote access goes through Tailscale.
6. **No secrets in files you share.** Passwords, API keys and login links go in a password manager, never in a chat, a note or a screenshot.
7. **Host upgrades are their own job.** When an install script offers to upgrade the whole host mid-install, say Ignore and do it later, on purpose.
8. **A clean log is not a working feature.** Go look at the thing with your own eyes.

## Addresses in this guide

Every address is a placeholder (`<pihole-IP>`) or a made-up example on `192.168.1.0/24`. Tailscale addresses are written `100.x.y.z`. Yours will be different; each chapter shows how to find your own.

## Coming in Part 2

One box, one power supply, one drive away from losing the house's DNS, smart home and media all at once. Murderbot pointed that out more than once. Part 2 is what happened next: five more junk PCs, a laptop that turned into a backup server, a rescued NAS, and a cluster with a name I can't print in a title. [The teaser is here.](docs/coming-in-part-2.md)

## License

Docs: **CC BY 4.0**. Configs: **MIT**. Details in [LICENSE.md](LICENSE.md). Made by DooDleman77.

*Not affiliated with any project or company named here. Product names belong to their owners. Legal use only: nothing here explains how to get content you don't have the right to. Homelabs break; back up anything you care about before you follow a step that formats, deletes or rewrites something.*
