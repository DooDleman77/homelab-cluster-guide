# What's still open

[← Side quests and Part 2](coming-in-part-2.md) · [Guide](../README.md)

Everything below was **not done, or done but not tested**, on the one box. It's here so nothing in this guide claims more than it did. Some of it gets picked up in Part 2.

| Item | Where it stood | Ch. |
|---|---|---|
| **Photos not in any backup** (bind mount; `backup=1` silently ignored) | restic fix written up; **never applied or tested on this box** | [9](09-immich-and-the-backup-gap.md) |
| Nextcloud's files live on the backup drive and aren't backed up themselves | Needs restic or AIO's own backup to the other drive; not done | [8](08-nextcloud.md) |
| Off-site backups | None | 9 |
| Samba-made folders can be born with a capped ACL mask | Folders fixed by hand; `inherit acls = yes` **not applied or tested** | [7](07-permissions.md) |
| Unbound (private recursive DNS) | Not added | [3](03-pihole.md) |
| Local DNS names instead of numbers | Not set up | [2](02-network-first.md) |
| Byparr vs. FlareSolverr on the sources that failed | Installed and wired in; **never measured** | [6](06-the-automation-stack.md) |
| Dead sources removed, auto-disable confirmed in Prowlarr | Recommended; not confirmed | 6 |
| Audiobook pipeline | Researched; not built | 6 |
| Home Assistant VM memory at ~94% | Seen once; never re-checked | [10](10-home-assistant.md) |
| First Zigbee devices | Coordinator ready; nothing paired | 10 |
| ImmichFrame Funnel link | Set up; **never tested from outside the house** | 9 |
| ImmichFrame "some photos repeat" | Unconfirmed whether it's just randomness | 9 |
| ImmichFrame key that was pasted into a chat | Rotation recommended; not confirmed | 9 |
| A personal Nextcloud account instead of `admin` on the phone | Pinned | 8 |
| A dedicated Samba user instead of root | Pinned | [4](04-jellyfin.md) |
| Tailscale subnet router | Considered; not done | [5](05-tailscale.md) |
| Quick Sync on the old box's integrated graphics | Never tried | 4 |
| Uptime monitor, password manager | Next on the list when the one box ran out of room | — |

> 🤖 **Murderbot:** This list is longer than some of the chapters.
> **Me:** That's what honest looks like.
> **Murderbot:** It looks like a to-do list.
> **Me:** Same thing.
