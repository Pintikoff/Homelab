# Homelab

A self-hosted homelab running on a repurposed ThinkPad T530, built from scratch as a hands-on way to learn Docker, Linux networking, and self-hosting.

## Hardware

| | |
|---|---|
| Host | Lenovo ThinkPad T530 |
| CPU | Intel i7-3630QM (8 threads) @ 3.4GHz |
| GPU | Intel 3rd Gen Core Graphics + NVIDIA NVS 5400M |
| Memory | 12 GB RAM |
| Storage | ~470 GB (LVM) |
| OS | Ubuntu 24.04 LTS |

## Stack

All services run as isolated Docker containers, managed with Docker Compose — one folder per service, each with its own `docker-compose.yml`.

| Service | What it does | Docs |
|---|---|---|
| [Jellyfin](./jellyfin.md) | Self-hosted media server (movies, music) with a web UI and apps for most devices | [jellyfin.md](./jellyfin.md) |
| [Pi-hole](./pihole.md) | Network-wide DNS ad and tracker blocking, for every device on the network | [pihole.md](./pihole.md) |
| [Beszel](./beszel.md) | Lightweight server monitoring — CPU, RAM, disk, and per-container stats | [beszel.md](./beszel.md) |
| [Tailscale](./tailscale.md) | Mesh VPN for secure remote access to the homelab from anywhere, without port forwarding | [tailscale.md](./tailscale.md) |


Each linked file covers what the service is for, its Docker Compose setup, and the specific issues hit while deploying it.

## Network layout

- Home network behind a consumer router, ISP connection uses **DS-Lite** (IPv4 tunneled over IPv6, no dedicated public IPv4 address — CGNAT on the provider side)
- Pi-hole set as the DNS server for the whole network, over both IPv4 and IPv6, with local DNS records for `.home.local` service names
- Tailscale provides remote access without depending on the missing public IPv4 address, sidestepping the port-forwarding problem DS-Lite creates for a classic self-hosted VPN

## What this project covered

Starting point was "SSH access to the server, nothing else." Building it out surfaced a lot of adjacent systems knowledge along the way:

**Docker & containers**
- Images vs. containers, namespaces and cgroups as the isolation mechanism behind "containers," why containers need their own IP inside a Docker network
- Docker Compose as the declarative, one-folder-per-service alternative to long `docker run` commands
- Volume mounts (`host:container`) for persisting data and exposing host files/folders into a container
- Port mapping (`host:container`) and why it's needed for isolated container networks
- Docker's default bridge network vs. custom networks with fixed IPs, IPv6 support in Docker (off by default, needs explicit configuration), and subnet conflicts between them

**Networking fundamentals**
- DNS resolution, `/etc/hosts` vs. `/etc/resolv.conf`, `systemd-resolved` and why it competes with Pi-hole for port 53
- NAT and CGNAT, including how **DS-Lite** tunnels IPv4 over IPv6 via a provider-side AFTR server — and why that makes disabling IPv6 on the router take down the entire internet connection, not just IPv6 sites
- IPv6 addressing: link-local vs. global scope, ULA addresses (`fd00::/8`) as IPv6's equivalent of private IPv4 ranges, and SLAAC/Router Advertisement as a second, independent path (besides DHCPv6) for distributing IPv6 configuration to clients
- The difference between a router acting as a DNS proxy (forwarding queries under its own identity) versus handing out an upstream DNS server directly via DHCP — and why the former hides per-device visibility in Pi-hole's query log
- IP forwarding and why a VPN server needs it (acting as a relay for traffic addressed elsewhere) while a DNS resolver like Pi-hole doesn't (it's always the final endpoint of the request)
- Unix domain sockets vs. TCP ports as two different IPC mechanisms — same-host-only vs. network-reachable — and when each applies (Docker's own socket, Beszel's hub/agent link)

**Linux systems**
- LVM: volume groups, logical volumes, and extending an undersized root partition without a reinstall (`lvextend` + `resize2fs`)
- `netplan` for declarative network configuration, including pinning a static IPv6 ULA address
- File permissions and ownership (`chmod`/`chown`), and why Docker-created directories often end up root-owned
- `cron` for scheduled tasks, including the difference between a per-user crontab and drop-in files under `/etc/cron.d/`

**Reverse proxies**
- What a reverse proxy actually adds on top of DNS: DNS only resolves a name to an IP, a reverse proxy reads the HTTP `Host` header and routes to the correct internal port — the piece that lets multiple services share ports 80/443 under clean hostnames

## Status

Actively growing. Current focus areas: reverse proxy setup for clean internal hostnames, and hardening the server against the CIS Ubuntu Benchmark as a separate learning project.
