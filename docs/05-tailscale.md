# 5. Reaching your server from anywhere (Tailscale)

[← 4. Jellyfin](04-jellyfin.md) · [Guide](../README.md) · Next: [6. The automation stack →](06-the-automation-stack.md)

## What we're building and why

Everything so far only works at home. This chapter makes it work from a phone on cellular, from anywhere, **without opening a single port on your router.**

The usual way is **port forwarding**: "send traffic on port 8096 to Jellyfin". It works, and it also puts that service on the open internet, where scanners find it within hours. A poor first move. (And this house's gateway barely offers it anyway.)

**Tailscale** builds a private, encrypted network (a **tailnet**) between your own devices, wherever they are. It's built on WireGuard and handles the hard parts for you: keys, getting through home routers, sign-in. Phone and server both on the tailnet = the phone reaches the server as if it were home.

## Before you start

A free Tailscale account. The free Personal plan covers this whole guide.

## Four ways to use Tailscale

| Mode | Who can reach it | Used in |
|---|---|---|
| **Device on the tailnet** | Only your tailnet | This chapter (Jellyfin, the host) |
| **`tailscale serve`** | Only your tailnet, with a real HTTPS name and certificate | [Chapter 8](08-nextcloud.md) (Nextcloud) |
| **`tailscale funnel`** | **Anyone on the internet** with the link | [Chapter 9](09-immich-and-the-backup-gap.md) (sharing photos) |
| **Subnet router** | Your tailnet reaches your whole home network | Considered, not done (Roads not taken) |

Start narrow, one service at a time, widen only with a reason.

## The build: Tailscale inside the Jellyfin container

### Step 1: Install it

```bash
pct enter <jellyfin-ID>
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

`tailscale up` should print a login link. Here it printed:

```
failed to connect to local tailscaled; it doesn't appear to be running
```

`tailscaled` is the background service that does the real work; `tailscale` is just its remote control.

### Step 2: The service won't stay up

```bash
systemctl start tailscaled
systemctl enable tailscaled
systemctl status tailscaled          # "Start request repeated too quickly": it's crash-looping
journalctl -u tailscaled -n 30 --no-pager
```

`journalctl -u` reads one service's log. The key lines:

```
is CONFIG_TUN enabled in your kernel? modprobe tun failed ...
CreateTUN("tailscale0") failed; /dev/net/tun does not exist
```

> 💡 **Why.** A VPN needs a **TUN device**, a virtual network card programs can create. Unprivileged LXC containers don't get `/dev/net/tun` by default. It's a security boundary. The host has it; the container doesn't.

### Step 3: Give the container a TUN device

`exit` the container. On the **host**:

```bash
pct set <jellyfin-ID> -features nesting=1,keyctl=1

cat >> /etc/pve/lxc/<jellyfin-ID>.conf << 'EOF'
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
EOF

pct reboot <jellyfin-ID>
```

`>>` **appends** (a single `>` would overwrite the whole config, so don't). Line 1 allows the TUN device (major/minor `10:200`); line 2 makes the host's `/dev/net/tun` appear inside. Copy: [`configs/lxc-tun.conf.example`](../configs/lxc-tun.conf.example).

### Step 4: Again

```bash
pct enter <jellyfin-ID>
ls -la /dev/net/tun
systemctl restart tailscaled
tailscale up
```

Device exists, service stays up, login URL printed.

> ⚠️ **Landmine: that URL is a live key.** Open it where you're signed in to Tailscale and approve. Never paste it into a chat, note or screenshot; until it's used, anyone holding it can attach the device to *their* account.

> 🤖 **Murderbot:** Paste me the link and I'll check it's the right format.
> **Me:** You just told me never to paste it anywhere.
> **Murderbot:** I am not "anywhere".
> **Me:** You are exactly anywhere.

### Step 5: Test from outside

```bash
tailscale ip -4     # prints 100.x.y.z
```

On your phone: Tailscale app, same account, **Wi-Fi off**, open `http://100.x.y.z:8096`. Jellyfin, on cellular, nothing exposed to the internet.

## The build: Tailscale on the Proxmox host

To manage the server itself from anywhere. On the host, no TUN fix needed:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
tailscale ip -4
```

`https://100.x.y.z:8006` now opens the Proxmox page from anywhere on the tailnet. Checked at home and on cellular.

**SSH keys while you're at it.** On your laptop:

```bash
ssh-keygen -t ed25519 -C "laptop-to-server"
```

That makes a private key (never share it) and a `.pub` public key. On the server:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo "<contents-of-your-.pub-file>" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Now `ssh root@100.x.y.z` lands straight in the server's shell, from anywhere.

## Two things Tailscale does *not* do

**It doesn't harden anything.** I asked about putting Tailscale on the automation containers "as hardening". **Tailscale adds a way in; it doesn't remove one.** They stayed reachable from the home network as before. Real protection from the internet = no ports forwarded on the router. With no need for remote access to those tools, I skipped it there and checked for stray port forwards.

**It doesn't help people who aren't on your tailnet.** Family won't install a VPN to see photos. [Chapter 9](09-immich-and-the-backup-gap.md) handles that with `funnel` and per-album links.

## Roads not taken

- **Headscale:** self-hosted coordinator for the Tailscale apps; no company in the loop, but you run the piece everything depends on.
- **NetBird / ZeroTier:** other mesh VPNs; NetBird is fully self-hostable.
- **Plain WireGuard (wg-easy):** no middleman, one forwarded port, every device a manual step.
- **Cloudflare Tunnel:** great for public web apps on your own domain; its terms rule out heavy video, so not for Jellyfin.
- **Port forward + reverse proxy:** how the public internet works. Worth learning eventually. Not where this guide starts.
- **Subnet router** (`tailscale up --advertise-routes=192.168.1.0/24`): the whole home network reachable over the tailnet. Convenient, broad. Not done on the one box.

## Skip ahead?

Optional if you only use things at home. [Chapter 8](08-nextcloud.md) and [chapter 9](09-immich-and-the-backup-gap.md) build on it; read at least the TUN fix and the mode table.
