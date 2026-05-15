<div align="center">

# 🛡️ Pi-hole as a Full Home Router
### Raspberry Pi 3B+ · Single Ethernet · 802.1Q VLAN Trunk

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
![Platform](https://img.shields.io/badge/platform-Raspberry%20Pi%203B%2B-red)
![OS](https://img.shields.io/badge/OS-Raspberry%20Pi%20OS%20Lite%2064--bit-blue)
![DNS](https://img.shields.io/badge/DNS-Pi--hole%20%2B%20Unbound-brightgreen)
![Routing](https://img.shields.io/badge/routing-iptables%20NAT-orange)

</div>

---

> Convert a Raspberry Pi 3B+ into a full network router with DNS-level ad blocking, recursive DNS resolution, NAT, and DHCP — using a **single ethernet port** and **802.1Q VLAN trunking** over a managed switch. No second NIC needed.

---

## 📖 Full Interactive Guide

👉 **[View the styled guide on GitHub Pages](https://mattymomo1993.github.io/pihole-router-guide)**

---

## ⚙️ Stack

| Component | Role |
|---|---|
| Raspberry Pi 3B+ | Router · DNS · firewall · DHCP |
| TP-Link TL-SG108E (~$30) | 802.1Q VLAN trunk · device access |
| Pi-hole | Network-wide ad & tracker blocking |
| Unbound | Recursive DNS · no Google · no Cloudflare |
| iptables | NAT · packet forwarding · firewall |
| vlan (8021q) | VLAN sub-interfaces on single eth0 |
| dnsmasq (Pi-hole) | DHCP server for all LAN devices |
| Raspberry Pi OS Lite 64-bit | Base headless OS |
| UPS (APC BE600M1 ~$80) | Whole-network power backup |

---

## 🗺️ Network Topology

<pre>
Internet
   │
Modem (WAN IP from ISP)
   │
TL-SG108E Managed Switch (802.1Q)
   ├─ Port 1 ── VLAN 10 tagged    ← modem (WAN)
   ├─ Port 2 ── VLAN 10+20 tagged ← Raspberry Pi (trunk)
   └─ Ports 3-8 VLAN 20 untagged  ← LAN devices
         │
   Raspberry Pi 3B+
   ├─ eth0.10 = WAN  (DHCP from modem)
   ├─ eth0.20 = LAN  (192.168.2.1 gateway)
   ├─ Pi-hole + Unbound (DNS)
   ├─ iptables NAT (routing)
   └─ dnsmasq (DHCP)
         │
   WiFi Access Point (bridge mode)
   ├─ DHCP disabled
   ├─ Static IP 192.168.2.2
   └─ DNS 192.168.2.1
         │
   📱 Phone · 💻 Laptop · 📺 TV · 🖥️ Desktop · 🎮 Console
</pre>

---

## 📋 Steps

<details>
<summary><b>Step 01 — Flash Raspberry Pi OS Lite (64-bit)</b></summary>
<br>

Use **Raspberry Pi Imager**. Select **Raspberry Pi OS Lite (64-bit)**. In advanced settings (⚙):

<pre>
hostname:  pihole
SSH:       enabled
username:  pi
password:  [strong password]
WiFi:      do NOT configure — ethernet only
</pre>

> ⚠️ Do not use Raspberry Pi OS Desktop — wastes RAM the router stack needs.

</details>

---

<details>
<summary><b>Step 02 — First Boot & System Update</b></summary>
<br>

```bash
# SSH in
ssh pi@pihole.local

# Update all packages
sudo apt update && sudo apt full-upgrade -y

# Install required tools
sudo apt install -y vlan iptables iptables-persistent curl

# Load 802.1Q kernel module
sudo modprobe 8021q

# Persist on boot
echo "8021q" | sudo tee -a /etc/modules
```

</details>

---

<details>
<summary><b>Step 03 — Configure VLAN Interfaces</b></summary>
<br>

> ℹ️ One physical cable carries both VLANs via 802.1Q tagging. No second NIC needed.

`/etc/network/interfaces`:

```
# Loopback
auto lo
iface lo inet loopback

# Physical port — no IP, carries tagged frames
auto eth0
iface eth0 inet manual

# WAN — VLAN 10 — gets IP from modem
auto eth0.10
iface eth0.10 inet dhcp
  vlan-raw-device eth0

# LAN — VLAN 20 — Pi is the gateway
auto eth0.20
iface eth0.20 inet static
  address 192.168.2.1
  netmask 255.255.255.0
  vlan-raw-device eth0
```

```bash
sudo systemctl restart networking
```

</details>

---

<details>
<summary><b>Step 04 — Enable IP Forwarding + NAT</b></summary>
<br>

```bash
# Enable IP forwarding — persist across reboots
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# NAT — masquerade LAN traffic out WAN
sudo iptables -t nat -A POSTROUTING -o eth0.10 -j MASQUERADE

# Allow return traffic from WAN to LAN
sudo iptables -A FORWARD -i eth0.10 -o eth0.20 \
  -m state --state RELATED,ESTABLISHED -j ACCEPT

# Allow LAN to initiate connections out
sudo iptables -A FORWARD -i eth0.20 -o eth0.10 -j ACCEPT

# Save rules — survives reboot
sudo netfilter-persistent save
```

> ⚠️ Without `net.ipv4.ip_forward=1` the Pi receives packets but drops them. This single line separates a router from a regular Linux host.

</details>

---

<details>
<summary><b>Step 05 — Install Pi-hole</b></summary>
<br>

```bash
curl -sSL https://install.pi-hole.net | bash
```

Installer selections:

<pre>
Interface:     eth0.20           ← LAN only
Upstream DNS:  127.0.0.1#5335   ← Unbound (next step)
Block lists:   default
Admin web UI:  Yes
</pre>

Enable DHCP:

```bash
sudo pihole -a enabledhcp 192.168.2.10 192.168.2.254 192.168.2.1 24 pihole.local
```

> ℹ️ DHCP range 192.168.2.10–254 · Gateway 192.168.2.1 · Domain pihole.local

</details>

---

<details>
<summary><b>Step 06 — Install Unbound (Recursive DNS)</b></summary>
<br>

```bash
sudo apt install -y unbound
```

`/etc/unbound/unbound.conf.d/pi-hole.conf`:

```
server:
    verbosity: 0
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    do-ip6: no
    harden-glue: yes
    harden-dnssec-stripped: yes
    hide-identity: yes
    hide-version: yes
    cache-max-ttl: 86400
    prefetch: yes
    num-threads: 1
    root-hints: "/var/lib/unbound/root.hints"
```

```bash
# Download root hints
sudo wget -O /var/lib/unbound/root.hints \
  https://www.internic.net/domain/named.root

sudo systemctl restart unbound

# Verify
dig google.com @127.0.0.1 -p 5335
```

Then in Pi-hole web UI: **Settings → DNS** → uncheck all → set custom upstream to `127.0.0.1#5335`

</details>

---

<details>
<summary><b>Step 07 — Configure Managed Switch VLANs</b></summary>
<br>

> ℹ️ Configure switch BEFORE connecting modem or Pi. Access web UI at 192.168.0.1 with laptop directly connected.

<pre>
Port 1:   VLAN 10 tagged    ← modem (WAN)
Port 2:   VLAN 10+20 tagged ← Raspberry Pi (trunk)
Port 3:   VLAN 20 untagged  ← WiFi access point
Port 4:   VLAN 20 untagged  ← wired device
Port 5:   VLAN 20 untagged  ← wired device
Port 6:   VLAN 20 untagged  ← wired device
Port 7:   VLAN 20 untagged  ← wired device
Port 8:   VLAN 20 untagged  ← wired device
</pre>

> ⚠️ Port 2 (Pi trunk) MUST be tagged for BOTH VLANs. Device ports are untagged VLAN 20 only.

</details>

---

<details>
<summary><b>Step 08 — Configure WiFi Access Point</b></summary>
<br>

Any router brand works. Set it to bridge/AP mode:

<pre>
Mode:         Bridge Mode (or AP mode)
DHCP Server:  Disabled
Static IP:    192.168.2.2
Gateway:      192.168.2.1
DNS:          192.168.2.1
WiFi SSID:    your existing name
WiFi pass:    unchanged
</pre>

</details>

---

<details>
<summary><b>Step 09 — Verify Everything Works</b></summary>
<br>

```bash
# From Pi
ip addr show eth0.10       # WAN IP from modem
ip addr show eth0.20       # LAN IP 192.168.2.1
ip route show              # default via eth0.10
ping -c 3 8.8.8.8          # internet reachable
dig google.com @127.0.0.1 -p 5335   # Unbound working
dig doubleclick.net @192.168.2.1    # returns 0.0.0.0 = blocked

# Pi-hole dashboard
http://192.168.2.1/admin
```

</details>

---

<details>
<summary><b>Step 10 — Harden & Maintain</b></summary>
<br>

```bash
# Hardware watchdog — auto-reboot on freeze
sudo apt install -y watchdog
echo "bcm2835_wdt" | sudo tee -a /etc/modules
sudo systemctl enable watchdog --now

# Unattended security upgrades
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades

# Update Pi-hole
pihole -up

# Update blocklists
pihole -g

# Schedule weekly gravity update — 3am Sunday
echo "0 3 * * 0 root pihole -g" | sudo tee /etc/cron.d/pihole-gravity
```

> ⚠️ Plug modem + switch + Pi into a UPS. Watchdog handles software freezes. UPS handles power loss. You need both.

</details>

---

## License

MIT — use freely, modify freely, no attribution required.
