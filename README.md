# Enterprise Network Design — Cisco Packet Tracer

> **VLAN Segmentation · Inter-VLAN Routing · Centralised DHCP/DNS · OSPF Dynamic Routing**

A hands-on simulation of a two-site enterprise network built in **Cisco Packet Tracer**.
The project covers every major layer of a real SMB deployment: layer-2 segmentation, layer-3 routing,
centralised services, and WAN connectivity — all verified with live pings and `show` commands.

---

## Network Topology

![Full network topology](topology-overview.png)

| Site | Devices |
|------|---------|
| **Headquarters (HQ)** | Router1 (Cisco 2911), Switch0 (Catalyst 2960-24TT), PC1 · PC2 · PC3, Server0 |
| **Branch Office** | Router2 (Cisco 2911), Switch2 (Catalyst 2960-24TT), PC-Branch |
| **WAN Link** | Ethernet cross-connect — `10.0.0.0/30` |

---

## VLAN Design (HQ — Switch0)

VLANs isolate department traffic at layer 2.
Each VLAN maps to a dedicated switch access port and a `/24` subnet.

| VLAN ID | Name   | Subnet            | Switch Port | Gateway      |
|---------|--------|-------------------|-------------|--------------|
| 10      | HR     | 192.168.10.0/24   | Fa0/1       | 192.168.10.1 |
| 20      | SALES  | 192.168.20.0/24   | Fa0/2       | 192.168.20.1 |
| 30      | IT     | 192.168.30.0/24   | Fa0/3       | 192.168.30.1 |
| 50      | SERVER | 192.168.50.0/24   | Fa0/4       | 192.168.50.1 |

**Verified with `show vlan brief`:**

![VLAN configuration — show vlan brief output](screenshots/vlan-config.png)

### Switch Access Port Configuration

```
Switch0(config)# vlan 10
Switch0(config-vlan)# name HR
Switch0(config)# vlan 20
Switch0(config-vlan)# name SALES
Switch0(config)# vlan 30
Switch0(config-vlan)# name IT
Switch0(config)# vlan 50
Switch0(config-vlan)# name SERVER

Switch0(config)# interface FastEthernet0/1
Switch0(config-if)# switchport mode access
Switch0(config-if)# switchport access vlan 10

Switch0(config)# interface FastEthernet0/2
Switch0(config-if)# switchport mode access
Switch0(config-if)# switchport access vlan 20

Switch0(config)# interface FastEthernet0/3
Switch0(config-if)# switchport mode access
Switch0(config-if)# switchport access vlan 30

Switch0(config)# interface FastEthernet0/4
Switch0(config-if)# switchport mode access
Switch0(config-if)# switchport access vlan 50
```

### Trunk Uplink to Router1

The uplink from Switch0 to Router1 is configured as an **802.1Q trunk** to carry all four VLANs over a single cable:

```
Switch0(config)# interface GigabitEthernet0/1
Switch0(config-if)# switchport mode trunk
```

---

## Inter-VLAN Routing — Router-on-a-Stick (Router1)

Router1's `GigabitEthernet0/0` carries all VLAN traffic via **dot1Q subinterfaces**.
Each subinterface acts as the default gateway for its VLAN.

```
Router1(config)# interface GigabitEthernet0/0
Router1(config-if)# no shutdown

Router1(config)# interface GigabitEthernet0/0.10
Router1(config-subif)# encapsulation dot1Q 10
Router1(config-subif)# ip address 192.168.10.1 255.255.255.0

Router1(config)# interface GigabitEthernet0/0.20
Router1(config-subif)# encapsulation dot1Q 20
Router1(config-subif)# ip address 192.168.20.1 255.255.255.0

Router1(config)# interface GigabitEthernet0/0.30
Router1(config-subif)# encapsulation dot1Q 30
Router1(config-subif)# ip address 192.168.30.1 255.255.255.0

Router1(config)# interface GigabitEthernet0/0.50
Router1(config-subif)# encapsulation dot1Q 50
Router1(config-subif)# ip address 192.168.50.1 255.255.255.0
```

---

## DHCP — Centralised IP Assignment (Server0)

**Server0** (`192.168.50.10`) runs the DHCP service for all HQ VLANs.
Router1 subinterfaces relay DHCP broadcasts from each VLAN to the server using `ip helper-address`.

### DHCP Pools on Server0

| Pool Name  | Default Gateway | DNS Server    | Start IP        | Max Users |
|------------|-----------------|---------------|-----------------|-----------|
| HR         | 192.168.10.1    | 192.168.50.10 | 192.168.10.100  | 50        |
| SALES      | 192.168.20.1    | 192.168.50.10 | 192.168.20.100  | 50        |
| IT         | 192.168.30.1    | 192.168.50.10 | 192.168.30.100  | 50        |
| serverPool | —               | —             | 192.168.50.0    | 512       |

![DHCP pool configuration on Server0](screenshots/dhcp-config.png)

### DHCP Relay (ip helper-address)

Each Router1 subinterface forwards DHCP broadcasts to Server0:

```
Router1(config)# interface GigabitEthernet0/0.10
Router1(config-subif)# ip helper-address 192.168.50.10

Router1(config)# interface GigabitEthernet0/0.20
Router1(config-subif)# ip helper-address 192.168.50.10

Router1(config)# interface GigabitEthernet0/0.30
Router1(config-subif)# ip helper-address 192.168.50.10
```

---

## DNS — Internal Name Resolution (Server0)

DNS is co-hosted on **Server0** (`192.168.50.10`).
Clients in all VLANs automatically receive this DNS server address via DHCP.

| No. | Hostname           | Type     | Resolves To   |
|-----|--------------------|----------|---------------|
| 0   | www.company.local  | A Record | 192.168.50.10 |

![DNS A record configuration on Server0](screenshots/dns-config.png)

> Additional records (`dns.company.local`, `files.company.local`) can be added in the Server0 DNS tab pointing to this or future dedicated hosts.

---

## OSPF — Dynamic Routing Between Sites

OSPF (Process 1, Area 0) dynamically advertises all subnets between Router1 and Router2 across the WAN link.
No static routes are needed — both routers learn all remote networks automatically.

### WAN Link

| Device  | Interface          | IP Address |
|---------|--------------------|------------|
| Router1 | GigabitEthernet0/1 | 10.0.0.1   |
| Router2 | GigabitEthernet0/0 | 10.0.0.2   |

**Subnet:** `10.0.0.0/30`

### Router1 OSPF Configuration

```
Router1(config)# interface GigabitEthernet0/1
Router1(config-if)# ip address 10.0.0.1 255.255.255.252
Router1(config-if)# no shutdown

Router1(config)# router ospf 1
Router1(config-router)# network 192.168.10.0 0.0.0.255 area 0
Router1(config-router)# network 192.168.20.0 0.0.0.255 area 0
Router1(config-router)# network 192.168.30.0 0.0.0.255 area 0
Router1(config-router)# network 192.168.50.0 0.0.0.255 area 0
Router1(config-router)# network 10.0.0.0 0.0.0.3 area 0
```

### Router2 OSPF Configuration

```
Router2(config)# interface GigabitEthernet0/0
Router2(config-if)# ip address 10.0.0.2 255.255.255.252
Router2(config-if)# no shutdown

Router2(config)# interface GigabitEthernet0/1
Router2(config-if)# ip address 192.168.60.1 255.255.255.0
Router2(config-if)# no shutdown

Router2(config)# router ospf 1
Router2(config-router)# network 192.168.60.0 0.0.0.255 area 0
Router2(config-router)# network 10.0.0.0 0.0.0.3 area 0
```

### Branch Office

| Device    | Interface          | IP Address    |
|-----------|--------------------|---------------|
| Router2   | GigabitEthernet0/1 | 192.168.60.1  |
| PC-Branch | NIC                | 192.168.60.10 |

### OSPF Neighbour — Verified FULL/DR

![OSPF neighbor adjacency — FULL/DR state](screenshots/ospf-neighbor.png)

```
Router> show ip ospf neighbor

Neighbor ID     Pri   State      Dead Time   Address    Interface
192.168.60.1      1   FULL/DR    00:00:36    10.0.0.2   GigabitEthernet0/1
```

Router2 achieved **FULL/DR** state — LSA exchange is complete and the adjacency is stable.

---

## Complete IP Addressing Table

| Device      | Interface    | IP Address    | Role                 |
|-------------|--------------|---------------|----------------------|
| Router1     | G0/0.10      | 192.168.10.1  | HR VLAN gateway      |
| Router1     | G0/0.20      | 192.168.20.1  | SALES VLAN gateway   |
| Router1     | G0/0.30      | 192.168.30.1  | IT VLAN gateway      |
| Router1     | G0/0.50      | 192.168.50.1  | SERVER VLAN gateway  |
| Router1     | G0/1         | 10.0.0.1      | WAN to Router2       |
| Router2     | G0/0         | 10.0.0.2      | WAN to Router1       |
| Router2     | G0/1         | 192.168.60.1  | Branch LAN gateway   |
| Server0     | FastEthernet0 | 192.168.50.10 | DHCP + DNS server   |
| PC1 (HR)    | NIC          | 192.168.10.100+ | DHCP assigned      |
| PC2 (SALES) | NIC          | 192.168.20.100+ | DHCP assigned      |
| PC3 (IT)    | NIC          | 192.168.30.100+ | DHCP assigned      |
| PC-Branch   | NIC          | 192.168.60.10 | Static               |

---

## Verification & Test Results

### 1 · Cross-VLAN Ping (HQ Internal)

A ping from PC1 (HR — 192.168.10.x) to PC2 (SALES — 192.168.20.x) passes through Router1's subinterfaces, confirming Inter-VLAN Routing is working correctly.

### 2 · End-to-End HQ → Branch Ping

![Ping from HQ PC to Branch PC — 192.168.60.10](screenshots/ping-test.png)

```
C:\>ping 192.168.60.10

Pinging 192.168.60.10 with 32 bytes of data:
Request timed out.                           <- ARP resolution (normal on first packet)
Reply from 192.168.60.10: bytes=32 time=10ms TTL=126
Reply from 192.168.60.10: bytes=32 time=12ms TTL=126
Reply from 192.168.60.10: bytes=32 time=10ms TTL=126

Packets: Sent=4, Received=3, Lost=1 (25% loss)
Minimum=10ms, Maximum=12ms, Average=10ms
```

> The first packet times out while ARP resolves the next-hop MAC — **this is expected behaviour in Packet Tracer**.
> The 3 successful replies (TTL=126, 2 hops decremented) confirm full end-to-end OSPF-routed connectivity.

### 3 · OSPF Neighbour State

Confirmed `FULL/DR` — both routers have exchanged all LSAs and maintain a stable adjacency. (See screenshot above.)

### 4 · DNS Resolution

`www.company.local` resolves to `192.168.50.10` via the internal DNS server on Server0.
Any HQ VLAN client that receives an IP via DHCP inherits `192.168.50.10` as its DNS server automatically.

---

## Technologies & Key Cisco Commands

| Technology | Purpose | Key Commands |
|------------|---------|-------------|
| **VLAN** | Layer-2 traffic segmentation | `vlan <id>`, `switchport access vlan <id>` |
| **802.1Q Trunking** | Carry multiple VLANs on one uplink | `switchport mode trunk` |
| **Router-on-a-Stick** | Inter-VLAN routing via subinterfaces | `encapsulation dot1Q <id>`, `ip address` |
| **DHCP Relay** | Forward DHCP broadcasts to central server | `ip helper-address <server-ip>` |
| **DHCP Server** | Centralised IP assignment | Packet Tracer Server GUI → DHCP tab |
| **DNS** | Internal hostname resolution | Packet Tracer Server GUI → DNS tab |
| **OSPF** | Dynamic WAN routing (Area 0) | `router ospf 1`, `network <subnet> <wildcard> area 0` |
| **Verification** | Check routing table / OSPF state | `show ip route`, `show ip ospf neighbor`, `show vlan brief` |

---

## How to Open the Simulation

1. Download and install [Cisco Packet Tracer](https://www.netacad.com/courses/packet-tracer) (free with a NetAcad account — v8.x recommended).
2. Clone or download this repository.
3. Open **`design1-co.pkt`** in Packet Tracer.
4. Use **Realtime Mode** to run pings and verify connectivity, or switch to **Simulation Mode** to step through packet flows frame by frame.

---

## Troubleshooting Guide

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| PCs in the same VLAN can't ping each other | Access port not assigned to correct VLAN | `switchport access vlan <id>` on the port |
| Inter-VLAN ping fails | Trunk misconfigured or subinterface missing | Run `show interfaces trunk` on Switch0 |
| DHCP address not assigned | `ip helper-address` missing on subinterface | Add `ip helper-address 192.168.50.10` to each VLAN subinterface |
| OSPF neighbour stuck in INIT/2-WAY | Mismatched hello/dead timers or area IDs | Check `show ip ospf interface` on both routers |
| Branch PC unreachable from HQ | OSPF not advertising the branch subnet | Verify `network 192.168.60.0 0.0.0.255 area 0` on Router2 |
| DNS resolution fails | Client not using 192.168.50.10 as DNS | Verify DHCP pool DNS Server field = `192.168.50.10` |

---

## Author

**Amin Badparvar**  
Computer Engineering Student
