# 3. Blocking ads for the whole house (Pi-hole)

[← 2. Network first](02-network-first.md) · [Guide](../README.md) · Next: [4. Jellyfin →](04-jellyfin.md)

## What we're building and why

**Pi-hole** blocks ads and trackers for every device on your network at once, with nothing installed on any of them. It works through **DNS**, the internet's phone book: before your TV loads `ads.example.com`, it asks a DNS server for that name's address. Pi-hole becomes the house's DNS server. Normal names get answered; names on its blocklists get "nothing here", and the ad never loads. Phones, laptops, TVs, consoles, and the gadgets that phone home without asking: one install covers them all.

## Before you start

[Chapter 1](01-old-pc-to-proxmox.md) and [chapter 2](02-network-first.md). Access to your router's DNS settings, or patience to set DNS on each device.

## The build

### Step 1: Create the container with a community script

From here on, most services are installed with the **community-scripts Proxmox VE helper scripts**: an open-source collection with one script per popular self-hosted app. Each one creates an LXC container and installs the app, from one command on the Proxmox host:

```bash
bash -c "$(wget -qLO - https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/pihole.sh)"
```

Inside out: `wget -qLO -` downloads the script quietly, follows redirects and prints it; `$( … )` captures that; `bash -c` runs it. (Newer versions of the collection use `curl -fsSL` instead; same idea.)

> ⚠️ **Landmine: that's someone else's script, running as root.** The collection is popular, open and reviewed, which is why I trusted it. You can always open the URL in a browser and read it first. Pi-hole's is short.

I chose **Basic (Default)** settings: a small unprivileged container, one CPU core, no extras (it skips **Unbound**, see Roads not taken). The script prints the container ID and `http://<pihole-IP>/admin`.

> 📌 **Pinned: one core was too few.** Fine for months, then it wasn't (part 2 below). Installing fresh? Choose Advanced and give it two.

### Step 2: Set your admin password

```bash
pct enter <pihole-ID>
pihole setpassword
exit
```

`pct enter` opens a shell inside the container. (If `pihole` isn't found, use `/usr/local/bin/pihole setpassword`.) Nothing shows while you type; that's normal.

### Step 3: Reserve Pi-hole's address

Same as [chapter 2](02-network-first.md). The whole house is about to depend on this address. If it moves, the house loses DNS, which looks exactly like "the internet is down".

### Step 4: Point the house at Pi-hole

In your router, find the **DNS servers** that DHCP hands out and set the primary to Pi-hole's address. On this house's gateway that was the provider's app → Advanced → DNS → **Custom DNS**, Primary and Secondary.

> ⚠️ **Landmine: the app wouldn't let Primary and Secondary match.** I wanted both set to Pi-hole. Refused. I ended with Primary = Pi-hole, Secondary = a hosted filtering DNS service I'd used before. It works, but devices may use the secondary whenever they like, and then Pi-hole's blocking is skipped. The real fix is a **second Pi-hole**. (Part 2 has two. Just saying.)

Router won't let you change DNS at all? Set it per device:

- **Mac:** System Settings → Network → Wi-Fi → Details → DNS → `+`.
- **iPhone/iPad:** Settings → Wi-Fi → ⓘ → Configure DNS → Manual → Add Server.
- **Windows:** Settings → Network & Internet → your connection → DNS server assignment → Edit → Manual.

### Step 5: Prove it works

```bash
nslookup google.com
```

The first lines say which DNS server answered: it should be Pi-hole. Then open Pi-hole's **Query Log** and watch lookups arrive live, blocked ones marked. On a normal day here: about 19,000 lookups, about 13% blocked. Devices already connected may hold the old DNS until you toggle their Wi-Fi.

## What went wrong, part 1: the container that couldn't reach anything

After the network fix in chapter 2, I rebooted Pi-hole. It got an address, and nothing could reach it. Jellyfin, right next to it, was fine.

> 🔍 **Diagnostic pattern: "it has an IP but nothing works."** Shrink the problem one layer at a time, and compare with a container that works.

```bash
pct exec <pihole-ID> -- ping -c 3 <gateway-IP>     # 100% loss, "Destination Host Unreachable"
pct exec <jellyfin-ID> -- ping -c 3 <gateway-IP>   # fine. So it's this one container.
pct exec <pihole-ID> -- ip route show              # default route: normal
pct config <pihole-ID> | grep -i firewall          # nothing
pct exec <pihole-ID> -- ping -c 3 <jellyfin-IP>    # also unreachable. That's the clue.
```

Reaching a neighbor on the same network involves no routing at all. If even that fails, the container isn't plugged in. Every container's network card has a host-side cable end named `veth<ID>i0`:

```bash
ip addr show veth<pihole-ID>i0     # exists, UP
brctl show vmbr0                   # lists what's plugged into the virtual switch
```

`brctl show vmbr0` listed **only Jellyfin's** veth. Pi-hole's virtual cable existed but had never been plugged into the switch.

**The fix:** a full stop and start (not a reboot), so Proxmox rebuilds the plumbing:

```bash
pct stop <pihole-ID>
pct start <pihole-ID>
brctl show vmbr0
```

Both veths listed, ping works. Rebooted again to make sure it stuck. It did.

> 💡 **A two-second habit.** After building any new container: `brctl show vmbr0`, check its veth is there. Cost me one evening once; never again.

## What went wrong, part 2: the day apps stopped loading

Months later: some apps wouldn't load at home (a sports app, email images, a phone game), others were fine. The provider had had an outage the day before. Very tempting to blame them.

> 🤖 **Murderbot Check.** Murderbot's first answer: "Pi-hole is overloaded, or it's a blocklist problem." Pi-hole's own dashboard said **0.4% CPU, 8% memory.** I sent the numbers back. The explanation had to change, and the second one was right.

**What was really happening:** Pi-hole has a **per-client rate limit**: one device making more than about 1,000 lookups in 60 seconds gets told to wait. That has nothing to do with CPU, so the dashboard looks calm while one chatty client is throttled, retries, and gets throttled again.

The Query Log told the story:

- A flood of lookups from **one address: the download container** from [chapter 6](06-the-automation-stack.md). Torrent clients "announce" to trackers constantly, and every announce starts with DNS lookups.
- Apple **Private Relay** lookups being blocked (expected; Pi-hole blocks them so Apple devices use normal DNS), adding retries to the pile.
- Lots of noisy, legitimate chatter: cameras, streaming-stick telemetry, TV "content recognition", game analytics.

**The fixes:**

1. **Two cores instead of one.** Pi-hole container → Resources → Cores → 2 (usually takes effect live).
2. **A second upstream DNS server** in Pi-hole: Settings → DNS → Upstream DNS Servers.
3. **Calmer downloads.** Capping the download client's upload speed made it re-announce less often.

It ran noticeably smoother right away. (You can also raise the rate limit in Settings → DNS → Rate limiting, or point the download container at different DNS. I preferred fixing the cause to raising a limit that exists for a reason.)

## Roads not taken

| Option | Standout | Weak spot |
|---|---|---|
| **Pi-hole** *(chosen)* | Huge community, great Query Log (which solved part 2) | Encrypted DNS to clients needs add-ons |
| AdGuard Home | Encrypted DNS and per-device rules built in | Smaller community |
| Technitium DNS | A full DNS server that also blocks | More to learn |
| Unbound (add-on) | Pi-hole asks the root servers itself; no DNS company sees your lookups | One more service. **The first upgrade I'd add; not done on CommandCenter** |
| NextDNS / Control D | Works on phones anywhere, no server | A company sees your lookups; free tiers cap usage |
| Browser extensions | Block ads DNS can't touch | Per device; nothing for TVs |
| A Raspberry Pi | DNS keeps working while the big server is down | One more box |

That last row matters more than it looks: when Pi-hole lives on your only server, rebooting the server takes the whole house's DNS with it. Remember that for Part 2.

## Skip ahead?

Standalone. Nothing later requires Pi-hole, but the diagnostic in part 1 is worth knowing no matter what you run.
