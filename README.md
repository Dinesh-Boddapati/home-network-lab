# Home Network Documentation & Analysis

Hands-on network documentation lab using command-line tools to map, analyse, and understand a home network environment. Demonstrates practical knowledge of IPv4/IPv6 addressing, DHCP, DNS resolution, and packet routing — core skills for IT support and network troubleshooting roles.

---

## How My Network Gets Configured 

When I connect a new device to my network, my PC sends a broadcast asking if any device can assign it an IP address. My router, acting as the DHCP server, responds with an IP address, subnet mask, default gateway, and DNS server information. This gives my device a full identity on the network — its own address, the boundary of its local network, the exit point to reach the internet, and the means to resolve domain names. I verified this using `ipconfig /all`, which displayed all four values.

---

## Network Environment

| Property | Value |
|---|---|
| **Router/Gateway** | Eero Mesh WiFi System (SAX2V1S) |
| **ISP** | Spectrum (Charter Communications) |
| **Connection type** | Wired LAN (Intel I226-V 2.5GbE NIC) |
| **Network adapter** | Intel Ethernet Controller I226-V |
| **WiFi adapter** | Qualcomm FastConnect 7800 (Wi-Fi 7) |
| **DNS Suffix** | lan |

---

## IP Configuration (`ipconfig /all`)

### Active Connection — Wired Ethernet

| Property | Value |
|---|---|
| **IPv4 Address** | 192.168.1.170 (DHCP assigned) |
| **Subnet Mask** | 255.255.255.0 (/24) |
| **Default Gateway** | 192.168.1.1 |
| **DHCP Server** | 192.168.1.1 |
| **DNS Servers** | 192.168.1.1 (local), ISP IPv6 DNS |
| **IPv6** | Assigned (redacted — public prefix) |
| **DHCP Lease** | 12 hours |
| **Node Type** | Hybrid (NetBIOS) |

### Additional Adapters Detected

| Adapter | Status | Notes |
|---|---|---|
| Norton VPN (OpenVPN) | Disconnected | VPN tunnel adapter, inactive |
| Norton VPN (Wintun) | Disconnected | Secondary VPN adapter, inactive |
| Wi-Fi (Qualcomm FastConnect 7800) | Disconnected | Wi-Fi 7 capable, currently using LAN |
| Bluetooth Network | Disconnected | PAN adapter, inactive |

### Key Observations
- Machine is on a **192.168.1.0/24** private network — standard Class C home subnet supporting up to 254 devices
- DHCP is enabled and the router (192.168.1.1) acts as both **DHCP server and DNS resolver**
- Multiple virtual Wi-Fi adapters present due to Wi-Fi 7 HBS (High Band Simultaneous) — allows concurrent 2.4GHz + 5GHz + 6GHz connections
- Norton VPN adapters present but inactive — when active, these would route traffic through an encrypted tunnel, changing the DNS and gateway

---

## DNS Resolution (`nslookup google.com`)

```
Server:  [Spectrum ISP DNS via Eero]
Address:  [ISP DNS - redacted]

Non-authoritative answer:
Name:    google.com
Addresses:  2607:f8b0:4009:810::200e (IPv6)
            142.251.41.142 (IPv4)
```

### Analysis
- DNS query goes to the **Eero router first**, which forwards to **Spectrum's DNS servers**
- Response is **non-authoritative** — meaning the answer came from Spectrum's DNS cache, not Google's own nameservers directly
- Google resolved to both an **IPv4** and **IPv6** address — confirms dual-stack network support
- DNS resolution time: sub-5ms (local cache hit on router)

---

## Packet Route Trace (`tracert google.com`)

```
Hop 1  — <1ms    — Eero router / Spectrum ISP handoff point
Hop 2  — timeout — ISP internal hop (ICMP blocked, normal)
Hop 3  — ~10ms   — charter.com backbone (Denton TX region)
Hops 4-6 — timeout — ISP backbone (ICMP filtered, normal)
Hops 7-8 — ~13ms  — Google peering point
Hops 9-16 — 13-40ms — Google's internal backbone (1e100.net)
Hop 17 — 34ms   — google.com destination reached
```

### Analysis
- **17 hops** total from home to Google's servers
- Timeouts at hops 2, 4, 5, 6 are **normal** — ISP routers block ICMP ping requests for security, does not mean packet loss
- Traffic routes through **Charter/Spectrum's backbone in Denton TX** before peering with Google's network
- Latency jumps from ~10ms (ISP backbone) to ~34ms (Google destination) — consistent with cross-continental routing
- `1e100.net` is Google's internal network domain (10^100 = googol, hence the company name)
- Final latency of **34ms is excellent** for a transatlantic-capable connection

---

## Network Diagram

```
[Gaming PC - 192.168.1.170]
        |
    [LAN Cable]
        |
[Eero Mesh Router - 192.168.1.1]
    DHCP Server
    DNS Forwarder
        |
  [Spectrum ISP]
  charter.com backbone
        |
   [Google DNS / Internet]
   142.251.41.142
```

---

## Practical Troubleshooting Framework

1. **Physical layer** — Is the cable connected? Is WiFi turned on? Are the router's lights normal? Most "internet is down" tickets are solved here.
2. **IP check** — Run `ipconfig`. A real `192.168.x.x` address means DHCP succeeded. A `169.254.x.x` address means the broadcast for an IP never got a response — points to a physical or router problem, not a settings issue.
3. **Gateway check** — `ping 192.168.1.1` (the router itself). If this fails, the problem is local — bad cable, WiFi issue, or the router is down. If it succeeds, the local network is healthy and the problem is further out.
4. **DNS-bypass check** — `ping 8.8.8.8` (Google's public DNS server, a fixed IP with no name lookup needed). If this works but `ping google.com` fails, the issue is isolated to DNS specifically — the internet connection itself is fine.
5. **Path check** — `tracert` to see exactly where the trail goes cold, hop by hop.

This lets me isolate *which layer* is broken instead of guessing — physical, local network, DNS, or somewhere further out on the path.

---

## Why This Matters: TTL and Tracert

Every packet carries a Time To Live (TTL) value that decreases by 1 at each router hop. If it hits 0, the packet is destroyed — this prevents packets from looping forever if two routers are misconfigured and keep forwarding to each other. `tracert` actually exploits this mechanism on purpose: it sends a packet with TTL=1 first (dies at hop 1, reveals who that is), then TTL=2 (dies at hop 2), and so on — building the hop-by-hop map one TTL increment at a time.

---

## Troubleshooting Scenarios Practised

| Scenario | Command Used | Finding |
|---|---|---|
| Verify IP assignment and adapter config | `ipconfig /all` | DHCP working, correct subnet, dual-stack IPv4+IPv6 |
| Test connectivity to external host | `ping google.com` | 0% packet loss, 34ms avg latency |
| Trace packet path to destination | `tracert google.com` | 17 hops, timeouts at ISP hops are normal/expected |
| Verify DNS resolution | `nslookup google.com` | Resolving via local router → ISP DNS, non-authoritative response |

---

## Skills Demonstrated

- IPv4 and IPv6 addressing and subnetting
- DHCP lease analysis and gateway identification
- DNS query analysis — authoritative vs non-authoritative responses
- Traceroute interpretation including normal ICMP timeout behaviour
- Network adapter identification and status analysis
- VPN adapter recognition
- Understanding of ISP routing and Google's backbone infrastructure
- Security awareness — public IPs and MAC addresses redacted from documentation

---

## Tools Used

- `ipconfig /all` — full network adapter configuration
- `ping` — connectivity and latency testing
- `tracert` — hop-by-hop packet route tracing
- `nslookup` — DNS resolution analysis

---

*Part of a home IT lab series. See also: [Custom PC Build](../custom-pc-build/README.md)*
