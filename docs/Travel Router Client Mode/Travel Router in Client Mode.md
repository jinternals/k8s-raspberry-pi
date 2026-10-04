# Travel Router Uplink

A Wi-Fi travel router bridging a UniFi network to an Omada gateway, and how to reach a
service behind it. The whole configuration is four settings across three devices —
everything else here exists to explain why the obvious approaches fail.

**Entry point:** `192.168.20.101:8081` → `192.168.0.201:8081`

---

## The path a request takes

```
  192.168.20.0/24 · MAIN                    │        192.168.0.0/24 · LAB
                                            │
 ┌──────────────┐   ┌──────────────┐   ┌────┴─────────────────┐   ┌──────────────┐
 │    UniFi     │   │ Travel router│   │   Omada ER7212PC     │   │    worker    │
 │ MAIN ROUTER  │──▶│ CLIENT·BRIDGE│──▶│   WAN3   │    LAN    │──▶│ SERVICE HOST │
 │ 192.168.20.1 │Wi-│192.168.20.107│RJ4│.20.101   │ .0.1      │   │192.168.0.201 │
 │ DHCP .150-254│Fi │   DHCP OFF   │ 5 │ STATIC   │ NAT       │   │    :8081     │
 └──────────────┘   └──────────────┘   └────┬─────────────────┘   └──────────────┘
                           └──────── one MAC upstream ────────┘
                         UniFi cannot tell these apart
```

The travel router is a media converter, not a router — it adds no NAT hop, which is
exactly why this works with a single port forward. It rewrites the MAC address on
everything passing through it, so at layer 2 UniFi sees a single client covering both
`.107` and `.101`. That is what rules out per-device firewall rules, policy and traffic
stats for anything behind the bridge — see the client-list detail below for the one thing
that does survive.

## Why there are two addresses

1. **The travel router joins the SSID as an ordinary station**, the same way a laptop's
   Wi-Fi card does, and takes a DHCP lease of its own: `192.168.20.107`. That address is
   purely for managing the travel router — nothing routes through it.
2. **It then bridges its Ethernet port onto that wireless link.** When the Omada gateway's
   WAN port broadcasts a DHCP request, the travel router has to relay it over Wi-Fi.
3. **But a station can only transmit frames carrying its own MAC.** Ordinary Wi-Fi uses a
   three-address frame format with no field for a third party's source MAC — only
   4-address mode, which WDS and vendor mesh uplinks use, can carry one. So the travel
   router rewrites the source MAC to its own on the way out, rewrites it back on the way
   in, and keeps a table of which IP belongs to which downstream device.
4. **UniFi answers the relayed request normally** and issues a second lease,
   `192.168.20.101`. Both addresses are real and fully routable on `192.168.20.0/24` —
   which is exactly why `.101` works as a port-forward target.
5. **UniFi ends up with two client entries, but only one of them is real.** A DHCP request
   carries the client's MAC in the packet payload (`chaddr`), which the bridge does *not*
   rewrite — it only rewrites the Ethernet header. So UniFi learns the Omada's true MAC
   from the payload and creates a client entry for it, while every actual frame from that
   device arrives under the travel router's MAC.

![UniFi client list showing the offline .101 entry above the live travel router at .107](Pasted%2021-09-2026%20at%2001.22.42.png)

   The result is one live entry — *Home Office | Lab Client*, TP-Link, `192.168.20.107` —
   and one permanently **offline** entry holding `192.168.20.101` under MAC
   `a8:29:48:97:49:13`, with no vendor, AP or SSID, because UniFi has never seen a frame
   from it. On the wire both addresses answer from the same synthesized MAC:

   ```
   ? (192.168.20.101) at 62:a4:b7:e:41:fb on en0
   ? (192.168.20.107) at 62:a4:b7:e:41:fb on en0
   ```

   `62:a4:b7:0e:41:fb` is locally administered — the bridge invented it.
   `a8:29:48:97:49:13` is vendor-assigned, and is the Omada's own WAN3 MAC.

   The client list also carries a stale `TL-WR902AC …41:fc`, offline, one octet below the
   bridge MAC ending `…41:fb`. The synthesized address is derived from the travel router's
   own, which is why the two sort next to each other. Harmless; do not try to bind anything
   to it.

   What follows from the split: **DHCP-level binding does reach the Omada.** UniFi accepts a
   Fixed IP and a Local DNS Record on that vendor MAC, and both are configured on this
   network — see the client panel further down. **Layer-2 features cannot work at all** — firewall
   rules, per-device policy, bandwidth limits and traffic stats all key off the Ethernet
   header, and nothing behind the bridge ever appears there. Static addresses on each
   device sidestep the whole question, which is what this build does.

One consequence worth remembering: the translation table is populated by outbound traffic.
On some firmware an unsolicited inbound connection arrives with no matching entry and is
dropped — that is the hang-then-timeout case below. This hardware handles it correctly.

## Where to set what

Four settings. Change the UniFi DHCP range **first**, so nothing leases an address you are
about to pin statically.

### UniFi — `192.168.20.1` · main router

**Settings → Networks → 192.168.20.0/24 → DHCP Range**

| Field | Value |
|---|---|
| Range | `192.168.20.150 – 192.168.20.254` |

Frees everything below `.150` for static assignment.

**As built — Client Devices → `a8:29:48:97:49:13` → Settings**

The name is not a standalone DNS record. It is attached to the Omada's ghost client entry,
using that entry's own Fixed IP and Local DNS Record fields:

![UniFi client settings panel for a8:29:48:97:49:13 with Fixed IP 192.168.20.101 and Local DNS Record omada.home.internal](Screenshot%202026-09-21%20at%201.24.57%E2%80%AFAM.png)

| Field | Value |
|---|---|
| Note | `192.168.20.101` |
| Fixed IP Address | enabled — `192.168.20.101` |
| Local DNS Record | enabled — `omada.home.internal` |

`.internal` is ICANN-reserved for private use, so it can never collide with a public name.
Avoid `.local` — that suffix belongs to mDNS, and Apple clients bypass your resolver for it
entirely.

This also settles the reservation question from the section above: UniFi accepted a Fixed IP
binding on the Omada's **real** MAC, the one it only ever learned from DHCP payloads. Whether
that binding is what actually hands out `.101` cannot be observed here, because WAN3 is
statically addressed and never asks — the two simply agree.

> **The alternative, if you want the name decoupled from the device.** Settings → Policy
> Table → Create New Policy → DNS *(Network 9.4)*, or Settings → Routing → Local DNS Record
> *(8.x)*, creates a Host (A) record with no MAC binding at all. Functionally equivalent
> here. Worth switching to if the Omada is ever replaced, since the client-bound record
> disappears with the client entry.

> **Known issue: the name does not resolve on macOS.** UniFi answers
> `dig omada.home.internal @192.168.20.1` authoritatively, but stamps the record with
> **TTL 0**, and macOS will not resolve it — `getaddrinfo`, `ping` and `curl` all fail at
> name resolution while the same request to the raw IP returns 200. Reach the service by IP
> from Apple clients until this is resolved.

### TP-Link travel router — `192.168.20.107` · client mode

**Network → LAN**

| Field | Value |
|---|---|
| DHCP Server | Disable |
| IP Type | Static |
| IP | `192.168.20.107` |
| Mask | `255.255.255.0` |
| Gateway | `192.168.20.1` |

DHCP off is what makes it bridge. With DHCP on it would NAT instead, and you would need a
second port forward here — which client mode does not offer.

> **Nothing else lives here.** Client mode has no WAN/LAN split and no port forwarding menu
> at all. Anything you aim at `.107:8081` hits the travel router itself.

### Omada ER7212PC — WAN3 `192.168.20.101` · LAN `192.168.0.1`

**Network Config → Internet → Edit WAN3**

![Omada Edit WAN3 page: Static IP 192.168.20.101, mask 255.255.255.0, gateway 192.168.20.1, Use Default MAC Address](Screenshot%202026-09-21%20at%201.21.33%E2%80%AFAM.png)

| Field | Value |
|---|---|
| Connection Type | Static IP |
| IP Address | `192.168.20.101` |
| Subnet Mask | `255.255.255.0` |
| Default Gateway | `192.168.20.1` |
| MAC Address | Use Default MAC Address |
| IPv6 | disabled |

`Use Default MAC Address` matters more than it looks: it is what keeps WAN3 presenting
`a8:29:48:97:49:13`, the MAC UniFi has the Fixed IP and DNS record bound to. Cloning a MAC
here would silently detach both.

DNS is not on this page — it sits under the collapsed **Advanced Settings**, set to
`192.168.20.1`.

**Network Config → NAT → Port Forwarding**

![Omada port forwarding rule list: five enabled rules, each bound to SFP WAN/LAN1 and WAN3](Screenshot%202026-09-21%20at%201.29.08%E2%80%AFAM.png)

`longhorn` is the worked example this document follows. Two things to copy when adding a
rule:

- **Bind the interface list, not just WAN3.** WAN1 is unplugged on this gateway and new
  rules default to it. Including `SFP WAN/LAN1` alongside `WAN3` is what makes a rule
  survive a move between uplinks; a rule bound only to the dead interface saves cleanly
  and does nothing.
- **Protocol is `All`, not TCP.** Harmless for HTTP services, but worth matching the rest.

After saving: Devices should read **Provisioned**, not Pending, and Network Tools → Ping to
`8.8.8.8` should succeed.

## Reading a failure

The shape of the failure identifies the layer. Timing matters more than the error text.

| Symptom | What it means | Fix |
|---|---|---|
| curl fails in ~10 ms | A TCP reset. Something answered and said nothing is listening. | Usually the wrong host — `.107` is the travel router. Otherwise the forward isn't matching. |
| curl hangs, then times out | Packets dropped in flight, not refused. | Firewall drop, or the bridge has no translation entry for an inbound flow. |
| Works from `.20.x`, fails from `.0.x` | Hairpin NAT. Omada gateways don't reliably loop a WAN address back to the LAN. | From inside the lab subnet, address `192.168.0.201` directly. |
| Rule saved, no effect | Bound to a dead WAN, or never pushed to the hardware. | Set Interface to WAN3; confirm the gateway shows Provisioned. |
| Reservation won't stick | Check which MAC you bound. The travel router's MAC covers both devices at layer 2; the Omada's real MAC only ever appears inside DHCP payloads. | Bind to the Omada's own MAC (`a8:29:48:…`), or skip DHCP entirely and set static addresses outside the pool. |
| Per-device policy has no effect | Firewall rules, bandwidth limits and stats key off the Ethernet header, where everything behind the bridge is one MAC. | No fix at this layer. An Ethernet run or a matched AP pair is the only cure. |
| `.101` will not ping | Nothing is wrong. Omada blocks ICMP on the WAN interface by default. | Not a path test here. Test the forwarded port instead. |
| Name fails, IP works | DNS, not routing. Record missing, at the wrong label, or the client rejects it. | `dig name @192.168.20.1`, then compare against `ping name`. If dig answers and ping does not, the record is there but the client is rejecting it. |
| Port 80 answers on `.101` | On this gateway it is the `gateway-http` rule reaching the ingress at `192.168.0.200` — **not** the Omada WAN management page. A bare `404` from `Server: nginx` is the ingress with no matching host rule. | Proves the path works. Test the real port before concluding anything. |
| TLS fails on `443`, TCP connects | `tlsv1 unrecognized name` — the ingress needs SNI and has no default certificate for a bare IP. | Not a forwarding fault. Use a hostname that the ingress serves. |

## Rules that generalise

1. **A port forward lives on the WAN address of the device that owns the LAN.** Not on
   whatever sits in front of it. The travel router's address was never a candidate.
2. **Client mode bridges; WISP mode routes.** Bridging costs you upstream visibility but
   saves a NAT layer and a forwarding rule. Prefer it unless something upstream limits you
   to one device, like a hotel captive portal.
3. **802.11 client mode presents a single MAC — but only in the Ethernet header.** DHCP
   payloads pass through intact, so the upstream router still learns the real MAC of each
   device and lists it, permanently offline. Anything keyed to DHCP can still work;
   anything keyed to layer 2 — policy, firewall, bandwidth, stats — cannot. An Ethernet run
   or a matched AP pair is the only cure.
4. **Static addresses belong outside the DHCP pool.** A static inside the range works for
   weeks and then collides on a day you have forgotten all of this.
5. **Test from outside the NAT boundary.** A forward tested from inside the LAN it forwards
   into proves nothing, and fails for reasons unrelated to the rule.

## Verification

Checked from `192.168.20.11` (main subnet, outside the NAT boundary) on 2026-09-21.

| Check | Result |
|---|---|
| `curl http://192.168.20.101:8081/` | **200 OK**, nginx, 11 ms connect |
| `.101` / `.107` share one MAC on the wire | Confirmed — both `62:a4:b7:0e:41:fb` in ARP (locally administered) |
| UniFi lists `.101` separately | Confirmed — offline entry, MAC `a8:29:48:97:49:13` (vendor-assigned) |
| `curl http://192.168.20.107:8081/` | Refused (~100 ms) — wrong host, as documented |
| `curl http://192.168.0.201:8081/` | Times out at 5 s — no route across the boundary |
| `curl http://192.168.20.101/` (port 80) | 404 — gateway WAN page listening |
| `ping 192.168.20.101` | 100% loss — WAN ICMP blocked, expected |
| `dig omada.home.internal @192.168.20.1` | `192.168.20.101`, authoritative, **TTL 0** |
| `curl http://omada.home.internal:8081/` | **Fails** — macOS cannot resolve the name |

All five forwards exercised from the same host:

| Rule | Probe | Result |
|---|---|---|
| `gateway-http` | `http://192.168.20.101:80/` | 404 from `Server: nginx` — ingress reached |
| `gateway-https` | `https://192.168.20.101:443/` | TCP connects; TLS `unrecognized name` (SNI required) |
| `headlamp` | `http://192.168.20.101:8080/` | **200 OK** |
| `k3s-api` | `https://192.168.20.101:6443/version` | **401** — API server answering |
| `longhorn` | `http://192.168.20.101:8081/` | **200 OK** |

Outstanding: the DNS record resolves via `dig` but not through the macOS system resolver.

---

Home Lab · UniFi → TL travel router (client) → Omada ER7212PC v2.0
