# PDF 3 — VLAN, VPN, VXLAN

> Full title: "Virtualized Networking for Traffic Engineering and Resource Provisioning"
> All three answer the same question: how do I carve shared physical wires into
> many logical networks?

## The big picture

Three flavors, increasing complexity:

| | Operates at | Solves |
|---|---|---|
| **VLAN** | Layer 2 (Ethernet) | Multiple isolated LANs sharing one switch |
| **VPN** | Layer 3 (IP) | Secure connection between distant networks over the internet |
| **VXLAN** | Layer 2 over Layer 3 | Datacenter-scale virtualization (millions of tenants) |

Common philosophy: **the cable is just a cable; the logical network is software on top of it.** This is the seed of the "softwarization" theme of the whole course.

### Why we needed this at all

The old way: dedicated leased lines between every pair of company sites. Expensive and wasteful. The internet existed but sending corporate traffic over public IP seemed insane. VLAN, VPN, VXLAN are different answers to: *how can we share infrastructure safely?*

---

## VLAN — the layer-2 chopping board

### The problem

One physical switch, two departments (HR, Engineering). You want them isolated as if on two separate switches, without buying two switches.

### The solution

Configure each port as belonging to a "VLAN." The switch refuses to forward traffic between VLANs unless explicitly allowed.

```
       One physical switch (16 ports)
       +--------------------------------+
       | 1  2  3  4  5  6  7  8  9 10...|
       +-+--+--+--+--+--+--+--+--+--+---+
         |  |  |  |  |  |  |  |  |  |
         HR HR HR ENG ENG ENG ENG ENG ENG ENG
         |--VLAN 10--|  |-------VLAN 20--------|
```

Result: physical wiring is shared, but broadcast domains are not. Logical
topology painted on top of physical infrastructure.

### How: IEEE 802.1Q tagging

802.1Q inserts **4 extra bytes** into every Ethernet frame.

Normal frame:
```
+----------+----------+------+--------+--------+
| Dest MAC | Src MAC  | Type | Data   | CRC    |
+----------+----------+------+--------+--------+
```

Tagged frame:
```
+----------+----------+------+------+------+--------+--------+
| Dest MAC | Src MAC  | TPID | TCI  | Type | Data   | CRC    |
+----------+----------+------+------+------+--------+--------+
                       |-VLAN tag (4B)-|
```

- **TPID** = 0x8100, signals "I'm a tagged frame."
- **TCI** = 3 bits priority + 1 bit CFI + **12 bits VLAN ID**.
- 12 bits -> **0 to 4095 VLANs max**.

> The 4096 limit is why **VXLAN** was invented later -- datacenters with
> thousands of customers ran out.

### Two kinds of switch ports

| Type | Behavior | Used for |
|---|---|---|
| **Access port** | One VLAN. Frames leave untagged toward the host. | Host-facing ports (HR's laptop, etc.) |
| **Trunk port** | Carries multiple VLANs. Frames stay tagged. | Switch-to-switch links |

End-host never knows VLANs exist -- the switches do all the tagging/untagging.

### Why VLAN matters for the rest of the course

It's the simplest "virtualize on top of shared hardware" idea. Everything else generalizes it:
- VPN = a VLAN-like thing over the **entire internet**.
- VXLAN = VLAN with **16M slots** instead of 4096, that can travel over routed networks.

---

## VPN — Virtual Private Network

### The problem
Your company has offices in different cities. They need to share an internal network. Options:
1. Lease a dedicated cable between them -> secure but very expensive
2. Use the public internet -> cheap but exposed to eavesdroppers

VPN makes option 2 safe.

### Definition
> A VPN is a type of private network that uses public telecommunication
> (like the internet) instead of leased lines to communicate.

Gives you two things:
1. The "private network" feel (an internal IP in Milan reachable from Rome)
2. Security (encrypted traffic over the public internet)

### The four jobs of every VPN

| Job | Means | Failure mode |
|---|---|---|
| **Authentication** | Prove who the sender is | Random stranger appears to be your CEO |
| **Access control** | Only authorized peers can connect | Strangers join your private network |
| **Confidentiality** | Eavesdroppers see gibberish | Coffee-shop sniffer reads your email |
| **Data integrity** | Detect tampering mid-flight | Attacker changes "€100" to "€10000" |

All four matter. Encryption alone is not enough.

### Tunneling -- THE key mechanic

Take the original IP packet, encrypt it, wrap it in a NEW outer IP packet
that goes from one VPN endpoint to the other.

```
Original packet (private)         Encrypted blob          Outer packet (public)
+---------------------+           +------------+          +------------------+
| src: 10.0.0.5       |  =====>   | [ENCRYPTED |  =====>  | src: 203.0.113.5 |
| dst: 10.0.0.99      |           |   PAYLOAD] |          | dst: 198.51.100.9|
| data: "..."         |           +------------+          | payload: blob    |
+---------------------+                                    +------------------+
```

- Original src/dst IPs are **inside** the encryption -> internet never sees them.
- Eavesdroppers only see "endpoint A talks to endpoint B."
- This pattern is called **encapsulation** or **tunneling**.

> Mental model: the original packet is the *passenger*; the new outer packet
> is the *vehicle*. VXLAN later uses the same trick, just without encryption.

### Protocols by name

| Protocol | One-line | Use today? |
|---|---|---|
| **PPTP** | Microsoft's old VPN | No - broken crypto |
| **L2TP** | Layer-2 tunneling, often + IPsec | Sometimes |
| **IPsec** | The big enterprise standard (L3 + encryption + auth) | Yes - site-to-site corporate |
| **SOCKS** | Old application-level proxy | Rarely |
| **WireGuard** | Modern, simple, fast, runs on UDP | **Yes - used in Kathará lab** |

### Three deployment styles

| Style | Connecting whom | Example |
|---|---|---|
| **Intranet site-to-site** | Two offices of same org | Bank's branch offices |
| **Extranet site-to-site** | Different orgs | Manufacturer <-> supplier |
| **Remote access** | One user <-> corporate | Laptop on hotel WiFi -> HQ |

### Device types

- **Hardware VPN appliance** -- fast, expensive, inflexible
- **Firewall-integrated VPN** -- most common in enterprises
- **Software VPN** -- cheapest, most flexible (Kathará lab uses this)

### Pros / cons

Pros: no leased lines (cost), strong security, scales by adding endpoints.
Cons: depends on internet performance, interop between vendors can be painful,
misconfigured VPN is worse than none.

---

## VXLAN — Virtual eXtensible LAN

### The 3 problems with plain VLAN at cloud scale

1. **Only 4096 VLAN IDs** -- not enough for thousands of cloud tenants.
2. **VLANs are layer 2 only** -- they don't survive crossing a router. Modern
   datacenters route at L3 between racks, so VLANs get stuck on one rack.
3. **VM mobility breaks** -- live-migrating a VM to a different rack puts it on
   a different L3 network. It loses its L2 neighbors.

### VXLAN's answers

| Problem | VXLAN solution |
|---|---|
| 4096 IDs limit | **24-bit VNI** -> ~16 million tenants |
| L2 doesn't cross routers | Encapsulate the L2 frame inside an **IP packet** that can be routed |
| VM mobility breaks | The L2 tunnel follows the VM wherever it goes |

> **One-line catchphrase:** VXLAN is a layer-2 overlay on a layer-3 network.

The L3 routed datacenter is the **underlay**. The virtual L2 segments painted on top are the **overlay**.

### Sidebar: what do "underlay" and "overlay" mean?

- **Underlay** = the real, physical network. Actual wires, switches, IP addresses.
- **Overlay** = a virtual network drawn on top of the underlay. Doesn't physically exist;
  it's an illusion the underlay creates for you.

**Subway analogy:** the Red Line on the map (overlay) doesn't physically exist. Underground there are tangled tunnels, tracks, junctions (underlay). The transit authority paints clean colored lines on top so passengers don't have to think about the actual tracks.

**Networking analogy you already use:** when your browser opens a TCP connection to google.com, you see *one direct pipe* (overlay). Reality is dozens of physical hops through backbone routers, undersea cables, etc. (underlay).

**For VXLAN specifically:**

| | What it looks like |
|---|---|
| **Overlay** (tenant view) | "All my VMs are on one flat LAN, we can ARP each other" |
| **Underlay** (real network) | VMs are scattered across racks on different L3 subnets; VTEPs wrap/unwrap their packets and route them via real routers |

Both are true simultaneously. **VTEPs are the translators** that maintain the illusion.

**The single sentence:** *underlay = reality, overlay = illusion.*

| Underlay | Overlay |
|---|---|
| Physical Ethernet wires | VLAN segments |
| The public internet | A VPN tunnel |
| The L3 datacenter fabric | VXLAN segments |
| Hop-by-hop IP routing | A TCP connection |

### Packet format

Same passenger/vehicle pattern as VPN, no encryption.

```
+-----+-----+-----+--------+----------------------+
|Outer|Outer| UDP | VXLAN  |   Original L2 frame  |
| MAC | IP  | hdr |  hdr   |   (the passenger)    |
+-----+-----+-----+--------+----------------------+
 14B   20B    8B    8B          variable
 |---------- 50 bytes overhead --------|
```

- Outer MAC: cross the local rack's switch
- Outer IP: addresses one VTEP to another, routable
- UDP: chosen for ECMP load-spreading (no connection state)
- VXLAN header: holds the **24-bit VNI** (tenant ID)

**Total overhead: ~50 bytes per packet.** Important for small frames.

**Why UDP, not TCP?** No connection state needed, and routers use UDP source ports to spread flows across parallel paths (ECMP).

### VTEP — VXLAN Tunnel Endpoint

The device that does the wrapping/unwrapping. Has **two interfaces**:

```
   Tenant VMs --- [VTEP local LAN side]                  [VTEP IP side] --- L3 underlay
                  (speaks plain Ethernet)                (has its own IP)
```

Two jobs:
1. **Encap** (egress): wrap incoming Ethernet frame in VXLAN/UDP/IP/MAC, ship to destination VTEP.
2. **Decap** (ingress): strip outer wrappers, hand the original frame to local VM.

Where it runs:
- Software VTEP inside a hypervisor (most common)
- Hardware VTEP on a top-of-rack switch (faster)
- Dedicated VXLAN gateway box

> Key property: **the L3 underlay doesn't know or care about VXLAN.** It just routes UDP packets. Independence between overlay and underlay is what makes this scale.

---

## VXLAN packet walkthrough (the key example)

### Setup
Overlay view:
```
Host-A: 10.1.1.100, MAC-A, VNI 864
Host-B: 10.1.1.101, MAC-B, VNI 864
```
From the hosts' perspective, they're on the same flat LAN.

Underlay reality:
```
VTEP-1 (hosts A's server): IP 165.123.1.1, MAC-1
VTEP-2 (hosts B's server): IP 140.123.1.1, MAC-4
L3 routed network between them.
```

### Step 1: Host-A builds a normal Ethernet frame
```
Eth: MAC src=MAC-A, MAC dst=MAC-B
IP:  src=10.1.1.100, dst=10.1.1.101
```

### Step 2: VTEP-1 wraps it (encapsulation)
```
Outer MAC: src=MAC-1, dst=Router1 MAC
Outer IP:  src=165.123.1.1, dst=140.123.1.1
UDP + VXLAN(VNI=864) + [original frame as payload]
```

### Step 3: Packet rides the underlay
- The OUTER packet behaves like any normal IP packet.
- Routers rewrite outer MAC at every hop, outer IP stays the same.
- Same A->R->B pattern from the TCP/IP lecture, but on the OUTER packet.
- Routers don't know or care about VXLAN.

### Step 4: VTEP-2 unwraps (decapsulation)
- Outer IP destination is VTEP-2 -> accept.
- VXLAN header says VNI 864 -> tenant Blue.
- Strip outer MAC/IP/UDP/VXLAN. Inner frame is intact.

### Step 5: Local delivery
- Inner destination MAC = MAC-B.
- VTEP-2 hands the frame to Host-B over the local virtual switch.
- Host-B never knew anything unusual happened.

### What changed vs didn't

| | Egress (VTEP-1) | On underlay | Ingress (VTEP-2) |
|---|---|---|---|
| Inner MAC/IP | MAC-A/MAC-B, 10.1.1.100/101 | (hidden) | unchanged |
| Outer MAC | rewritten every hop | rewritten every hop | rewritten last hop |
| Outer IP | 165.../140... | unchanged | unchanged |
| VNI | 864 | 864 (opaque) | 864 |

> **The inner addresses never change.** That's the whole illusion.

---

## VXLAN <-> VLAN mapping

Reality: most datacenters didn't drop VLAN overnight. A **VXLAN gateway** (a VTEP that also speaks VLAN) translates between them.

Simple lookup table:

| VLAN ID | VXLAN ID (VNI) |
|---|---|
| 10 | 1010 |
| 20 | 1020 |

This lets legacy racks (VLAN) and modern racks (VXLAN) coexist as a single logical network. Critical for incremental migration.

---

## VXLAN use cases

| Use case | Why VXLAN |
|---|---|
| Cloud hosting provider | Each customer gets own L2 segment past the 4096 VLAN limit |
| VM farm / private cloud | Live-migrate VMs across racks without losing L2 context |
| Cross-DC connectivity | Make two distant DCs look like one giant L2 network |
| Multi-tenant DCs | Tenants get isolation + flexibility to use any IP scheme |

One-liner job description from the slides: *"connect two or more L3 network domains and make them appear as a single L2 domain."*

---

## Datacenter fabric: spine-leaf

Modern DCs using VXLAN almost always use spine-leaf topology for the underlay.

```
   Spine: ┌──┐ ┌──┐ ┌──┐ ┌──┐    (high-speed routers)
           ╲ ╱ ╲ ╱ ╲ ╱ ╲ ╱
            ╳   ╳   ╳   ╳        (every leaf to every spine)
           ╱ ╲ ╱ ╲ ╱ ╲ ╱ ╲
   Leaf:  ┌──┐ ┌──┐ ┌──┐ ┌──┐    (top-of-rack, run VTEPs)
            │    │    │    │
          Rack Rack Rack Rack
```

Properties:
- Many parallel paths between any two racks.
- Failures don't take much down.
- Grow horizontally by adding leaves (or spines for more bandwidth).

VXLAN sits as an overlay on top. Leaves are usually where VTEPs run. Spines just route IP packets — they don't know VXLAN exists.

---

# PDF 3 Final Recap

| | Operates at | Solves | Mechanism |
|---|---|---|---|
| **VLAN** | L2 | Multi-tenant on one switch | 4-byte 802.1Q tag (12-bit VLAN ID) |
| **VPN** | L3 | Secure connection over the internet | Encrypted tunneling |
| **VXLAN** | L2 over L3 | Datacenter-scale multi-tenancy | MAC-in-UDP encapsulation, 24-bit VNI |

**Three threads that run through everything:**

1. **Encapsulation/tunneling** (passenger inside vehicle) is the universal mechanic.
2. **Overlay / underlay** separates logical view from physical view.
3. **Softwarization** — implemented in software in hypervisors, OS kernels, switch firmware.

These themes carry directly into SDN (PDF 4) and the virtualization lecture (PDF 6).
