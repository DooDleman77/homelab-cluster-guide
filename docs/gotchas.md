# Landmines we stepped on (read this first)

[← Guide](../README.md)

The short list. Each one cost at least an evening on CommandCenter. The chapter has the full story and the fix.

## Setup

1. **`401 Unauthorized` on the first update.** Proxmox points at the paid repository. Switch to `pve-no-subscription`, and check for a Ceph file throwing the same 401. → [ch 1](01-old-pc-to-proxmox.md)
2. **The installer saves a static address.** Change networks (new internet provider) and the server insists on an address that no longer exists. Use DHCP + a router reservation, and fix `/etc/hosts` too. → [ch 2](02-network-first.md)
3. **Old notes lie.** An address written down before the network changed is wrong until re-checked. It bit twice more after the fix. → [ch 2](02-network-first.md), [ch 10](10-home-assistant.md)
4. **Host upgrades offered mid-install.** Say Ignore. Upgrade the host on purpose, later. → [ch 1](01-old-pc-to-proxmox.md)

## Containers

5. **A container with an address that can't talk to anything.** Its virtual cable was never plugged into the switch. `brctl show vmbr0` after every new container; `pct stop` + `pct start` fixes it. → [ch 3](03-pihole.md)
6. **`pct exec` doesn't expand `*`.** Wrap it: `pct exec <ID> -- bash -c "…"`. → [ch 4](04-jellyfin.md)
7. **`pct` only exists on the host,** and only for containers. VMs use `qm`. → [ch 6](06-the-automation-stack.md), [ch 10](10-home-assistant.md)
8. **Tailscale in an unprivileged container needs TUN.** Two lines in the container's config. Read the TUN line on a script's review screen *before* confirming. → [ch 5](05-tailscale.md), [ch 8](08-nextcloud.md)
9. **Root inside a container is a nobody on the real drive** (container UID N = host UID 100000 + N). → [ch 7](07-permissions.md)

## Data

10. **`chmod -R 777` is never the fix.** Measure each service's real UID, grant exactly that with ACLs. → [ch 7](07-permissions.md)
11. **ACL masks.** Folders made over Samba can be capped to read-only even when the ACL says `rwx`. Look for `#effective:`. → [ch 7](07-permissions.md)
12. **The nightly "All" backup skips bind-mounted folders**, with or without `backup=1`, and keeps everything forever until you set retention. Search your backup log for "excluding". → [ch 9](09-immich-and-the-backup-gap.md)
13. **A migration tool that says "success" may have fixed only half.** Immich's tool updated the database, not `.env`. Check the file. → [ch 9](09-immich-and-the-backup-gap.md)
14. **Jellyfin's transcode cache can fill its disk.** Move it to a big drive. → [ch 4](04-jellyfin.md)

## Performance

15. **An old GPU can't transcode.** Keep the library in H.264. → [ch 4](04-jellyfin.md)
16. **A bot-wall helper can eat the whole CPU** (load average 94 on 4 threads). Delete dead sources, shorten timeouts. When interactive tools hang, use `uptime` and `free -h`. → [ch 6](06-the-automation-stack.md)
17. **Pi-hole's per-client rate limit** looks like "the internet is broken" while the dashboard says 0.4% CPU. A chatty download client was the cause. → [ch 3](03-pihole.md)

## Home Assistant

18. **Never interrupt a VM's first boot.** One stop/start to change the CPU type gave a crash loop no restart, downgrade or rebuild of Core could fix. Destroy and rebuild. → [ch 10](10-home-assistant.md)
19. **A "Healthy" status page may be watching the wrong layer.** Port 4357 reports the Supervisor, not Home Assistant itself. → [ch 10](10-home-assistant.md)
20. **USB added to a running VM isn't seen until a restart.** → [ch 10](10-home-assistant.md)

## Secrets

21. **Login links, API keys and passphrases never go in a chat, note or screenshot.** If one does, rotate it. → [ch 5](05-tailscale.md), [ch 9](09-immich-and-the-backup-gap.md)

## Murderbot

22. **A clean log is not a working feature,** and an AI co-pilot is confidently wrong sometimes. Trust, then verify, and say so when something doesn't add up.

> 🤖 **Murderbot:** I object to number 22.
> **Me:** Noted.
