# Network Architecture

## Topology

```
Internet
   │
   ▼
VMnet8 (NAT) — WAN segment — 192.168.23.0/24
   │
   ▼
┌─────────────────────────────────────────┐
│              OPNsense-FW                 │
│  em0 → WAN     (192.168.23.x, DHCP)      │
│  em1 → LAN     10.10.10.1/24             │
│  em2 → OPT1    10.10.20.1/24             │
└─────────────────────────────────────────┘
        │                        │
        ▼                        ▼
VMnet2 — LAN/Attack       VMnet3 — Target
10.10.10.0/24             10.10.20.0/24
   │                          │
Kali Linux                Metasploitable2
10.10.10.10                10.10.20.50
```

## Hypervisor networking

Every VMware host-only network (VMnet2, VMnet3) automatically creates a matching virtual adapter on the Windows host itself ("VMware Network Adapter VMnetX"). These adapters are separate from the firewall's own gateway IPs and had to be moved off the `.1` address on both segments to avoid conflicting with the OPNsense gateway (see [troubleshooting.md](troubleshooting.md)).

| Windows host adapter | Segment | IP |
|---|---|---|
| VMware Network Adapter VMnet2 | 10.10.10.0/24 | 10.10.10.254 |
| VMware Network Adapter VMnet3 | 10.10.20.0/24 | 10.10.20.254 |

## OPNsense firewall/router

- **Version:** 26.7 (amd64)
- **Specs:** 1 vCPU, 2 GB RAM, 20 GB disk (UFS)

### Interface assignment

| Identifier | Description | Device | MAC | IP |
|---|---|---|---|---|
| wan | WAN | em0 | 00:0c:29:b2:ae:f2 | DHCP (192.168.23.0/24) |
| lan | LAN | em1 | 00:0c:29:b2:ae:fc | 10.10.10.1/24 |
| opt1 | TARGET | em2 | 00:0c:29:b2:ae:06 | 10.10.20.1/24 |

Interface names inside FreeBSD (`emX`) are not guaranteed to correspond to the order network adapters were added in VMware's VM settings. Real mapping was confirmed by running `tcpdump -ni emX` on each interface individually and observing which ARP/IP traffic actually appeared on it, rather than trusting the assumed order.

### Firewall rules

- **OPT1 (TARGET):** pass, any protocol, any source, any destination — permissive by design for lab use
- **LAN:** default allow LAN-to-any, plus an explicit rule permitting traffic toward the TARGET segment
- Both rules use `Direction: in` (relying on OPNsense's stateful tracking for return traffic — an earlier `Direction: Both` configuration caused inconsistent behavior and was corrected)

## Hosts

### Kali Linux (attacker)
- Segment: VMnet2
- IP: `10.10.10.10/24`, gateway `10.10.10.1`
- Tools used: `nmap`, Metasploit Framework (`msfconsole`)

### Metasploitable2 (target)
- Segment: VMnet3
- IP: `10.10.20.50/24`, gateway `10.10.20.1`
- Specs: 512 MB RAM, 8 GB disk
- Network configured statically via `/etc/network/interfaces`:
  ```
  auto eth0
  iface eth0 inet static
      address 10.10.20.50
      netmask 255.255.255.0
      gateway 10.10.20.1
  ```

## Design rationale

- **Host-only, not bridged**, for the attack and target segments — keeps intentionally vulnerable hosts off the home network entirely.
- **A dedicated firewall VM instead of VMware's built-in NAT/routing** — the goal was to practice real router/firewall administration (interface assignment, stateful rules, routing between subnets), not just get VMs talking to each other.
- **Private RFC1918 ranges (10.10.10.0/24, 10.10.20.0/24)** chosen to be unambiguous and never collide with typical home-network address spaces (usually 192.168.x.x).
