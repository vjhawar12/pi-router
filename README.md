# Raspberry Pi Linux Router & Stateful Firewall

A Linux-based router built on a Raspberry Pi to explore **TCP/IP routing, NAT, stateful packet filtering, DHCP/DNS, connection tracking, and packet-level troubleshooting**.

The Raspberry Pi acts as a gateway between a downstream Wi-Fi network and an upstream Ethernet network. Linux handles packet forwarding, `nftables` implements firewalling and NAT, and `dnsmasq` provides DHCP and DNS services to downstream clients.

The project is intended as a hands-on networking platform for understanding how packets move through a Linux router and how routing, firewalling, NAT, and network services interact.

## Network Architecture

```text
                    Upstream Network
                     / Internet
                         |
                         |
                       eth0
                  DHCP-assigned IP
                         |
               +--------------------+
               |   Raspberry Pi     |
               |    Linux Router    |
               |                    |
               |  IPv4 Forwarding   |
               |  nftables          |
               |  conntrack         |
               |  NAT / Masquerade  |
               |  dnsmasq           |
               +--------------------+
                         |
                       wlan0
                    10.0.50.1/24
                         |
              -----------------------
              |          |          |
            Client     Client     Client
         10.0.50.x   10.0.50.x  10.0.50.x
```

### Interfaces

| Interface       | Purpose                            |
| --------------- | ---------------------------------- |
| `eth0`          | Upstream network / Internet access |
| `wlan0`         | Downstream Wi-Fi network           |
| `wlan0` address | `10.0.50.1/24`                     |
| Client subnet   | `10.0.50.0/24`                     |
| DHCP pool       | `10.0.50.50 – 10.0.50.250`         |

## Packet Flow

For a downstream client accessing an external host:

```text
Client
10.0.50.x
    |
    | packet destined outside 10.0.50.0/24
    v
wlan0
10.0.50.1
    |
    | Linux routing decision
    v
nftables FORWARD chain
    |
    | stateful firewall / conntrack
    v
NAT masquerade
    |
    v
eth0
    |
    v
Upstream Gateway
    |
    v
Destination
```

Linux first determines the outgoing interface using the routing table. The packet then traverses the `FORWARD` firewall chain.

New connections originating from the downstream network are permitted, while unsolicited forwarded connections are denied by the default-drop policy.

Before leaving `eth0`, source NAT masquerading rewrites the packet's source address to the router's upstream address. Linux connection tracking maintains the flow state so return packets can be translated and forwarded back to the correct downstream client.

## Stateful Firewall

The router uses `nftables` for packet filtering.

The forwarding policy is **default deny**:

```nft
chain forward {
    type filter hook forward priority filter;
    policy drop;

    ct state established,related accept
    iifname "wlan0" oifname "eth0" ct state new accept
}
```

This provides two primary rules:

### Outbound connections

New connections may originate from the downstream network:

```text
wlan0 → eth0
```

### Return traffic

Packets belonging to connections already tracked by Linux are allowed:

```text
ct state established,related
```

Unmatched forwarded traffic is dropped.

This prevents an upstream host from initiating an arbitrary new connection directly to a downstream device while still allowing responses to connections initiated by downstream clients.

Packet counters are also maintained for accepted and dropped traffic to assist with debugging and firewall validation.

## NAT

The downstream network uses private addresses from:

```text
10.0.50.0/24
```

Packets leaving through `eth0` are translated using masquerading:

```nft
chain postrouting {
    type nat hook postrouting priority 100;
    oifname "eth0" masquerade
}
```

Masquerading dynamically uses the current upstream address assigned to `eth0`.

A client may therefore send:

```text
Source:      10.0.50.62
Destination: external host
```

while the upstream network sees:

```text
Source:      router eth0 address
Destination: external host
```

Connection tracking stores the translation so reply packets can be mapped back to `10.0.50.62`.

## IPv4 Forwarding

Linux normally behaves as an endpoint rather than forwarding packets between interfaces.

Routing is enabled through:

```text
net.ipv4.ip_forward = 1
```

This allows packets received on one network interface to be routed through another according to the system routing table and firewall rules.

## DHCP & DNS

`dnsmasq` provides network configuration to downstream Wi-Fi clients.

Clients receive addresses from:

```text
10.0.50.50 – 10.0.50.250
```

with the Raspberry Pi advertised as both:

```text
Default gateway: 10.0.50.1
DNS server:       10.0.50.1
```

The relevant configuration is contained in:

```text
configs/dnsmasq.conf
```

Restricting `dnsmasq` to the downstream interface prevents it from unintentionally serving DHCP on the upstream network.

## Repository Structure

```text
pi-router/
├── README.md
├── configs/
│   ├── dnsmasq.conf
│   ├── nftables.conf
│   └── sysctl.conf
└── .gitignore
```

### `nftables.conf`

Defines:

* stateful forwarding rules
* default-drop forwarding policy
* accepted/dropped packet counters
* source NAT masquerading

### `dnsmasq.conf`

Defines:

* downstream DHCP service
* DHCP address pool
* default gateway advertisement
* DNS server advertisement
* interface binding

### `sysctl.conf`

Enables Linux IPv4 forwarding.

## Networking Concepts Explored

This project is primarily intended to develop practical understanding of:

* IPv4 routing
* routing tables
* Ethernet and network interfaces
* TCP/IP
* DHCP
* DNS
* NAT
* source address translation
* Linux packet forwarding
* stateful firewalls
* connection tracking
* default-deny network policy
* ARP and neighbor discovery
* packet capture and inspection
* network troubleshooting

## Troubleshooting & Validation

The router is validated using standard Linux networking tools.

### Routing

```bash
ip route
```

Used to inspect the routing table and determine which interface Linux selects for a destination.

### Interfaces

```bash
ip addr
```

Used to verify interface addresses and subnet configuration.

### Neighbor / ARP table

```bash
ip neigh
```

Used to inspect Layer-2 neighbor resolution for directly connected hosts.

### Firewall

```bash
sudo nft list ruleset
```

Used to verify active filtering and NAT rules and inspect packet counters.

### Connection tracking

```bash
sudo conntrack -L
```

Used to inspect active flows and understand how Linux tracks state across routed and NATed connections.

### Packet capture

```bash
sudo tcpdump -i wlan0
sudo tcpdump -i eth0
```

Capturing the same flow on both interfaces makes it possible to observe how a packet changes as it traverses the router.

For example:

```text
wlan0:
10.0.50.62 → external server

eth0:
router-upstream-IP → external server
```

The difference between these captures demonstrates the effect of source NAT.

`tshark` is also used when filtered or structured packet output is more useful than a full interactive capture.

## Example Debugging Method

When a client cannot reach a remote host, the problem can be isolated progressively through the network stack:

```text
1. Does the client have a valid IP address?
            ↓
2. Can it reach 10.0.50.1?
            ↓
3. Does the Pi receive the packet on wlan0?
            ↓
4. Does Linux have a route to the destination?
            ↓
5. Does the firewall permit the flow?
            ↓
6. Does the packet leave eth0?
            ↓
7. Is NAT being applied?
            ↓
8. Does return traffic reach eth0?
            ↓
9. Does conntrack associate it with the original flow?
            ↓
10. Does the translated response return through wlan0?
```

This approach helps distinguish failures caused by addressing, Layer-2 connectivity, routing, firewall policy, NAT, DNS, or upstream connectivity.

## Security Model

The downstream network is treated as a separate network boundary.

The forwarding firewall follows a simple security policy:

* downstream clients may initiate connections toward the upstream network
* return traffic for established connections is allowed
* unsolicited forwarded connections are denied
* forwarding defaults to deny rather than allow
* NAT hides downstream private addresses from the upstream network

The current implementation focuses primarily on **data-plane forwarding and firewall behavior**.

Further hardening of the router's own management plane is intentionally separate from the forwarding policy.

## Future Work

Planned extensions include:

* restrictive firewall policy for traffic destined to the router itself
* management-plane isolation
* more granular firewall logging
* automated network validation tests
* WireGuard VPN integration
* network segmentation
* traffic inspection and telemetry
* programmable Linux packet filtering using eBPF

These extensions will build on the current router to explore increasingly advanced Linux networking and network-security concepts.

## Hardware & Software

**Hardware**

* Raspberry Pi 4
* Ethernet uplink
* onboard Wi-Fi interface

**Software**

* Linux
* NetworkManager
* nftables
* dnsmasq
* conntrack-tools
* tcpdump
* tshark

## Purpose

Rather than treating Linux networking commands as isolated configuration steps, this project focuses on understanding the complete packet path:

```text
Ethernet / Wi-Fi
      ↓
IP addressing
      ↓
Routing
      ↓
Firewall / conntrack
      ↓
NAT
      ↓
Upstream network
```

The goal is to build an understanding of network behavior from the packet level upward and provide a platform for future work in **embedded networking, Linux systems, network security, and programmable packet processing**.

