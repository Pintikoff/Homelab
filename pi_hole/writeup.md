# Setting Up Pi-hole in a Home Homelab

A walkthrough of the full journey: from first spinning up the container to full DNS-level ad blocking across IPv4 and IPv6 for every device in the house.

> Addresses, hostnames, and router-specific menu paths below are placeholders — swap in your own.

## Stack

- Server: old laptop, Ubuntu 24.04 LTS, quad-core CPU, 12 GB RAM
- Docker + Docker Compose, one service per folder with its own `docker-compose.yml`
- Consumer router with an ISP using DS-Lite (IPv4 tunneled over IPv6)

---

## 1. Installing Pi-hole

### Problem: port 53 already in use
Ubuntu holds port 53 by default via `systemd-resolved` (the DNS stub listener). Pi-hole can't bind to that port while it's occupied.

**Fix:**
```bash
sudo nano /etc/systemd/resolved.conf
# DNSStubListener=no

sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
```

A more radical alternative is disabling `systemd-resolved` entirely (`systemctl disable --now systemd-resolved`) and writing `/etc/resolv.conf` by hand. In that case, make sure `/etc/resolv.conf` ends up as a plain file, not a symlink pointing at `/run/systemd/resolve/stub-resolv.conf` (that file lives in tmpfs and disappears on reboot).

### Problem: `sudo: unable to resolve host`
After the DNS changes, `sudo` started failing to resolve the hostname on every invocation. Cause: a mismatch between the login username and the actual system hostname in `/etc/hosts` (e.g. `my_user` vs. `myuser`). This had been silently masked before — the DNS lookup apparently succeeded by inertia — and the change to DNS made the failure explicit.

**Fix:** align `/etc/hosts` with the real hostname (check with the `hostname` command):
```
127.0.1.1 <your-actual-hostname>
```

### Base `docker-compose.yml`
```yaml
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8080:80/tcp"
    environment:
      TZ: "Etc/UTC"
      FTLCONF_webserver_api_password: "changeme"
      FTLCONF_dns_listeningMode: "ALL"
      FTLCONF_dns_dnssec: "true"
    volumes:
      - "./etc-pihole:/etc/pihole"
    restart: unless-stopped
```

Key points:
- **v6 environment variables** (`FTLCONF_*`) replaced the old v5 ones (`WEBPASSWORD`, `DNSSEC`) — worth double-checking against current docs rather than copying old configs found online.
- **`FTLCONF_dns_listeningMode: ALL`** is required in a Docker setup: without it, Pi-hole only answers "local" queries, and requests arriving via port-forwarding technically look like they come from outside the container.
- The web UI is deliberately mapped to `8080`, not `80` — to keep port 80 free for future services (e.g. a reverse proxy for another app).

---

## 2. DNS only covered the server, not the whole house

The router kept handing its own address out to DHCP clients by default. Fixed in the router's admin panel, under its internet/WAN DNS settings: switched from "use ISP-assigned DNS" to a custom server, with Pi-hole's LAN IP set as preferred and a public resolver (`1.1.1.1`) as failover.

Important to lock in a **static IP** for the server ahead of time in the router's DHCP reservation settings — otherwise the server's IP changing later would break DNS for the entire house.

---

## 3. IPv6 was bypassing Pi-hole entirely

### Symptom
On Windows, `ipconfig /all` showed DNS Servers as long IPv6 addresses only (not Pi-hole). Windows prefers IPv6 when available, so all DNS traffic was routing around the filter.

### Root cause
IPv6 isn't distributed only through DHCPv6 — there's a separate mechanism, **SLAAC** (Stateless Address Autoconfiguration), driven by **Router Advertisement (RA)**. Disabling only DHCPv6 on the router doesn't remove IPv6 addresses from clients — RA operates independently.

### First attempt: disable IPv6 on the router
Broke the internet **entirely**. Cause: the ISP uses **DS-Lite** — IPv4 access physically tunnels over IPv6 to the provider's AFTR server, which performs CGNAT (Carrier-Grade NAT — address translation at the ISP level, not just the home router). Without IPv6, there's no tunnel — no IPv4 internet access either. IPv6 can't be turned off on DS-Lite connections.

### Final fix: give Pi-hole its own stable IPv6 address

**Step 1 — pin a ULA address on the server.**
The server's automatic IPv6 addresses (SLAAC + privacy extensions) are temporary (`dynamic`, `valid_lft` of a few hours). What's needed is a permanent **ULA** (Unique Local Address, `fd00::/8` — IPv6's equivalent of `192.168.x.x`, not routable to the internet).

Generate a random ULA (any `fdXX:XXXX:XXXX::/64` prefix works, just make one up per RFC 4193) and pin it via netplan:
```yaml
network:
  version: 2
  wifis:
    wlan0:
      dhcp4: false
      addresses:
        - 192.168.1.10/24          # server's static LAN IPv4
        - fd00:1234:5678::1/64     # server's pinned ULA
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [1.1.1.1]
      access-points:
        "your-wifi-ssid":
          password: "your-wifi-password"
```
```bash
sudo netplan apply
```

**Step 2 — enable IPv6 in Docker.**
Docker doesn't create IPv6 networks for containers by default.
```json
// /etc/docker/daemon.json
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00:aaaa:bbbb::/64"
}
```
```bash
sudo systemctl restart docker
```

**Step 3 — custom Docker network with a fixed IPv6 for Pi-hole.**
```yaml
services:
  pihole:
    # ...
    networks:
      pihole_net:
        ipv6_address: fd00:cccc:dddd::2

networks:
  pihole_net:
    enable_ipv6: true
    ipam:
      config:
        - subnet: 172.20.0.0/24
        - subnet: fd00:cccc:dddd::/64
```

**Gotcha:** the first attempt reused the *same* IPv6 range for both `daemon.json` and Pi-hole's custom network — conflict (`Pool overlaps with other one on this address space`), because Docker's default bridge network had already claimed that range on daemon restart. Fix: use different ranges for the default network and any custom one.

**Step 4 — point the router at it.**
In the router's WAN DNS settings, under the IPv6 tab: switched from "ISP-assigned" to a custom DNSv6 server, using the host's ULA as preferred and a public IPv6 resolver (e.g. `2606:4700:4700::1111`) as failover.

---

## 4. Query Log only ever showed the router as the client

### Symptom
After fixing IPv6, every entry in Query Log was attributed to the router's own LAN IP, not the actual devices — impossible to tell which device requested what.

### Root cause
Many consumer routers act as a DNS intermediary by default: the WAN-side DNS setting controls who the router asks **on its own behalf**, but DHCP still hands out **the router itself** as the DNS server to clients — which then forwards to Pi-hole under its own identity.

### Fix
Look for a separate setting — usually under LAN/local network settings rather than WAN/internet settings — for the DNS server handed out **directly via DHCP** (sometimes labeled "local DNS server"). Point it at Pi-hole's IP. Do the same for the IPv6 equivalent, and make sure any "advertise DNS via Router Advertisement (RFC 5006)" option is enabled — otherwise the new server won't reach clients via RA.

After changing the setting, clients sometimes need to force-renew their lease:
```powershell
ipconfig /release6
ipconfig /renew6
ipconfig /flushdns
```
If that doesn't help, rebooting the router forces a fresh Router Advertisement broadcast.

---

## Final architecture

```
Device → (DHCP/RA from router announces: DNS = Pi-hole)
       → Pi-hole (server's static LAN IPv4 / pinned ULA)
            ├─ domain on blocklist → 0.0.0.0 / NXDOMAIN
            └─ clean domain → forwarded to upstream (Cloudflare/Quad9)
```

## Left for later

- **Unbound** as a self-hosted recursive resolver instead of a public upstream (Quad9/Cloudflare) — removes reliance on a third-party DNS provider and improves privacy.
- A dashboard widget showing live Pi-hole stats.
- Expanding an undersized LVM root partition that was discovered along the way (`lvextend` + `resize2fs`).
