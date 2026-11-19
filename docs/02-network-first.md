# 2. Getting the network right before anything else

[← 1. Proxmox](01-old-pc-to-proxmox.md) · [Guide](../README.md) · Next: [3. Pi-hole →](03-pihole.md)

## What we're building and why

Nothing exciting. It's the chapter I most wish I'd read first. Every service in this guide is reached by an address on your home network. If those addresses drift, or the server clings to an address from an older network, everything downstream breaks in ways that look like something else entirely.

By the end:

- the server is on the right address range, with an address that won't change;
- every container has a stable address;
- you have the habit that would have saved me two evenings: **never trust a written-down address you haven't re-checked since the network last changed.**

## The story: the server that "couldn't be reached"

Partway through the build, the house switched internet providers. New gateway, new address range. Phones and laptops picked up new addresses on their own. The server's screen still proudly showed its **old** address, from the old range. From the laptop, the server was simply gone.

It looked like a network-segmentation problem (client isolation? separate VLANs?). It wasn't.

> 🤖 **Murderbot:** Your gateway is isolating wired devices from wireless ones.
> **Me:** It's one network. The app says one range.
> **Murderbot:** Then the server is lying.
> **Me:** …the server is lying.

## The build: finding and fixing the stale address

### Step 1: Confirm what the network actually is

One gateway; the server on a cable, phones and laptops on Wi-Fi. The provider's app showed a single address range. One network.

### Step 2: Ask the server what it thinks

On the server's own keyboard:

```bash
ip addr show
cat /etc/network/interfaces
```

`ip addr show` lists every network interface and its address. The one that matters on Proxmox is **`vmbr0`**, a virtual network switch that the physical port and every container plug into. It showed the old address. The config file showed why:

```
auto vmbr0
iface vmbr0 inet static
        address <old-address>/24
        gateway <old-gateway>
        bridge-ports <your-network-port>
        bridge-stp off
        bridge-fd 0
```

`inet static` = "use this exact address forever, whatever network you're plugged into." The installer wrote it on day one ([chapter 1](01-old-pc-to-proxmox.md), Step 4) and nobody touched it since.

### Step 3: Switch the host to DHCP

Let the gateway hand the server an address (DHCP), then tell the gateway to always hand it the **same** one.

```bash
nano /etc/network/interfaces
```

```
auto vmbr0
iface vmbr0 inet dhcp
        bridge-ports <your-network-port>
        bridge-stp off
        bridge-fd 0
```

Change `static` to `dhcp`, delete the `address` and `gateway` lines, leave the `bridge-` lines alone. Apply and check:

```bash
systemctl restart networking     # or, gentler: ifreload -a   (a reboot always works)
ip addr show vmbr0
```

New address, right range, marked `dynamic`. Web page, file share, everything came back. Copy: [`configs/interfaces-dhcp.example`](../configs/interfaces-dhcp.example).

> ⚠️ **Landmine: Proxmox remembers its address in a second place.** `/etc/hosts` has the host's address next to its name, and some Proxmox services look there. Update that line to the new address.

### Step 4: Make it permanent with a reservation

In your router's settings (on this house's gateway: the provider's phone app, "New IP Reservation"), tie the server's **MAC address** (the network port's hardware ID) to the address it just got:

```bash
ip link show vmbr0
```

The MAC is the `link/ether` value, six pairs like `aa:bb:cc:dd:ee:ff`. Now the server always gets the same address, and if the network changes again it simply asks the new gateway instead of insisting on a dead one.

### Step 5: Update everything that remembered the old address

The step everyone forgets:

- browser bookmarks to `https://<old-address>:8006`;
- the file-share bookmark (`smb://…` on a Mac, `\\…` on Windows); on a Mac, remove the old entry from Finder's sidebar;
- **your notes.** Mine kept the old address for weeks, and it bit twice more (see the Murderbot Check in [chapter 10](10-home-assistant.md)).

## Stable addresses for containers too

Every container gets its own address through the same `vmbr0`. Two ways:

- **`ip=dhcp`** plus a router reservation for the container's MAC. What this build used.
- **A static address** set in the container's Network settings, picked from outside your router's DHCP pool.

```bash
pct config <container-ID>                       # look at the net0: line: hwaddr= is the MAC
pct reboot <container-ID>
pct exec <container-ID> -- ip addr show eth0    # run one command inside the container
```

`pct` is the Proxmox Container Toolkit. After the host's fix, the `ip=dhcp` containers sorted themselves out with a reboot. Then I added a reservation for each.

> ⚠️ **Landmine: containers show up as bare MAC addresses.** Your router's device list says "unknown device", not "Pi-hole". Match them up with the `hwaddr=` from `pct config`.

## Two things in the logs you can ignore

- **A `169.254.x.x` address.** A device gave itself one because DHCP didn't answer in time. Once is a hiccup. Only chase it if it's constant.
- **"IPv6 Internet Not Connected."** Every install script tests IPv6. Nothing in this guide needs it.

## Roads not taken

| Approach | Pros | Cons |
|---|---|---|
| **DHCP + router reservation** *(chosen)* | One place for every address; survives network changes | Re-enter reservations if the router is replaced |
| Static on the device | Works if DHCP is down | Breaks silently when the range changes (exactly what happened here) |
| Static outside the DHCP pool | No clashes | Same weakness if the range changes |
| Names instead of numbers (Pi-hole local DNS) | Remember names; change an address in one place | Needs Pi-hole first ([chapter 3](03-pihole.md)) |

**The road I'd take next:** Pi-hole's **Local DNS Records** page. Bookmark `http://jellyfin.home.arpa:8096` instead of a number (`.home.arpa` is reserved for home networks), and an address change is one edit, not every bookmark in the house. (We didn't get to this on CommandCenter.)

## Skip ahead?

Don't. Even if your server is already running, run `ip addr show vmbr0` and `cat /etc/network/interfaces` now and make sure the address is the one you think it is, on the network you're actually on.
