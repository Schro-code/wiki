---
domain: system-design
tags: [system-design]
sources: []
updated: 2026-09-01
---

# IP Addresses

## What it is

An IP address is a unique identifier assigned to every device on a network. IP stands for Internet Protocol — the set of rules governing how data is sent across networks.

## What problem it solves

Devices need a way to find and communicate with each other across a network. IP addresses provide location information so that data knows where to go and where to come back from.

## Notes

### IPv4 vs IPv6

| | IPv4 | IPv6 |
|---|---|---|
| Bits | 32 | 128 |
| Notation | Dot-decimal: `10.5.112.105` | Hex: `2001:0db8:85a3::8a2e:0370:7334` |
| Addresses | ~4 billion | ~340 undecillion |
| Status | Still dominant | Deployed, growing (~45% of Google traffic) |

Every device auto-generates an IPv6 link-local address (`fe80::...`) regardless of whether the network supports IPv6. A globally routable IPv6 address (`2001::...`) requires the ISP or network to provide it via DHCP.

### Types of IP addresses

| Type | Description | Example |
|---|---|---|
| **Public** | Assigned to the router by the ISP. Shared by all devices on the network. Visible on the internet. | Your company's or ISP's IP |
| **Private** | Assigned to each device inside a network by DHCP. Not visible on the internet. | `10.5.112.105` on your laptop |
| **Static** | Manually configured. Never changes. Used for servers, geo-location services, remote access. | A production server |
| **Dynamic** | Assigned by DHCP. Can change. Most common for consumer and developer devices. | Your laptop's IP |

### Special / reserved ranges

Private ranges are reserved by international agreement and never appear on the public internet. Routers drop them if they try to leave the local network.

| Range | Typical use | Address count |
|---|---|---|
| `10.0.0.0 – 10.255.255.255` | Corporate networks, cloud VPCs | 16 million |
| `172.16.0.0 – 172.31.255.255` | Less common, some corporate | ~1 million |
| `192.168.0.0 – 192.168.255.255` | Home routers | ~65 thousand |
| `127.0.0.1` | Loopback — machine talks to itself, never leaves the OS | 1 |
| `0.0.0.0` | Wildcard — "all interfaces on this machine", used in server configs | — |

Convention: corporate networks tend to use `10.x.x.x` (more addresses), home routers tend to use `192.168.x.x`. Not a rule — just common practice.

### DORA — how a device gets an IP

When a device joins a network it has no IP. DHCP (Dynamic Host Configuration Protocol) assigns one automatically via a four-step handshake:

```
1. Discover  — device broadcasts: "anyone there? I need an IP"
2. Offer     — DHCP server: "here, take 10.5.112.105"
3. Request   — device: "I'll take it"
4. Acknowledge — server: "confirmed, it's yours for 8 hours (lease)"
```

The DHCP server runs on the router (home) or a dedicated server (corporate). It hands out four things: IP address, subnet mask, default gateway, DNS server.

IPs are leased — they expire. That's why dynamic IPs can change.

### NAT — how private and public IPs coexist

Your whole network (home or company) shares one public IP. NAT (Network Address Translation) runs on the gateway/router:

```
Outgoing: private IP (10.5.112.105) → swapped for public IP → internet
Incoming: public IP → router checks NAT table → routes to private IP
```

This is why IPv4 hasn't run out: millions of devices share a small pool of public IPs.

### ifconfig — reading network interfaces

`ifconfig` (interface configuration) shows every network interface on the machine — physical and virtual. Each block is one interface.

Key fields on the active `en0` (WiFi) block:

| Field | Meaning |
|---|---|
| `inet` | IPv4 address (from DHCP) |
| `inet6` | IPv6 address (link-local, auto-generated) |
| `ether` | MAC address — hardware ID burned into the network card at the factory |
| `mtu` | Max packet size in bytes (1500 = standard Ethernet) |
| `netmask` | Which IPs are local vs need the router |
| `broadcast` | Address to send to all devices on the network at once |
| `status: active` | This interface is connected and working |

Common interfaces:

| Interface | What it is |
|---|---|
| `lo0` | Loopback — `127.0.0.1`, machine to itself |
| `en0` | WiFi — your main connection to the network |
| `utun0–N` | VPN tunnels — route traffic through corporate network |
| `bridge0` | Bridge — used by Docker and virtual machines |

MAC vs IP: the IP is logical (assigned by DHCP, can change). The MAC is physical (hardcoded in hardware, never changes). Inside a local network, devices find each other by MAC using a protocol called ARP — IP is the human-readable address, MAC is what the hardware actually uses.

## Developer relevance

| Situation | What matters |
|---|---|
| Run `localhost:3000` | Traffic goes through `lo0` — never touches the network |
| Share dev server with a colleague | Give them your private IP (`en0`) + port |
| Hit an internal company API | Goes through `utun` (VPN) — blocked without it |
| Docker containers communicate | Use virtual bridge interface, not `en0` |
| Server config shows `0.0.0.0` | Listens on all interfaces — reachable from outside |
| Never hardcode IPs | DHCP can change them; use hostnames or env vars |

## Relations

- [[ipc-transports]] — once devices find each other via IP, they communicate through transport mechanisms (TCP, UDP, pipes, sockets)

## Tags

#system-design
