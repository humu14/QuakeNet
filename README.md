# QuakeNet — Earthquake Early Warning & Emergency Response Network

A multi-site enterprise network for a national earthquake early-warning and emergency-response system, designed and simulated in **Cisco Packet Tracer**. This is my own project, built for the CSE421 (Computer Networks) lab.

The network connects seven emergency-response units through a mix of fibre, Ethernet backbone, and serial WAN links, combining **VLSM addressing, RIPv2, static and floating static routing, DHCP with relay, DNS, web, and email services**, with tested failover paths.

---

## Highlights

- **VLSM** addressing of `16.83.0.0/16` across 7 LANs and 7 point-to-point/backbone links with zero overlap
- **Hybrid routing**: RIPv2 on the warning loop, static routes elsewhere, redistribution of static into RIP at the core
- **Redundancy**: recursive backup route at MEC and floating static route at CRC (AD 100)
- **No default routes** anywhere; every destination is explicitly routed
- **Centralised DHCP** with relay for remote LANs
- **DNS-based services**: 2 web servers, 7 mail servers, cross-domain email
- 7 routers, 10 servers, 7 network printers, 14 DHCP clients

---

## Network Units

| Unit | Required hosts | Connection |
|------|---------------:|------------|
| NEOC | 5000 | Core: backbone, fibre to MEC, warning loop |
| USR  | 1040 | Stub behind MEC (serial) |
| MEC  | 500 | Backbone, fibre to NEOC |
| FRH  | 320 | Backbone, warning loop |
| SMS  | 180 | Warning loop |
| CRC  | 100 | Backbone |
| MOP  | 50 | Stub behind CRC (serial) |

---

## Topology

```
  Central backbone switch (16.83.46.0/29): NEOC, FRH, MEC, CRC

            SMS
           /   \
     serial     serial            Warning loop (RIPv2)
         /       \
     NEOC ─serial─ FRH
       ║
     fibre
       ║
      MEC ─serial─ USR            Stub site (static)

      CRC ─serial─ MOP            Stub site (static)
```

---

## VLSM Plan

Allocated largest block first from `16.83.0.0/16`.

| Unit | Hosts | Network | Mask | Usable range | Broadcast |
|------|------:|---------|------|--------------|-----------|
| NEOC | 5000 | 16.83.0.0/19 | 255.255.224.0 | 16.83.0.1 – 16.83.31.254 | 16.83.31.255 |
| USR | 1040 | 16.83.32.0/21 | 255.255.248.0 | 16.83.32.1 – 16.83.39.254 | 16.83.39.255 |
| MEC | 500 | 16.83.40.0/23 | 255.255.254.0 | 16.83.40.1 – 16.83.41.254 | 16.83.41.255 |
| FRH | 320 | 16.83.42.0/23 | 255.255.254.0 | 16.83.42.1 – 16.83.43.254 | 16.83.43.255 |
| SMS | 180 | 16.83.44.0/24 | 255.255.255.0 | 16.83.44.1 – 16.83.44.254 | 16.83.44.255 |
| CRC | 100 | 16.83.45.0/25 | 255.255.255.128 | 16.83.45.1 – 16.83.45.126 | 16.83.45.127 |
| MOP | 50 | 16.83.45.128/26 | 255.255.255.192 | 16.83.45.129 – 16.83.45.190 | 16.83.45.191 |

**Links**

| Subnet | Purpose |
|--------|---------|
| 16.83.46.0/29 | Central backbone (NEOC, FRH, MEC, CRC) |
| 16.83.46.8/30 | NEOC – MEC (fibre) |
| 16.83.46.12/30 | NEOC – SMS (warning loop) |
| 16.83.46.16/30 | SMS – FRH (warning loop) |
| 16.83.46.20/30 | FRH – NEOC (warning loop) |
| 16.83.46.24/30 | CRC – MOP |
| 16.83.46.28/30 | MEC – USR |

---

## Router Interfaces

| Router | Interface | IP address | Connected to |
|--------|-----------|------------|--------------|
| NEOC | Gi0/0 | 16.83.0.1/19 | NEOC LAN |
| | Gi0/1 | 16.83.46.1/29 | Backbone |
| | Gi0/2 | 16.83.46.9/30 | MEC (fibre) |
| | Se0/3/0 | 16.83.46.13/30 | SMS |
| | Se0/3/1 | 16.83.46.21/30 | FRH |
| SMS | Gi0/0 | 16.83.44.1/24 | SMS LAN |
| | Se0/3/0 | 16.83.46.14/30 | NEOC |
| | Se0/3/1 | 16.83.46.17/30 | FRH |
| FRH | Gi0/0 | 16.83.42.1/23 | FRH LAN |
| | Gi0/1 | 16.83.46.2/29 | Backbone |
| | Se0/3/0 | 16.83.46.18/30 | SMS |
| | Se0/3/1 | 16.83.46.22/30 | NEOC |
| MEC | Gi0/0 | 16.83.40.1/23 | MEC LAN |
| | Gi0/1 | 16.83.46.3/29 | Backbone |
| | Gi0/2 | 16.83.46.10/30 | NEOC (fibre) |
| | Se0/3/0 | 16.83.46.29/30 | USR |
| CRC | Gi0/0 | 16.83.45.1/25 | CRC LAN |
| | Gi0/1 | 16.83.46.4/29 | Backbone |
| | Se0/3/0 | 16.83.46.25/30 | MOP |
| USR | Gi0/0 | 16.83.32.1/21 | USR LAN |
| | Se0/3/0 | 16.83.46.30/30 | MEC |
| MOP | Gi0/0 | 16.83.45.129/26 | MOP LAN |
| | Se0/3/0 | 16.83.46.26/30 | CRC |

DCE clock rate `64000` on NEOC Se0/3/0, NEOC Se0/3/1, SMS Se0/3/1, MEC Se0/3/0, CRC Se0/3/0.

---

## Routing Design

| Router | Method |
|--------|--------|
| NEOC, SMS, FRH | RIPv2 (`no auto-summary`) on the warning loop; NEOC redistributes its static routes into RIP (metric 5) |
| MEC | Static routes, with **recursive backup** toward NEOC via CRC at AD 100 |
| CRC | Static routes, with **floating static** toward NEOC via FRH at AD 100 |
| USR, MOP | Stub routers with exit-interface static routes (`Serial0/3/0`) |

LAN and backbone interfaces are passive in RIP. No router carries a default route.

**MEC — recursive backup**
```
ip route 16.83.0.0 255.255.224.0 16.83.46.9          ! primary over fibre
ip route 16.83.0.0 255.255.224.0 16.83.46.4 100      ! backup via CRC
```

**CRC — floating static**
```
ip route 16.83.0.0 255.255.224.0 16.83.46.1          ! primary to NEOC
ip route 16.83.0.0 255.255.224.0 16.83.46.2 100      ! floating via FRH
```

Full configurations for all seven routers are in [`configs/`](configs/).

---

## DHCP

| LAN | Served by | Method |
|-----|-----------|--------|
| NEOC | NEOC | Local pool |
| SMS | NEOC | Relay (`ip helper-address 16.83.46.13`) |
| FRH | NEOC | Relay (`ip helper-address 16.83.46.21`) |
| MEC | MEC | Local pool |
| USR | MEC | Relay (`ip helper-address 16.83.46.29`) |
| CRC | CRC | Local pool |
| MOP | CRC | Relay (`ip helper-address 16.83.46.25`) |

The first addresses of each subnet are excluded for gateways, servers and printers. All pools hand out DNS `16.83.0.2`.

---

## Services

**DNS** — `16.83.0.2` (NEOC), resolving all web and mail names.

**Web servers**

| Server | IP | URL |
|--------|----|-----|
| WEB-NEOC | 16.83.0.4 | www.neoc.alert.gov |
| WEB-MEC | 16.83.40.3 | www.mec.alert.gov |

**Mail servers** (SMTP + POP3)

| Server | IP | Domain |
|--------|----|--------|
| MAIL-NEOC | 16.83.0.3 | neoc.gov |
| MAIL-MEC | 16.83.40.2 | mec.gov |
| MAIL-SMS | 16.83.44.2 | sms.gov |
| MAIL-FRH | 16.83.42.2 | frh.gov |
| MAIL-USR | 16.83.32.2 | usr.gov |
| MAIL-CRC | 16.83.45.2 | crc.gov |
| MAIL-MOP | 16.83.45.130 | mop.gov |

Cross-domain email is set up between `neoc.gov` and `mec.gov`. Each unit also has a network printer with a static address.

---

## Verification

- ✅ All interfaces up/up on every router
- ✅ `show ip route | include 0.0.0.0` returns nothing on all seven routers
- ✅ End-to-end ping from the farthest stub (MOP) to every unit gateway, and from NEOC to the MOP mail server
- ✅ All 14 PCs receive correct addresses via local DHCP or relay
- ✅ **MEC failover**: shutting the fibre (`Gi0/2`) swaps the NEOC route from AD 1 via `16.83.46.9` to AD 100 via `16.83.46.4`; connectivity holds
- ✅ **CRC failover**: with primary routes removed, the floating route installs as `[100/0] via 16.83.46.2` and traffic continues
- ✅ Cross-domain email send/receive/reply between NEOC and MEC
- ✅ Web access by hostname from remote LANs (MOP → NEOC site, USR → MEC site)

> **Design note:** CRC's primary and floating next hops share the same backbone switch, so shutting CRC's backbone interface drops both. The floating route is therefore demonstrated by removing the primary routes rather than an interface.

---

## Repository Structure

```
QuakeNet/
├── QuakeNet.pkt          # Packet Tracer topology
├── configs/              # Router configurations
│   ├── NEOC.txt
│   ├── SMS.txt
│   ├── FRH.txt
│   ├── MEC.txt
│   ├── CRC.txt
│   ├── USR.txt
│   └── MOP.txt
├── report/               # Project report
└── README.md
```

## Getting Started

1. Install [Cisco Packet Tracer](https://www.netacad.com/courses/packet-tracer) (8.x or later).
2. Open `QuakeNet.pkt`.
3. Use simulation mode or the PC command prompt (`ping`, `ipconfig /all`) to explore.
4. To rebuild from scratch, paste each file in `configs/` into the matching router's CLI.

---

## Tech

Cisco Packet Tracer · Cisco IOS · VLSM · RIPv2 · Static / Floating Static Routing · DHCP Relay · DNS · HTTP · SMTP / POP3
