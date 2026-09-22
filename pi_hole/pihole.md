# Pi-hole — Network-Wide DNS Ad Blocking

## What it is

Pi-hole is a DNS sinkhole: it sits between every device on the network and the wider internet's DNS resolvers, checks each requested domain against blocklists of known ad and tracking servers, and refuses to resolve the ones on the list. Because it works at the DNS level, it blocks ads and trackers across every device on the network — phones, smart TVs, consoles — not just a browser with an extension installed.

## How it works

1. A device wants to load a domain, e.g. `doubleclick.net`. Before it can connect, it needs that domain's IP address, so it sends a DNS query.
2. Instead of going straight to the ISP's or a public DNS server, that query goes to Pi-hole first.
3. Pi-hole checks the domain against its blocklists (large text files of known ad/tracking domains).
   - **Blocked**: Pi-hole replies with `0.0.0.0` or `NXDOMAIN` — the device never even attempts to load the ad.
   - **Not blocked**: Pi-hole forwards the query to an upstream resolver (e.g. Cloudflare or Quad9), gets the real answer, and passes it back.

Pi-hole never touches the actual web traffic that follows, only the DNS lookup that precedes it — once a device knows the site's real IP, the connection to that site goes directly, bypassing Pi-hole entirely.

## Docker Compose setup

Note: Pi-hole must bind to port 53, and Ubuntu's `systemd-resolved` occupies that port by default. Free it first:

```bash
sudo nano /etc/systemd/resolved.conf
# set: DNSStubListener=no

sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
```

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

```bash
docker compose up -d
```

- Environment variables use the `FTLCONF_*` naming scheme introduced in Pi-hole v6; older guides referencing `WEBPASSWORD` or `DNSSEC` are for v5 and no longer apply.
- `FTLCONF_dns_listeningMode: ALL` is required in a Docker setup. Without it Pi-hole only answers queries that look "local," and requests arriving through the Docker port mapping technically appear to originate from outside the container.
- The web UI is mapped to `8080` rather than `80` to leave port 80 free for other services (e.g. a reverse proxy) later.

Access the dashboard at `http://<server-ip>:8080/admin`.

## Making Pi-hole the network's DNS server

Setting Pi-hole's IP as the DNS server in a router's WAN/internet settings only changes what the router itself uses — it doesn't change what DHCP hands out to other devices. Many consumer routers keep acting as a DNS proxy by default: they forward client queries to Pi-hole but do so under their own identity, so Pi-hole's Query Log shows every request as coming from the router, not the actual device.

To get real per-device visibility, look for a separate DHCP-side setting (often under LAN/local network settings, sometimes labeled "local DNS server") and point that at Pi-hole's IP directly. Do the same for the IPv6 equivalent if the network uses IPv6, and make sure any "advertise DNS via Router Advertisement" option is enabled, since IPv6 clients often get their DNS server via RA rather than DHCPv6 alone.

## Caveats

- If the Pi-hole host goes offline, DNS breaks for the whole network unless the router is configured with a public DNS server as automatic failover.
- IPv6 needs the same treatment as IPv4: modern OSes prefer IPv6 when available, so if only the IPv4 DNS server is changed, devices may silently keep resolving over IPv6 through the original resolver, bypassing the filter entirely.
