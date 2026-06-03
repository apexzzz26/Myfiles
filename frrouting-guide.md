# FRRouting (FRR) — Complete Learning Guide

---

## Table of Contents

1. [What is FRRouting?](#1-what-is-frrouting)
2. [History & Background](#2-history--background)
3. [Architecture Overview](#3-architecture-overview)
4. [Supported Protocols](#4-supported-protocols)
5. [Installation](#5-installation)
6. [Core Concepts](#6-core-concepts)
7. [Configuration Basics](#7-configuration-basics)
8. [Protocol Deep Dives](#8-protocol-deep-dives)
   - [BGP](#81-bgp-border-gateway-protocol)
   - [OSPF](#82-ospf-open-shortest-path-first)
   - [IS-IS](#83-is-is)
   - [EIGRP](#84-eigrp)
   - [RIP / RIPng](#85-rip--ripng)
   - [PIM / Multicast](#86-pim--multicast)
   - [BFD](#87-bfd-bidirectional-forwarding-detection)
   - [MPLS & LDP](#88-mpls--ldp)
   - [VRRP](#89-vrrp)
   - [Static Routes & Zebra](#810-static-routes--zebra)
9. [Zebra — Deep Dive](#9-zebra--deep-dive)
   - [What Zebra Does](#91-what-zebra-does)
   - [Zebra Architecture Internals](#92-zebra-architecture-internals)
   - [ZAPI — Zebra API](#93-zapi--zebra-api)
   - [RIB Management & Route Selection](#94-rib-management--route-selection)
   - [Netlink & Kernel FIB Programming](#95-netlink--kernel-fib-programming)
   - [Interface & Nexthop Tracking](#96-interface--nexthop-tracking)
   - [Zebra Configuration Reference](#97-zebra-configuration-reference)
   - [Dataplane Abstraction Layer](#98-dataplane-abstraction-layer)
   - [MPLS in Zebra](#99-mpls-in-zebra)
   - [Zebra Debugging & Diagnostics](#910-zebra-debugging--diagnostics)
10. [VRF (Virtual Routing and Forwarding)](#10-vrf-virtual-routing-and-forwarding)
11. [Route Maps & Filtering](#11-route-maps--filtering)
12. [Prefix Lists & Access Lists](#12-prefix-lists--access-lists)
13. [VTY Shell (vtysh)](#13-vty-shell-vtysh)
14. [Logging & Debugging](#14-logging--debugging)
15. [Integration with Linux Kernel](#15-integration-with-linux-kernel)
16. [FRR in Containerized / Cloud Environments](#16-frr-in-containerized--cloud-environments)
17. [Common Use Cases](#17-common-use-cases)
18. [Troubleshooting Cheat Sheet](#18-troubleshooting-cheat-sheet)
19. [Key Files & Paths Reference](#19-key-files--paths-reference)

---

## 1. What is FRRouting?

**FRRouting (FRR)** is a free, open-source IP routing protocol suite for Linux and Unix platforms. It provides implementations of virtually every major routing protocol used in enterprise, ISP, and data center networks. FRR runs as a user-space process and programs routes into the Linux kernel's routing table via Netlink.

Think of it as a full-featured software router that turns any Linux machine into a router capable of running BGP, OSPF, IS-IS, RIP, EIGRP, PIM, MPLS, BFD, and more.

### Why Use FRR?

| Use Case | Why FRR Fits |
|---|---|
| Software-defined networking | Runs on any Linux host |
| Network testing / labs | Free, widely available |
| Cloud / container networking | Lightweight, scriptable |
| ISP / data center routing | Full BGP, MPLS, EVPN support |
| Replacing expensive hardware routers | Same protocol stack, commodity hardware |

---

## 2. History & Background

```
Zebra (1999)
  └── Quagga (2003) — fork of Zebra
        └── FRRouting (2017) — fork of Quagga, by major tech companies
```

- **Zebra** — original open-source routing daemon by Kunihiro Ishiguro
- **Quagga** — long-lived fork; widely deployed but development slowed
- **FRRouting** — created in 2017 by a consortium including **Cumulus Networks, Big Switch, 6WIND, Volta Networks, OpenSourceRouting, and others**; maintained under the Linux Foundation

FRR moves much faster than Quagga, with modern features like EVPN, SR-MPLS, and YANG/NETCONF support. It is used in production by major ISPs and cloud providers.

---

## 3. Architecture Overview

FRR uses a **multi-daemon, multi-process architecture**. Each routing protocol runs as its own independent process. All daemons communicate with a central daemon called **Zebra**, which manages the Routing Information Base (RIB) and installs routes into the Linux kernel.

```
┌─────────────────────────────────────────────────────┐
│                    FRR User Space                   │
│                                                     │
│  bgpd   ospfd   ospf6d   isisd   eigrpd   ripd ...  │
│    │       │       │       │        │       │       │
│    └───────┴───────┴───────┴────────┴───────┘       │
│                        │                           │
│                   [ zebrad ]                        │
│              (Central RIB + FIB mgr)               │
│                        │                           │
└────────────────────────┼────────────────────────────┘
                         │  Netlink
               ┌─────────▼──────────┐
               │   Linux Kernel     │
               │  (Routing Table /  │
               │   FIB / dataplane) │
               └────────────────────┘
```

### Key Daemons

| Daemon | Protocol |
|---|---|
| `zebra` | Core RIB manager, kernel interface |
| `bgpd` | BGP (IPv4/IPv6, EVPN, VPN) |
| `ospfd` | OSPFv2 (IPv4) |
| `ospf6d` | OSPFv3 (IPv6) |
| `isisd` | IS-IS |
| `ripd` | RIPv2 |
| `ripngd` | RIPng (IPv6) |
| `eigrpd` | EIGRP |
| `pimd` | PIM (multicast) |
| `ldpd` | LDP (MPLS) |
| `bfdd` | BFD |
| `vrrpd` | VRRP |
| `staticd` | Static routes |
| `pathd` | SR-TE / Segment Routing |
| `nhrpd` | NHRP (DMVPN-like) |
| `sharpd` | Test/development protocol daemon |

### Inter-Daemon Communication

Daemons communicate over a **UNIX socket** using the **ZAPI (Zebra API)** protocol. This is how, for example, `bgpd` sends learned routes to `zebra` for installation into the kernel.

---

## 4. Supported Protocols

| Protocol | Standard | FRR Daemon | Notes |
|---|---|---|---|
| BGP-4 | RFC 4271 | bgpd | Full feature set |
| MP-BGP | RFC 4760 | bgpd | IPv6, VPN, EVPN |
| OSPFv2 | RFC 2328 | ospfd | IPv4 |
| OSPFv3 | RFC 5340 | ospf6d | IPv6 |
| IS-IS | ISO 10589 | isisd | IPv4/IPv6 |
| EIGRP | Cisco | eigrpd | Partial implementation |
| RIPv2 | RFC 2453 | ripd | |
| RIPng | RFC 2080 | ripngd | IPv6 |
| PIM-SM | RFC 7761 | pimd | Multicast |
| PIM-SSM | RFC 4607 | pimd | Source-Specific Multicast |
| IGMP | RFC 3376 | pimd | |
| LDP | RFC 5036 | ldpd | MPLS label distribution |
| MPLS | RFC 3031 | ldpd/zebra | With Linux kernel MPLS |
| BFD | RFC 5880 | bfdd | Fast failure detection |
| VRRP | RFC 5798 | vrrpd | Virtual gateway redundancy |
| NHRP | RFC 2332 | nhrpd | |
| Segment Routing | RFC 8402 | pathd/zebra | SR-MPLS, SRv6 |
| EVPN | RFC 7432 | bgpd | L2/L3 VPN |

---

## 5. Installation

### On Ubuntu/Debian

```bash
# Add FRR APT repository
curl -s https://deb.frrouting.org/frr/keys.gpg | sudo tee /usr/share/keyrings/frrouting.gpg > /dev/null

FRRVER="frr-stable"
echo deb '[signed-by=/usr/share/keyrings/frrouting.gpg]' \
  https://deb.frrouting.org/frr $(lsb_release -s -c) $FRRVER | \
  sudo tee -a /etc/apt/sources.list.d/frr.list

sudo apt update && sudo apt install frr frr-pythontools
```

### On RHEL/CentOS/Fedora

```bash
# Install from FRR RPM repo
curl -O https://rpm.frrouting.org/repo/frr-stable-repo-1-0.el8.noarch.rpm
sudo rpm -ivh frr-stable-repo-1-0.el8.noarch.rpm
sudo dnf install frr
```

### From Source

```bash
git clone https://github.com/FRRouting/frr.git
cd frr
./bootstrap.sh
./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var \
  --sbindir=/usr/lib/frr --enable-multipath=64
make -j$(nproc)
sudo make install
```

### Enable Daemons

Edit `/etc/frr/daemons` to enable/disable individual protocol daemons:

```
zebra=yes
bgpd=yes
ospfd=yes
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
ldpd=no
bfdd=yes
staticd=yes
eigrpd=no
vrrpd=no
```

### Start FRR

```bash
sudo systemctl enable frr
sudo systemctl start frr
sudo systemctl status frr
```

---

## 6. Core Concepts

### RIB vs FIB

| Term | Stands For | Location | Purpose |
|---|---|---|---|
| RIB | Routing Information Base | FRR (zebra) | All known routes from all protocols |
| FIB | Forwarding Information Base | Linux kernel | Best/active routes actually used for forwarding |

Zebra selects the best route from the RIB (based on Administrative Distance) and installs it into the FIB (Linux kernel routing table).

### Administrative Distance (AD)

When multiple protocols know a route to the same destination, the one with the **lowest AD** wins:

| Protocol | Default AD |
|---|---|
| Connected | 0 |
| Static | 1 |
| EIGRP Summary | 5 |
| BGP (external) | 20 |
| OSPF | 110 |
| IS-IS | 115 |
| RIP | 120 |
| EIGRP Internal | 90 |
| EIGRP External | 170 |
| BGP (internal) | 200 |

### Route Selection (BGP example path)

For BGP: Weight → Local Preference → Originated locally → AS Path length → Origin type → MED → eBGP over iBGP → IGP metric → Router ID

---

## 7. Configuration Basics

### Configuration Files

FRR configurations live in `/etc/frr/`. Each daemon has its own file:

```
/etc/frr/
  ├── frr.conf          ← unified config (modern, preferred)
  ├── zebra.conf
  ├── bgpd.conf
  ├── ospfd.conf
  ├── daemons           ← which daemons to start
  └── vtysh.conf        ← vtysh settings
```

### Config Structure (IOS-like Syntax)

FRR uses a Cisco IOS-inspired CLI. Example structure:

```
! Comments start with !
!
hostname router1
log syslog informational
!
interface eth0
 ip address 10.0.0.1/24
 description Link to Router2
!
router ospf
 router-id 1.1.1.1
 network 10.0.0.0/24 area 0
!
router bgp 65001
 neighbor 10.0.0.2 remote-as 65002
!
```

### Applying Configuration

You can either:
1. Edit files directly and reload: `sudo systemctl reload frr`
2. Use `vtysh` (the interactive shell) to configure live and save

---

## 8. Protocol Deep Dives

### 8.1 BGP (Border Gateway Protocol)

BGP is the routing protocol of the internet. FRR's `bgpd` is one of the most feature-complete open-source BGP implementations.

#### Basic eBGP Configuration

```
router bgp 65001
 bgp router-id 1.1.1.1
 !
 neighbor 192.168.1.2 remote-as 65002
 neighbor 192.168.1.2 description "Peer - Router2"
 neighbor 192.168.1.2 update-source lo
 !
 address-family ipv4 unicast
  network 10.10.0.0/24
  neighbor 192.168.1.2 activate
  neighbor 192.168.1.2 soft-reconfiguration inbound
 exit-address-family
```

#### iBGP with Route Reflector

```
router bgp 65001
 bgp router-id 10.0.0.1
 !
 neighbor 10.0.0.2 remote-as 65001
 neighbor 10.0.0.2 update-source loopback0
 !
 address-family ipv4 unicast
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 route-reflector-client
 exit-address-family
```

#### BGP with IPv6

```
router bgp 65001
 neighbor 2001:db8::2 remote-as 65002
 !
 address-family ipv6 unicast
  network 2001:db8:1::/48
  neighbor 2001:db8::2 activate
 exit-address-family
```

#### BGP EVPN (for VXLAN/datacenter)

```
router bgp 65001
 neighbor 10.0.0.2 remote-as 65001
 !
 address-family l2vpn evpn
  neighbor 10.0.0.2 activate
  advertise-all-vni
 exit-address-family
```

#### Key BGP Commands (vtysh)

```
show bgp summary
show bgp ipv4 unicast
show bgp neighbors 10.0.0.2
show bgp ipv4 unicast 10.10.0.0/24
clear bgp * soft
debug bgp updates
```

---

### 8.2 OSPF (Open Shortest Path First)

OSPF is a link-state IGP using Dijkstra's SPF algorithm.

#### OSPFv2 (IPv4) Basic Configuration

```
router ospf
 router-id 1.1.1.1
 log-adjacency-changes
 !
 network 10.0.0.0/24 area 0
 network 192.168.1.0/30 area 0
 !
 area 1 stub
 !
 passive-interface eth2
```

#### Interface-level OSPF config

```
interface eth0
 ip ospf area 0
 ip ospf cost 10
 ip ospf hello-interval 5
 ip ospf dead-interval 20
 ip ospf network point-to-point
```

#### OSPFv3 (IPv6)

```
interface eth0
 ipv6 ospf6 area 0.0.0.0
 ipv6 ospf6 cost 10
!
router ospf6
 router-id 1.1.1.1
 area 0.0.0.0 range 2001:db8:1::/48
```

#### Key OSPF Commands

```
show ip ospf neighbor
show ip ospf database
show ip ospf route
show ip ospf interface eth0
debug ospf adj
debug ospf packet all
```

---

### 8.3 IS-IS

IS-IS is a link-state IGP commonly used by ISPs.

```
router isis CORE
 net 49.0001.0000.0000.0001.00
 is-type level-2-only
 metric-style wide
!
interface eth0
 ip router isis CORE
 isis circuit-type level-2-only
 isis metric 10
```

---

### 8.4 EIGRP

FRR includes a partial EIGRP implementation (IPv4 only):

```
router eigrp 100
 network 10.0.0.0/8
 passive-interface eth1
```

---

### 8.5 RIP / RIPng

Simple distance-vector protocol. Useful for small or legacy networks.

#### RIPv2

```
router rip
 version 2
 network 192.168.1.0/24
 redistribute connected
 no auto-summary
```

#### RIPng (IPv6)

```
interface eth0
 ipv6 rip MYRIP enable
!
router ripng
 network eth0
```

---

### 8.6 PIM / Multicast

Protocol Independent Multicast (PIM) enables IP multicast routing.

#### PIM-SM Configuration

```
interface eth0
 ip pim
 ip igmp
!
router pim
 rp 10.0.0.1 224.0.0.0/4
```

#### Checking Multicast State

```
show ip pim neighbor
show ip pim interface
show ip mroute
show ip igmp groups
```

---

### 8.7 BFD (Bidirectional Forwarding Detection)

BFD provides sub-second failure detection for routing protocol adjacencies.

#### Enable BFD globally

```
bfd
 peer 10.0.0.2
  receive-interval 300
  transmit-interval 300
  detect-multiplier 3
 !
!
```

#### Attach BFD to BGP neighbor

```
router bgp 65001
 neighbor 10.0.0.2 bfd
```

#### Attach BFD to OSPF

```
interface eth0
 ip ospf bfd
```

---

### 8.8 MPLS & LDP

FRR supports MPLS via LDP (Label Distribution Protocol), requiring Linux kernel MPLS support.

#### Enable kernel MPLS

```bash
sudo modprobe mpls_router
sudo modprobe mpls_iptunnel
echo 1 | sudo tee /proc/sys/net/mpls/conf/eth0/input
echo 100000 | sudo tee /proc/sys/net/mpls/platform_labels
```

#### LDP Configuration

```
mpls ldp
 router-id 1.1.1.1
 !
 address-family ipv4
  discovery transport-address 1.1.1.1
  !
  interface eth0
  !
 exit-address-family
```

---

### 8.9 VRRP

Virtual Router Redundancy Protocol — provides IP gateway redundancy.

```
interface eth0
 vrrp 10 ip 192.168.1.1
 vrrp 10 priority 200
 vrrp 10 advertisement-interval 1000
```

---

### 8.10 Static Routes & Zebra

Zebra is the core daemon that manages static routes, connected routes, and programs the Linux kernel. Static routes are configured here and are available to all other daemons via redistribution. See **Section 9** for a complete Zebra deep-dive.

```
ip route 0.0.0.0/0 192.168.1.254          ! default route
ip route 10.10.0.0/16 192.168.1.1 200     ! with administrative distance of 200
ip route 10.20.0.0/24 blackhole           ! null route / blackhole
ip route 10.30.0.0/24 reject              ! reject with ICMP unreachable
ipv6 route 2001:db8::/32 2001:db8::1
ipv6 route ::/0 blackhole                 ! IPv6 default blackhole
```

---

## 9. Zebra — Deep Dive

Zebra is the **heart of FRR**. Every other daemon depends on it. Understanding Zebra deeply is essential for troubleshooting routing issues, building custom integrations, and tuning performance.

---

### 9.1 What Zebra Does

Zebra serves four fundamental roles:

| Role | Description |
|---|---|
| **RIB Manager** | Collects routes from all protocol daemons (BGP, OSPF, etc.) into one unified Routing Information Base |
| **Best-path selector** | Picks the best route to each destination using Administrative Distance and metric |
| **FIB programmer** | Installs selected routes into the Linux kernel (FIB) via Netlink |
| **Interface & nexthop monitor** | Tracks interface state, IP address changes, and nexthop reachability; notifies all daemons |

Without Zebra running, **no other FRR daemon can function** — they all register with Zebra on startup via ZAPI.

---

### 9.2 Zebra Architecture Internals

```
┌───────────────────────────────────────────────────────────────┐
│                          zebrad                               │
│                                                               │
│  ┌──────────┐   ┌────────────┐   ┌────────────────────────┐  │
│  │  ZAPI    │   │    RIB     │   │   Dataplane Layer      │  │
│  │ Listener │──▶│  (per-VRF) │──▶│  (kernel / DPDK / etc) │  │
│  │ (sockets)│   │            │   └────────────────────────┘  │
│  └──────────┘   └────────────┘              │                 │
│       ▲               │                     │ Netlink         │
│       │          Route Selection             ▼                 │
│  bgpd, ospfd,    (AD + metric)     ┌────────────────┐        │
│  isisd, ripd,         │            │  Linux Kernel  │        │
│  staticd, ...         ▼            │  (FIB / RIB)  │        │
│                  ┌──────────┐      └────────────────┘        │
│                  │ Nexthop  │                                  │
│                  │ Tracking │                                  │
│                  └──────────┘                                  │
└───────────────────────────────────────────────────────────────┘
```

#### Key internal subsystems

| Subsystem | Purpose |
|---|---|
| **rib_table** | Per-VRF, per-AFI/SAFI routing table (RIB) |
| **nexthop group** | Shared nexthop objects; enables ECMP and fast updates |
| **nhg (Nexthop Group)** | Hardware-offload-friendly nexthop abstraction |
| **interface table** | Tracks all interfaces, addresses, and link state |
| **zebra_dplane** | Abstraction layer between RIB and actual dataplane (kernel, DPDK, etc.) |
| **FPM (Forwarding Plane Manager)** | Optional: streams RIB updates to external forwarders (e.g. hardware ASICs) |

---

### 9.3 ZAPI — Zebra API

ZAPI is the private binary protocol used for communication between Zebra and all other FRR daemons. It runs over **UNIX domain sockets** located in `/var/run/frr/`.

#### Socket paths

```
/var/run/frr/zserv.api     ← main ZAPI socket
/var/run/frr/zebra.vty     ← VTY (vtysh) socket
```

#### What daemons do over ZAPI

| Operation | Direction | Example |
|---|---|---|
| Route add/delete | Daemon → Zebra | BGP installs a learned prefix |
| Route redistribute | Zebra → Daemon | Zebra sends connected routes to OSPF |
| Interface up/down events | Zebra → All | eth0 goes down, all daemons notified |
| Nexthop updates | Zebra → Daemon | Nexthop becomes unreachable |
| Label allocation (MPLS) | Daemon → Zebra | LDP requests labels |
| BFD status | Zebra ↔ bfdd | BFD session state changes |
| VRF add/delete | Zebra → All | New VRF created |

#### ZAPI message types (selected)

```
ZEBRA_ROUTER_ID_UPDATE       Zebra notifies daemons of router-id changes
ZEBRA_INTERFACE_ADD          New interface detected
ZEBRA_INTERFACE_UP/DOWN      Link state change
ZEBRA_ROUTE_ADD              Daemon adds a route to RIB
ZEBRA_ROUTE_DELETE           Daemon removes a route
ZEBRA_REDISTRIBUTE_ADD       Daemon requests route redistribution
ZEBRA_NEXTHOP_UPDATE         Nexthop reachability changed
ZEBRA_LABEL_MANAGER_*        MPLS label management
ZEBRA_VRF_ADD/DELETE         VRF lifecycle events
ZEBRA_NHG_ADD/DELETE         Nexthop group management
```

---

### 9.4 RIB Management & Route Selection

The RIB (Routing Information Base) in Zebra holds **all routes from all sources**. Zebra then selects the **best route(s)** for installation into the kernel FIB.

#### Route selection algorithm

```
1. Lowest Administrative Distance wins
      ↓  (tie)
2. Lowest metric wins
      ↓  (tie)
3. ECMP — install multiple nexthops if equal cost
```

#### Administrative Distance table

| Source | Default AD | Configurable? |
|---|---|---|
| Connected | 0 | No |
| Static | 1 | Yes |
| EIGRP Summary | 5 | Yes |
| BGP External | 20 | Yes |
| OSPF | 110 | Yes |
| IS-IS | 115 | Yes |
| RIP | 120 | Yes |
| EIGRP Internal | 90 | Yes |
| EIGRP External | 170 | Yes |
| BGP Internal | 200 | Yes |

#### Overriding Administrative Distance

```
! Change OSPF AD to 150
router ospf
 distance 150

! Per-protocol per-route AD
router ospf
 distance ospf intra-area 110 inter-area 115 external 120

! For BGP: separate eBGP, iBGP, and local
router bgp 65001
 distance bgp 20 200 200
```

#### ECMP (Equal-Cost Multi-Path)

FRR supports ECMP — multiple nexthops for the same prefix when metrics are equal:

```
! Allow up to 64 ECMP paths (set at compile time with --enable-multipath=64)

! Verify ECMP in RIB
show ip route 10.0.0.0/24

! Kernel will show multipath entries
ip route show 10.0.0.0/24
```

#### Nexthop Groups (NHG)

FRR 7.5+ uses **Nexthop Groups** — shared nexthop objects referenced by multiple routes. This enables:
- Faster convergence (update one NHG, all routes using it converge instantly)
- Hardware offload compatibility
- Efficient ECMP representation

```
! View nexthop groups
show nexthop-group rib
show nexthop-group rib singleton
show nexthop-group kernel
```

---

### 9.5 Netlink & Kernel FIB Programming

Zebra communicates with the Linux kernel exclusively via **Netlink** (AF_NETLINK socket family). This is how routes are installed, deleted, and monitored.

#### What Zebra does with Netlink

| Action | Netlink Message |
|---|---|
| Add route | `RTM_NEWROUTE` |
| Delete route | `RTM_DELROUTE` |
| Add address to interface | `RTM_NEWADDR` |
| Monitor interface events | `RTM_NEWLINK` / `RTM_DELLINK` |
| Monitor address events | `RTM_NEWADDR` / `RTM_DELADDR` |
| Add MPLS label | `RTM_NEWROUTE` with MPLS encap |
| ECMP | `RTM_NEWROUTE` with multiple nexthops (RTA_MULTIPATH) |

#### Verifying what Zebra installed in kernel

```bash
# All routes (kernel FIB)
ip route show

# Specific prefix
ip route show 10.0.0.0/24

# All routing tables (including VRFs)
ip route show table all

# Watch Netlink events in real time
ip monitor route
ip monitor all

# Raw Netlink dump (advanced)
ss -f netlink
```

#### Netlink socket options Zebra uses

Zebra enables `SO_RCVBUF` with large buffer sizes and uses **async Netlink** with batching to reduce system calls during bulk route updates (e.g. BGP full table install). This is critical for convergence performance with large routing tables (100k+ routes).

---

### 9.6 Interface & Nexthop Tracking

Zebra continuously monitors all network interfaces via Netlink and maintains an **interface table** shared with all daemons.

#### Interface events Zebra tracks

- Link up / down
- IP address added / removed
- MTU changes
- VRF membership changes
- VLAN changes (with switchdev)

#### Nexthop Tracking (NHT)

Nexthop Tracking is a critical Zebra feature. When a protocol daemon (e.g. BGP) registers a nexthop, Zebra watches whether that nexthop is reachable. If it becomes unreachable, Zebra immediately notifies the daemon.

```
! Enable nexthop tracking for BGP (enabled by default in modern FRR)
router bgp 65001
 bgp nexthop trigger delay 5    ! wait 5s before acting on NHT change
```

```
! View tracked nexthops
show zebra nexthop-tracking ipv4
show zebra nexthop-tracking ipv6
```

Without NHT, BGP would keep advertising routes via an unreachable nexthop. NHT is what enables **fast BGP convergence** without running BFD.

---

### 9.7 Zebra Configuration Reference

Zebra is configured in `/etc/frr/zebra.conf` or inside the unified `/etc/frr/frr.conf`.

#### Interface configuration

```
interface eth0
 description Uplink to Core
 ip address 10.0.0.1/24
 ipv6 address 2001:db8::1/64
 ip ospf cost 10
 link-detect                    ! track link state
 no shutdown
!
interface lo
 ip address 1.1.1.1/32          ! loopback for router-id
```

#### Static routes (full options)

```
! Basic
ip route 10.0.0.0/8 192.168.1.1

! With custom AD (distance)
ip route 10.0.0.0/8 192.168.1.1 210

! With tag
ip route 10.0.0.0/8 192.168.1.1 tag 42

! With nexthop interface
ip route 10.0.0.0/8 eth0

! Blackhole (silently drop)
ip route 192.0.2.0/24 blackhole

! Reject (drop + ICMP unreachable)
ip route 192.0.2.0/24 reject

! Floating static (higher AD = backup route)
ip route 0.0.0.0/0 10.0.0.2 200   ! AD 200 — only used if main route gone

! Recursive (resolve nexthop via RIB)
ip route 10.10.0.0/16 10.0.0.2

! With BFD monitoring
ip route 10.0.0.0/8 192.168.1.1 bfd
```

#### Redistribution

Routes can be redistributed between protocols via Zebra:

```
router ospf
 redistribute connected
 redistribute static route-map STATIC_TO_OSPF
 redistribute bgp metric 20 metric-type 2

router bgp 65001
 address-family ipv4 unicast
  redistribute ospf route-map OSPF_TO_BGP
  redistribute connected
  redistribute static
 exit-address-family
```

#### Router ID

```
! Set global router-id (used by OSPF, BGP, etc.)
router-id 1.1.1.1

! Or per-protocol
router ospf
 router-id 1.1.1.1

router bgp 65001
 bgp router-id 1.1.1.1
```

#### IP Forwarding enforcement

FRR can optionally enable kernel forwarding itself:

```
! In zebra.conf — enable IPv4 and IPv6 forwarding at startup
ip forwarding
ipv6 forwarding
```

#### Route table settings

```
! Assign zebra table ID for default VRF (advanced)
zebra route-table default

! Configure maximum ECMP paths
! (set at compile time: --enable-multipath=N)
```

---

### 9.8 Dataplane Abstraction Layer

In FRR 7.2+, Zebra has a **Dataplane Abstraction Layer** (`zebra_dplane`) that decouples route programming logic from the specific forwarding backend. This enables:

- Linux kernel (default) via Netlink
- DPDK-based fast path
- Hardware ASIC offload via **FPM** (Forwarding Plane Manager)
- P4-programmed dataplanes
- Custom/vendor dataplanes (via plugin API)

#### FPM (Forwarding Plane Manager)

FPM is an optional module that streams RIB updates to an external process using **Protobuf or Netlink encoding** over a TCP socket (default port 2620). This is used by:

- Hardware ASICs in whitebox switches (e.g. Mellanox, Broadcom)
- External controller software
- Hardware acceleration platforms

```
! Enable FPM in zebra
zebra
 fpm address 127.0.0.1 port 2620
```

```bash
# Verify FPM connection status
sudo vtysh -c "show zebra fpm stats"
```

#### Dplane threading model

The dataplane runs in a **separate thread** from the main Zebra RIB thread. This means:
- Route programming to kernel is non-blocking
- The RIB can continue processing new updates while the kernel installs old ones
- Supports batched Netlink writes for performance

---

### 9.9 MPLS in Zebra

Zebra is also responsible for MPLS label management in FRR.

#### Label Manager

Zebra includes a **Label Manager** that allocates MPLS label ranges to daemons that request them (LDP, BGP, Segment Routing). Labels are allocated from the platform label space.

```bash
# Enable MPLS in Linux kernel
sudo modprobe mpls_router
sudo modprobe mpls_iptunnel
sudo sysctl -w net.mpls.platform_labels=1048575

# Per-interface MPLS input
sudo sysctl -w net.mpls.conf.eth0.input=1
```

```
! Zebra config for MPLS
mpls enable

! View label table
show mpls table
show mpls ftn                 ! FEC-to-NHLFE table (FTN)
show mpls nhlfe               ! Next Hop Label Forwarding Entry
show mpls ldp-sync interface  ! LDP sync status
```

#### MPLS Route Entry Example

```bash
# View MPLS forwarding table in kernel
ip -M route show

# Example output:
# 16 as to inet 10.0.0.2/32 via inet 192.168.1.2 dev eth0
```

---

### 9.10 Zebra Debugging & Diagnostics

#### Essential show commands

```
show zebra version
show zebra client summary           ! which daemons are connected via ZAPI
show zebra router-id                ! current router-id per VRF

show interface                      ! all interfaces and addresses
show interface eth0                 ! specific interface detail

show ip route                       ! IPv4 RIB (all routes)
show ip route summary               ! route count per protocol
show ip route 10.0.0.0/24          ! specific prefix
show ip route ospf                  ! only OSPF routes
show ip route bgp                   ! only BGP routes
show ip route connected             ! only connected routes
show ip route static                ! only static routes

show ipv6 route                     ! IPv6 RIB

show nexthop-group rib              ! all nexthop groups
show zebra nexthop-tracking ipv4    ! NHT table

show ip nht                         ! nexthop tracking (shorthand)
show running-config                 ! zebra running config
```

#### Debug flags for Zebra

```
debug zebra events          ! general events (interface up/down, etc.)
debug zebra kernel          ! Netlink messages to/from kernel
debug zebra kernel msgdump  ! full raw Netlink message dump (very verbose)
debug zebra rib             ! RIB add/delete/select operations
debug zebra rib detailed    ! more verbose RIB debugging
debug zebra nht             ! nexthop tracking events
debug zebra fpm             ! FPM forwarding plane messages
debug zebra dplane          ! dataplane thread operations
debug zebra dplane detailed ! verbose dataplane
debug zebra mpls            ! MPLS label operations
debug zebra vxlan           ! VXLAN/EVPN dataplane events
debug zebra pw              ! pseudowire operations
```

#### Useful diagnostic workflow for route issues

```bash
# Step 1 — Is the route in the RIB?
sudo vtysh -c "show ip route 10.0.0.0/24"

# Step 2 — Is the route in the kernel?
ip route show 10.0.0.0/24

# Step 3 — If in RIB but not kernel, check dplane
sudo vtysh -c "debug zebra dplane"
sudo vtysh -c "debug zebra kernel"
sudo journalctl -u frr -f

# Step 4 — Is the nexthop reachable?
sudo vtysh -c "show ip nht"
ping <nexthop_ip>

# Step 5 — Are ZAPI clients connected?
sudo vtysh -c "show zebra client summary"

# Step 6 — Check for Netlink errors
sudo vtysh -c "show zebra kernel"
dmesg | grep -i netlink
```

---

## 10. VRF (Virtual Routing and Forwarding)

VRFs allow you to have multiple independent routing tables on a single device — essential for MPLS VPNs, multi-tenancy, and network segmentation.

### Create a VRF in Linux

```bash
sudo ip link add vrf-red type vrf table 10
sudo ip link set vrf-red up
sudo ip link set eth1 master vrf-red
```

### Configure in FRR

```
vrf red
 vni 100
!
interface eth1 vrf red
 ip address 10.1.0.1/24
!
router bgp 65001 vrf red
 neighbor 10.1.0.2 remote-as 65002
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
```

### Route Leaking Between VRFs

```
router bgp 65001
 address-family ipv4 unicast
  import vrf red
 exit-address-family
!
router bgp 65001 vrf red
 address-family ipv4 unicast
  import vrf default
 exit-address-family
```

---

## 11. Route Maps & Filtering

Route maps are the primary tool for manipulating and filtering routing information.

### Route Map Structure

```
route-map MY_MAP permit 10       ! clause 10 — permit matching routes
 match ip address prefix-list MY_PREFIXES
 set local-preference 200
 set community 65001:100
!
route-map MY_MAP deny 20         ! clause 20 — deny everything else
!
```

### Applying Route Maps to BGP

```
router bgp 65001
 neighbor 10.0.0.2 route-map MY_MAP in
 neighbor 10.0.0.2 route-map OUT_MAP out
```

### Common match conditions

```
match ip address prefix-list PL1         ! match a prefix list
match ip address access-list ACL1        ! match an ACL
match as-path AS_PATH_LIST               ! match AS path
match community COMM_LIST                ! match BGP community
match interface eth0                     ! match incoming interface
match metric 100                         ! match route metric
match tag 50                             ! match route tag
match origin igp                         ! match BGP origin
```

### Common set actions

```
set local-preference 150
set community 65001:200 additive
set as-path prepend 65001 65001 65001
set metric 100
set weight 50
set next-hop 192.168.1.1
set origin igp
set tag 99
set ip next-hop peer-address
```

---

## 11. Prefix Lists & Access Lists

### Prefix Lists (preferred for routing use)

```
ip prefix-list ALLOW_RFC1918 seq 5 permit 10.0.0.0/8 le 32
ip prefix-list ALLOW_RFC1918 seq 10 permit 172.16.0.0/12 le 32
ip prefix-list ALLOW_RFC1918 seq 15 permit 192.168.0.0/16 le 32
ip prefix-list ALLOW_RFC1918 seq 20 deny 0.0.0.0/0 le 32

! le = less-than-or-equal (any prefix of that length or more specific)
! ge = greater-than-or-equal
```

### Access Lists (ACLs)

```
access-list 10 permit 192.168.1.0/24
access-list 10 deny any

ip access-list extended FILTER
 permit ip 10.0.0.0/8 any
 deny   ip any any log
```

### AS Path Lists (for BGP)

```
bgp as-path access-list TRANSIT permit ^65100_
bgp as-path access-list NO_TRANSIT deny .*65200.*
```

### Community Lists

```
bgp community-list standard CUSTOMERS permit 65001:100
bgp community-list expanded MATCH_ALL permit .*
```

---

## 12. VTY Shell (vtysh)

`vtysh` is the unified command-line interface that talks to all running FRR daemons simultaneously.

### Starting vtysh

```bash
sudo vtysh
```

### Basic navigation

```
router1# ?                         ! list commands
router1# show ?                    ! list show commands
router1# conf t                    ! enter configuration mode
router1(config)# interface eth0
router1(config-if)# ip address 10.0.0.1/24
router1(config-if)# end            ! return to exec mode
router1# write                     ! save configuration
router1# write file                ! save to /etc/frr/frr.conf
```

### One-shot commands (from bash)

```bash
sudo vtysh -c "show bgp summary"
sudo vtysh -c "show ip route"
sudo vtysh -c "conf t" -c "router bgp 65001" -c "neighbor 10.0.0.2 shutdown"
```

### Useful show commands

```
show version
show ip route
show ip route summary
show ip bgp summary
show ip ospf neighbor
show mpls table
show bfd peers
show vrf
show interface
show running-config
show startup-config
```

---

## 13. Logging & Debugging

### Logging Configuration

```
log syslog informational
log file /var/log/frr/frr.log
log timestamp precision 6
```

Log levels (most → least verbose): `debugging`, `informational`, `notifications`, `warnings`, `errors`, `critical`, `alerts`, `emergencies`

### Protocol Debugging

```
! BGP
debug bgp updates
debug bgp events
debug bgp fsm

! OSPF
debug ospf adj
debug ospf event
debug ospf packet all detail

! IS-IS
debug isis adj-packets
debug isis update-packets

! Zebra / kernel
debug zebra kernel
debug zebra rib
```

### Disable all debugging

```
undebug all
```

### View logs in real time

```bash
sudo journalctl -u frr -f
sudo tail -f /var/log/frr/frr.log
```

---

## 14. Integration with Linux Kernel

FRR operates entirely in user space and relies on the Linux kernel for packet forwarding. Key integration points:

### Netlink

FRR uses **Netlink** (Linux kernel API) to:
- Install and remove routes in the kernel FIB
- Monitor interface state changes
- Receive kernel routing events

### IP Forwarding (required!)

```bash
# Enable IPv4 forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward
# Or permanently in /etc/sysctl.conf:
net.ipv4.ip_forward = 1

# Enable IPv6 forwarding
echo 1 > /proc/sys/net/ipv6/conf/all/forwarding
```

### Kernel Routing Table

```bash
# View the kernel FIB (what FRR installed)
ip route show
ip -6 route show

# View all routing tables
ip route show table all

# Specific table (e.g., VRF)
ip route show table 10
```

### MPLS Kernel Setup

```bash
modprobe mpls_router mpls_iptunnel
sysctl -w net.mpls.platform_labels=100000
sysctl -w net.mpls.conf.eth0.input=1
```

---

## 15. FRR in Containerized / Cloud Environments

FRR is widely used in containers and cloud-native networking.

### Docker

```bash
docker run -it --privileged --net=host \
  frrouting/frr:latest /bin/bash
```

### Kubernetes (via Helm / CNI)

- **Calico** uses BIRD (a BGP daemon), but FRR is used in Calico Enterprise and other CNIs
- **Metallb** (layer 2/BGP load balancer) can use FRR as its BGP speaker
- **Cilium** integrates FRR for BGP Control Plane

### MetalLB + FRR Mode

```yaml
apiVersion: metallb.io/v1beta1
kind: BGPPeer
metadata:
  name: upstream-router
spec:
  myASN: 65001
  peerASN: 65000
  peerAddress: 10.0.0.1
```

### Cloud / Overlay Networks

FRR is a common building block for:
- EVPN/VXLAN fabrics (leaf-spine datacenter design)
- SR-MPLS transport networks
- WireGuard + BGP dynamic routing

---

## 16. Common Use Cases

### Use Case 1: Simple BGP Router

Turn a Linux server into an internet-facing BGP router with upstream transit peers:

```
router bgp 64512
 bgp router-id 203.0.113.1
 neighbor 198.51.100.1 remote-as 1299       ! Upstream ISP 1
 neighbor 203.0.113.254 remote-as 3356      ! Upstream ISP 2
 !
 address-family ipv4 unicast
  network 203.0.113.0/24
  neighbor 198.51.100.1 activate
  neighbor 203.0.113.254 activate
  neighbor 198.51.100.1 route-map FILTER-IN in
  neighbor 203.0.113.254 route-map FILTER-IN in
 exit-address-family
```

### Use Case 2: OSPF Lab

Two routers exchanging OSPF routes in area 0 over a point-to-point link:

**Router 1:**
```
interface eth0
 ip address 10.0.0.1/30
 ip ospf network point-to-point
!
router ospf
 router-id 1.1.1.1
 network 10.0.0.0/30 area 0
 network 192.168.1.0/24 area 0
```

**Router 2:**
```
interface eth0
 ip address 10.0.0.2/30
 ip ospf network point-to-point
!
router ospf
 router-id 2.2.2.2
 network 10.0.0.0/30 area 0
 network 192.168.2.0/24 area 0
```

### Use Case 3: EVPN/VXLAN Leaf-Spine Fabric

FRR on each leaf switch running BGP EVPN to distribute MAC/IP bindings across a VXLAN fabric — used in modern datacenter designs.

### Use Case 4: Route Server

FRR as a BGP route server at an IXP (Internet Exchange Point) — receives routes from all members and redistributes without participating in forwarding.

---

## 17. Troubleshooting Cheat Sheet

| Problem | Commands to Run |
|---|---|
| BGP session not coming up | `show bgp summary`, `show bgp neighbors X.X.X.X`, check TCP port 179 |
| OSPF neighbor stuck in EXSTART | Check MTU mismatch (`ip ospf mtu-ignore`), check authentication |
| Route not in kernel | `show ip route`, `ip route show`, check AD/metric conflicts |
| Route in RIB but not FIB | `debug zebra dplane`, `debug zebra kernel`, check `show zebra client summary` |
| Wrong route selected (wrong protocol winning) | `show ip route X.X.X.X` — check AD, adjust with `distance` command |
| Nexthop unreachable, route withdrawn | `show ip nht`, check `debug zebra nht`, verify NHT reachability |
| ECMP not working | Check multipath compile flag `--enable-multipath=N`, `show ip route` for multiple nexthops |
| Route installed but not forwarding | Check `ip_forward` sysctl, check iptables/nftables |
| BFD flapping | Increase timers, check CPU/load |
| LDP session not forming | Check MPLS kernel module, check transport-address reachability |
| MPLS labels not installed in kernel | Check `net.mpls.platform_labels`, `net.mpls.conf.ethX.input`, `show mpls table` |
| VRF route not leaking | Check `import vrf` statements, check RT/RD in BGP |
| Zebra and kernel out of sync | `debug zebra kernel`, check `ip monitor route` for unexpected events |
| FRR daemon crashed | `sudo systemctl status frr`, `journalctl -u frr` |
| ZAPI client not connecting | Check `/var/run/frr/zserv.api` socket exists, check `show zebra client summary` |

### Quick Diagnostic Workflow

```bash
# 1. Are all daemons running?
sudo systemctl status frr
ps aux | grep frr

# 2. Can we reach the peer?
ping 10.0.0.2
traceroute 10.0.0.2

# 3. Is BGP session up?
sudo vtysh -c "show bgp summary"

# 4. Are routes being received?
sudo vtysh -c "show bgp ipv4 unicast"

# 5. Are routes in the kernel?
ip route show

# 6. Is forwarding enabled?
sysctl net.ipv4.ip_forward
```

---

## 18. Key Files & Paths Reference

| Path | Description |
|---|---|
| `/etc/frr/frr.conf` | Main unified configuration file |
| `/etc/frr/daemons` | Which daemons to enable/disable |
| `/etc/frr/vtysh.conf` | vtysh settings |
| `/var/log/frr/` | Log files |
| `/var/run/frr/` | PID files and UNIX sockets |
| `/usr/lib/frr/` | FRR daemon binaries |
| `/usr/bin/vtysh` | The vtysh CLI binary |

### Useful External Resources

- **Official Docs:** https://docs.frrouting.org
- **GitHub:** https://github.com/FRRouting/frr
- **Slack Community:** https://frrouting.slack.com
- **RFC Index for BGP:** RFC 4271, RFC 4760, RFC 7432 (EVPN)
- **RFC for OSPF:** RFC 2328 (v2), RFC 5340 (v3)

---

*Guide covers FRR stable branch (8.x/9.x). Features may vary slightly by version.*
