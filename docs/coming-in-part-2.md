# Side quests, and what's coming in Part 2

[← 10. Home Assistant](10-home-assistant.md) · [Guide](../README.md) · [What's still open →](whats-still-open.md)

## Where the one box ended up

By the end of chapter 10, one 2012-era PC with 16 GB of RAM is running the house's DNS, a streaming service, media automation, a private cloud, a photo library, a photo frame and the smart home. It cost $0.

It is also:

- **one power supply** away from the whole house losing DNS (which looks exactly like "the internet is down");
- **one drive** away from losing every photo, because the backups lived in the same box and the photos weren't even in them ([chapter 9](09-immich-and-the-backup-gap.md));
- **one reboot** away from the smart home, the ad blocker and movie night all going dark at once;
- out of RAM.

> 🤖 **Murderbot:** Single point of failure. I said this in chapter 3.
> **Me:** You said it in chapter 1.
> **Murderbot:** I was being polite in chapter 1.

## Side quests from the Book 1 era

These happened around the one box but weren't built *on* it, so they're not step-by-step here. Book 1 has the full versions.

- **A router behind the router.** The provider's gateway has no admin page, so a free old DSL router went in behind it ("double NAT, on purpose"). It came online and a laptop browsed through it end to end. The server itself was never moved behind it. Along the way I borrowed the server's own network cable to test it and took the whole house's DNS down. Use a spare cable.
- **A touchscreen control panel.** A cheap "Cheap Yellow Display" (an ESP32 board with a 3.5" touchscreen) running ESPHome: weather, cameras, and tomorrow's calendar, fed by Home Assistant. Murderbot reasoned its way to the wrong answer about the screen's colors, and I only caught it by asking "wait, why would going *back* make it appear?"
- **Hacking a dead photo frame.** A locked-down frame, five ways in, one accidental factory reset (twice), and a fine-tip soldering iron still on order.
- **A calendar on the wall, and the streaming stick that said no.** It can't install apps. At all. We checked.
- **The parts pile.** Once you build one server, people start handing you things. A five-step method for "is this junk worth using?", and a list of what I decided *not* to do.

## Coming in Part 2: the cluster

The one box had to go. Part 2 is what replaced it, built the same way: out of other people's junk, every command real, every mistake kept.

- **Five more computers** from e-waste piles and "you want this old thing?" offers: refurb office towers and tiny business PCs. Named after a certain 90s cartoon's planet-saving team. With their powers combined…
- **An old laptop that became the backup server**, whose battery doubles as a tiny UPS. Backups that live *outside* the thing they back up, finally.
- **A rescued NAS** with salvaged drives, set up so any two drives can die.
- **One cheap switch:** the only thing that had to be bought, apart from cables.
- **Moving every service off the old box**, one at a time, by backup and restore. Rule one: stop the old copy, never delete it, until the new one has run clean for a week.
- **Two Pi-holes**, so a reboot doesn't take the house's DNS with it (see [chapter 3](03-pihole.md)'s complaint).
- **Media moved onto the NAS**, and the Quick Sync transcoding chapter 4 couldn't have.
- **Dashboards that tell you something's down** before the house does.
- **A cluster that comes back on its own after a blackout.**
- And the old box's last day: still carrying everything, 18 days of uptime, while its guests moved off one by one.

The cluster has a name. It stays. It's not going in a heading.

> 🤖 **Murderbot:** Five computers. For a house.
> **Me:** Six, counting the laptop.
> **Murderbot:** I wasn't going to count the laptop.
> **Me:** The laptop is doing its best.

**Part 2 isn't in this repo yet.** Watch it, or star it, to see when it lands.
