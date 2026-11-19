# 10. Bringing in Home Assistant (and Zigbee)

[← 9. Immich](09-immich-and-the-backup-gap.md) · [Guide](../README.md) · Next: [Side quests and Part 2 →](coming-in-part-2.md)

## What we're building and why

**Home Assistant** is an open-source smart-home hub: one dashboard and one automation engine for thousands of devices, mostly local, instead of one cloud app per brand. This chapter builds it, survives the worst crash of the whole project, and adds a **Zigbee** radio.

## Before you start

Chapters 1 and 2. About 6 GB free RAM. Patience during the first boot. You'll see why.

## The decision: a VM, not a container

Home Assistant's main install is **Home Assistant OS** (HAOS), a whole operating system that expects to own a machine. On Proxmox that means a **VM**. The deciding question: **will you add a Zigbee or Z-Wave USB stick?** Handing USB to a VM is clean; to an LXC container it's a fight. I planned to: VM, now.

**Sizing:** 6 GB RAM (the script's default is 4; I left room for extras), 2 cores, 32 GB disk, port 8123.

## The build: first attempt

```bash
free -h; pct list; qm list      # qm = the tool for VMs; empty on a host with no VMs yet
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/vm/haos-vm.sh)"
```

Note `vm/`, not `ct/`. The wizard has **no Back button**. First try, wrong key on the CPU screen: `User exited script`. Nothing created; ran it again.

**Advanced settings:** stable HAOS; **CPU model: Host** (one machine, no cluster, so expose the real CPU); machine **q35**; 6 GB RAM.

And here's the mistake: the CPU selection landed on **KVM64** instead of Host. I noticed at the summary.

**The "harmless" fix:**

```bash
qm stop <haos-VM-ID>
qm set <haos-VM-ID> --cpu host
qm start <haos-VM-ID>
```

No errors. Clean. **That stop/start, about twenty minutes into the VM's very first boot, caused everything below.**

## What went wrong: the crash loop

Nothing at port 8123.

**Dead end 1: the wrong address, twice.**

> 🤖 **Murderbot Check.** Murderbot pointed me at the server's long-dead address from an old project note. I caught it: "That's the server's super-old IP." When the network has changed, old notes are guilty until proven innocent ([chapter 2](02-network-first.md)).

Finding the VM's real address: `arp -an` → `command not found` (not in a minimal install). The modern one:

```bash
ip neigh show      # match the VM's MAC (Proxmox → VM → Hardware → Network Device)
```

**Dead end 2: "it pings, so it's fine."** Ping answered, `qm status` said `running`. The network was fine; the problem was inside.

**Dead end 3: the healthy-looking red herring.** `curl -v http://<haos-IP>:4357` → `200 OK`, **Supervisor: Connected, Health: Healthy.** Port 4357 is the **Observer**, a status page for the Supervisor (the management layer). Home Assistant **Core**, the actual app on 8123, crashes separately, and the Observer doesn't report it.

> 🔍 **Diagnostic pattern: check the layer you actually care about.** A health page only knows about what it monitors.

**The real error.** VM console in the Proxmox page, at the `ha >` prompt: `ha core logs`. A traceback ending in:

```
AwesomeVersionCompareException: Can't compare <unknown> and <SimpleVer 0.50>
```

A version check choking on a version marker that's missing or corrupted.

**Wrong fix: downgrade** (`ha core update --version <previous-release>`). Same crash. **Same crash on two versions = not a version bug.** `ha core restart` → `Container homeassistant not found`; `ha core start` → crashed again ten seconds later.

Also along the way: `pct enter <haos-VM-ID>` → `Configuration file … does not exist`. Habit. **`pct` is for containers; VMs use `qm`**, and you get inside through the console.

> 🤖 **Murderbot:** We could keep debugging.
> **Me:** We've been at this all evening.
> **Murderbot:** We could keep debugging *faster*.

**Root cause:** stopping the VM mid-first-boot left a half-written saved state, most likely that version marker. Restarting, downgrading and recreating Core can't fix saved state.

**What fixed it: start over, get it right the first time.**

```bash
qm stop <haos-VM-ID>
qm destroy <haos-VM-ID>         # deletes the VM and its disk
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/vm/haos-vm.sh)"
```

**Host** on the CPU screen, then **hands off.** "Preparing Home Assistant… This may take 20 minutes or more." Then the onboarding page on 8123. Clean.

> 💡 **The lesson.** Every container in this guide shrugged off being stopped mid-setup. **VMs during their first boot don't.** Get the wizard right; if you get a setting wrong, destroying and rebuilding is usually faster than fixing in place. Restarting a fully set-up VM is fine.

### Onboarding and backups

Admin account, home location, units, time zone. One real advantage of a VM: its whole disk is one piece, so the nightly backup job ([chapter 1](01-old-pc-to-proxmox.md)) captures everything, with **none** of chapter 9's bind-mount gap. I checked the backup log to confirm the VM was in it.

## Three scares, all false alarms

- **"The VM is gone!"** Proxmox's mobile app showed an empty list. Desktop page: running. A mobile display quirk.
- **Memory at 93.8%.** Maybe real, maybe a memory bug fixed in a later point release. **Never re-checked on CommandCenter.** Watch Home Assistant's own Settings → System → Hardware graph over days, not one snapshot.
- **Safari won't load it.** Autocomplete dropped the `:8123`. Type the full `http://<haos-IP>:8123`.

## Adding a Zigbee radio

| Radio | Needs |
|---|---|
| Wi-Fi | Nothing extra; many devices phone home |
| **Zigbee** | **One** USB coordinator for the whole house |
| Z-Wave | One Z-Wave coordinator |
| Thread / Matter | A Thread border router / Matter support |

One Zigbee stick serves every Zigbee device; you don't need a hub per brand.

**The stick:** a Sonoff Zigbee 3.0 USB Dongle Plus (EFR32MG24 chip), with plain **Zigbee Coordinator** firmware for stability.

### Step 1: Flash it from a browser

The web flasher uses **WebSerial**, Chromium browsers only (Chrome, Edge, Brave). The stick plugs into the computer running the browser, not the server.

> ⚠️ **Landmine: no port, and the driver has no installer.** Windows didn't know the stick's CP210x chip, and the current driver download has **no `.exe`**. Unzip it → **Device Manager** → the stick under **Other devices** → **Update driver → Browse my computer** → the unzipped folder, "Include subfolders" → accept the warning. It shows under **Ports (COM & LPT)**. Reconnect the flasher.

### Step 2: Move it to the server

```bash
lsusb
# Bus 005 Device 002: ID 10c4:ea60 Silicon Labs CP210x UART Bridge
```

> 💡 **USB 2.0 ports are better for Zigbee.** USB 3.0 ports (often blue) put out interference in the same 2.4 GHz band. Use a black USB 2.0 port or a short extension cable.

### Step 3: Pass it through to the VM

```bash
qm set <haos-VM-ID> -usb0 host=10c4:ea60
```

By vendor:product ID, so it survives moving to another port.

### Step 4: ZHA

**ZHA** is built in (one moving part); **Zigbee2MQTT** needs an extra app and an MQTT broker (three). Fewer moving parts was the lesson of the crash: **ZHA.**

**Settings → Devices & services → Add integration → ZHA.** It failed: `Failed to connect`, offering `/dev/ttyS0` (a built-in serial port, not the stick). The clue: the VM couldn't see the stick yet. `qm config <haos-VM-ID> | grep usb` showed it was configured; the VM just hadn't picked up hardware added while running.

```bash
qm stop <haos-VM-ID>
qm start <haos-VM-ID>       # safe: a fully set-up VM, not a first boot
```

ZHA found it: radio **EZSP**, 115200 baud, flow control **Hardware**.

### Step 5: Create a network

Brand-new stick: **Create a network.** ZHA backs the network up itself (download it from the ZHA menu; that's what you'd restore onto a replacement stick).

**Honest status:** the coordinator is online. **No devices were paired on CommandCenter.** That's where Book 1 stops.

> 🤖 **Murderbot:** So you bought a radio for a house with no Zigbee devices.
> **Me:** It's a radio *for the future*.
> **Murderbot:** The future has no devices either.

## The mysterious "OpenWRT router"

Home Assistant discovered an "OpenWRT router". I don't own one. It came in over **UPnP**, and a common UPnP program's default self-description is literally "OpenWRT router", whatever the router runs. `ip neigh show | grep <mac-from-home-assistant>` matched the `.1` address with a `router` flag: the **provider's gateway**. Harmless.

## Roads not taken

- **Home Assistant Container (Docker):** lighter, fits the container pattern; no app store, and USB into a container is the fight.
- **Dedicated hardware** (a small box running HAOS): the smart home keeps running when you take the server down. (Hold that thought for Part 2.)
- **Zigbee2MQTT / deCONZ**, **Home Assistant's own Connect sticks**, **openHAB, Hubitat, Homey**: all real options.

## Skip ahead?

Standalone. This is where CommandCenter tops out. [What comes next →](coming-in-part-2.md)
