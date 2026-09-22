# Tailscale — Zero-Config Remote Access VPN

## What it is

Tailscale is a mesh VPN built on WireGuard. It creates a private virtual network ("tailnet") linking every device that joins with the same account, each getting a stable IP in the `100.x.x.x` range, reachable from anywhere without manually configuring encryption keys or opening any ports on the home router.

## Why use it instead of a self-hosted WireGuard server

A classic self-hosted WireGuard server (e.g. via `wg-easy`) requires forwarding a UDP port on the router to the home server. That's straightforward on a connection with its own public IPv4 address, but breaks down on connections using **CGNAT / DS-Lite**, where the ISP doesn't hand out a dedicated public IPv4 address at all — there's no port to forward, because the IP itself isn't exclusively the user's.

Tailscale sidesteps this with NAT traversal: devices behind NAT/CGNAT find each other using a lightweight coordination server, and in most cases then connect directly, peer-to-peer. No inbound port forwarding, no dealing with an ISP's networking quirks. The tradeoff is relying on Tailscale's coordination infrastructure for the initial handshake, versus a fully self-hosted WireGuard setup where nothing but the two endpoints is ever involved.

## Setup: server side

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

`tailscale up` prints a login link (`https://login.tailscale.com/a/xxxxx`). Opening it in a browser and authenticating (Google/GitHub/email) links the server to the account.

Check status:

```bash
tailscale status
```

Lists every device in the tailnet along with its `100.x.x.x` address.

## Setup: client side

Install the Tailscale app on any device to add (phone, laptop) and log in with the same account. It joins the same virtual network immediately, with no manual key exchange or router configuration.

## Using it

Once both the server and a client device are on the tailnet, any self-hosted service on the server becomes reachable from the client using the server's `100.x.x.x` address and the service's normal port, e.g.:

```
http://100.x.x.x:8096   # Jellyfin, from anywhere, over Tailscale
```

This works identically whether the client is on the same home network or on the other side of the world; Tailscale doesn't distinguish.

## Reinstalling / resetting

```bash
sudo tailscale down
sudo apt remove --purge tailscale
sudo rm -rf /var/lib/tailscale /etc/default/tailscaled
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Removing `/var/lib/tailscale` clears the device's local state (identity, keys), so it registers as a fresh device on next login rather than trying to resume a broken session.

## Relationship to a reverse proxy

Tailscale grants secure access to every port on the server directly; it doesn't provide clean URLs or route by hostname the way a reverse proxy (e.g. Nginx Proxy Manager, Caddy) does. The two solve different problems and can run side by side: Tailscale for secure remote access from outside the home network, a reverse proxy for tidy internal hostnames instead of `ip:port` addresses.
