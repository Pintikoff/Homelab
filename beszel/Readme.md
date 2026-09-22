# Beszel — Lightweight Server Monitoring

## What it is

Beszel is a lightweight, self-hosted monitoring tool for tracking CPU, memory, disk, and network usage across servers, including per-container stats for Docker workloads. It uses a **hub + agent** architecture.

## Why use it

- One dashboard for CPU/RAM/disk/network history and live stats, instead of SSHing in and running `htop`/`df` by hand
- Minimal resource footprint, well suited to older or resource-constrained hardware
- Scales naturally: adding a second server later just means deploying one more agent and pointing it at the existing hub, no hub-side reconfiguration
- Can break down resource usage per Docker container, not just for the host as a whole

## Architecture

- **Hub** — the web dashboard. Stores historical metrics (in an embedded database) and displays graphs for every connected system.
- **Agent** — a small collector deployed on each machine you want to monitor. Measures local CPU/RAM/disk/network and reports back to the hub.

On a single-server homelab, the hub and agent run as two separate containers on the same machine, communicating through a Unix socket — a local file both containers share, rather than talking over the network.

## Docker Compose setup (hub + agent on the same host)

```yaml
services:
  beszel:
    image: henrygd/beszel:latest
    container_name: beszel
    restart: unless-stopped
    ports:
      - "8090:8090"
    volumes:
      - ./beszel_data:/beszel_data
      - ./beszel_socket:/beszel_socket
    environment:
      APP_URL: "http://<server-ip>:8090"

  beszel-agent:
    image: henrygd/beszel-agent:latest
    container_name: beszel-agent
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./beszel_socket:/beszel_socket
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      LISTEN: "/beszel_socket/beszel.sock"
      KEY: ""
      TOKEN: ""
```

```bash
docker compose up -d
```

Notes on the config:
- `network_mode: host` on the agent lets it see the host's real network interfaces for accurate network stats, rather than only its own isolated container network.
- Mounting `/var/run/docker.sock` (read-only) lets the agent query the Docker daemon for per-container stats. This is a meaningful trust boundary — only mount it if per-container breakdown matters to you; the hub still shows host-level CPU/RAM/disk without it.
- `beszel_socket` is a shared folder mounted into both containers. The agent creates a Unix socket file there, and the hub reads from the same file. This only works because both containers run on the same host; monitoring a remote server requires a network-based setup instead (see below).

## First-time setup (two-step)

1. Open `http://<server-ip>:8090`, create the first admin user, log in.
2. Click **Add System**. For a local agent, set the connection method to the Unix socket path (`/beszel_socket/beszel.sock`) rather than an IP/port. This generates a **Public Key** and **Token**.
3. Copy the Public Key and Token into the agent's `KEY` and `TOKEN` environment variables, then recreate the container:
   ```bash
   docker compose up -d
   ```
4. Back in the UI, confirm **Add System**. The agent should now report live stats.

## Monitoring a second, separate server

For a remote agent (a different physical machine), the Unix socket isn't reachable, so the agent connects over the network instead. Point it at the hub with `HUB_URL`, and give it a real network port instead of a socket path:

```yaml
environment:
  LISTEN: "45876"
  HUB_URL: "http://<hub-ip>:8090"
  KEY: "..."
  TOKEN: "..."
```

The same port number can be reused across multiple remote servers without conflict, since a network address is the combination of IP *and* port — identical ports on different hosts don't collide.
