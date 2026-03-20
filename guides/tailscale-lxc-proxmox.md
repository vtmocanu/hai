# Tailscale in an LXC Container on Proxmox

## What is Tailscale?

[Tailscale](https://tailscale.com/) is a zero-config VPN built on WireGuard. It creates a secure mesh network between your devices without the hassle of traditional VPN setup — no port forwarding, no firewall rules, no certificates to manage.

**Key features:**

- **Mesh networking** — Devices connect directly to each other (peer-to-peer) when possible
- **Subnet routing** — Expose your entire LAN to your Tailscale network without installing Tailscale on every device
- **Exit nodes** — Route all your internet traffic through a specific device (like your home connection)
- **MagicDNS** — Access devices by hostname instead of IP
- **Free tier** — Generous free plan for personal use (up to 100 devices)

For homelabs, Tailscale is perfect: install it once, and you can access your entire network from anywhere — coffee shop, airport, phone on mobile data.

---

## Running Tailscale on Proxmox

Running Tailscale directly on the Proxmox host works, but it's not ideal from a security perspective. A dedicated LXC container isolates Tailscale from the hypervisor while still providing full network access to your homelab.

This tutorial walks through creating a minimal Debian LXC with Tailscale configured as both a **subnet router** (access your LAN from anywhere) and an **exit node** (route all traffic through your home network).

## The Challenge

Tailscale needs access to `/dev/net/tun` to create its network tunnel. Unprivileged LXC containers don't have this by default, so we need to explicitly grant it.

## Prerequisites

- Proxmox VE 7.0+ (uses cgroup2)
- A Tailscale account
- SSH access to your Proxmox host

## Step 1: Download a Container Template

SSH into your Proxmox host and download a Debian template:

```bash
# List available templates
pveam available --section system | grep debian

# Download Debian 13 (or 12 if you prefer stable)
pveam download local debian-13-standard_13.1-2_amd64.tar.zst
```

## Step 2: Create the LXC Container

```bash
pct create 121 local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst \
  --hostname tailscale \
  --memory 512 \
  --cores 1 \
  --rootfs local-lvm:4 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 1 \
  --features nesting=1 \
  --start 0
```

Adjust the CTID (121), storage (`local-lvm`), and network bridge (`vmbr0`) for your environment.

## Step 3: Enable TUN Device Access

This is the key step. Add these lines to the container config:

```bash
cat >> /etc/pve/lxc/121.conf << 'EOF'
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
EOF
```

Verify the config looks correct:

```bash
cat /etc/pve/lxc/121.conf
```

You should see the two `lxc.*` lines at the bottom.

## Step 4: Ensure TUN Device Exists on Host

The TUN device should already exist, but verify:

```bash
ls -la /dev/net/tun
```

If it doesn't exist:

```bash
mkdir -p /dev/net
mknod /dev/net/tun c 10 200
chmod 666 /dev/net/tun
```

## Step 5: Start Container and Install Tailscale

```bash
# Start the container
pct start 121

# Install curl (not included in minimal Debian)
pct exec 121 -- apt-get update
pct exec 121 -- apt-get install -y curl

# Install Tailscale
pct exec 121 -- bash -c 'curl -fsSL https://tailscale.com/install.sh | sh'
```

## Step 6: Enable IP Forwarding

For subnet routing and exit node functionality, enable IP forwarding:

```bash
pct exec 121 -- bash -c 'cat >> /etc/sysctl.conf << EOF
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF'

pct exec 121 -- sysctl -p
```

## Step 7: Connect Tailscale

Now bring up Tailscale with subnet routing and exit node enabled:

```bash
pct exec 121 -- tailscale up \
  --advertise-routes=192.168.1.0/24 \
  --advertise-exit-node
```

Replace `192.168.1.0/24` with your actual LAN subnet.

This outputs an authentication URL. Open it in your browser to authorize the device.

## Step 8: Approve in Admin Console

After authenticating, go to the [Tailscale admin console](https://login.tailscale.com/admin/machines) and:

1. Find your new `tailscale` machine
2. Click the three dots menu → **Edit route settings**
3. Enable the subnet route (`192.168.1.0/24`)
4. Enable **Use as exit node**
5. Optionally: **Disable key expiry** for a server that should stay connected permanently

## Result

You now have a lightweight LXC container (~512MB RAM) that:

- Provides access to your home LAN from anywhere via Tailscale
- Can act as an exit node to route all traffic through your home network
- Is isolated from the Proxmox host for better security
- Auto-starts with Proxmox (enable via: `pct set 121 --onboot 1`)

## Troubleshooting

**Tailscale won't start / TUN errors:**
- Verify the `lxc.cgroup2.devices.allow` line is in the container config
- Check that `/dev/net/tun` exists on the host
- Restart the container after config changes

**Can't reach LAN through subnet route:**
- Verify IP forwarding is enabled (`sysctl net.ipv4.ip_forward`)
- Check that the route is approved in Tailscale admin console
- Ensure your LAN subnet matches what you advertised

**Auth link expired:**
- Run `tailscale up` again to get a fresh URL
- If stuck, reset state: `systemctl stop tailscaled && rm /var/lib/tailscale/tailscaled.state && systemctl start tailscaled`

