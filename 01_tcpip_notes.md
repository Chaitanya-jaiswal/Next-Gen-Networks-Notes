# PDF 2 — TCP/IP Recap

> Segata calls this "extremely important." Everything else in the course assumes
> these concepts are solid.

This PDF has 5 themes:

1. The 5-layer TCP/IP model
2. How a host joins a network (DHCP, ARP, cross-LAN routing)
3. Wireless & mobile network basics
4. How we test/measure networks
5. Key performance indicators

---

## Theme 1: The 5-layer TCP/IP Model

```
+-----------------+
|   Application   |  HTTP, SMTP, FTP, DNS  -- user's program
+-----------------+
|   Transport     |  TCP (reliable), UDP (fast), QUIC
+-----------------+
|    Network      |  IP -- best-effort packet delivery
+-----------------+
|      Link       |  Ethernet, WiFi -- local hop
+-----------------+
|    Physical     |  the actual electrical/radio waveform
+-----------------+
```

**Core principle:** each layer has one job. The layer above doesn't care how
that job is done. That's why your Python TCP socket doesn't care if it's
flowing over fiber, copper, or 5G.

### Application layer
**Job:** be the user-facing program. HTTP, SMTP, FTP, DNS.

### Transport layer
**Job:** get data from one *process* to another, end-to-end. Uses **port numbers**.

| | TCP | UDP |
|---|---|---|
| Reliable? | Yes (retransmits) | No |
| Ordered? | Yes | No |
| Congestion control? | Yes | No |
| Speed | Slower | Faster |
| Use cases | Web, email, file transfer | DNS, video calls, gaming |
| Share of internet | ~90% | the rest |

**TCP flavors to know by name:**
- **TCP New Reno** — the classic; what textbooks describe.
- **TCP CUBIC** — Linux default; more aggressive on fast networks.
- **TCP BBR** — Google; doesn't use packet loss as the congestion signal.

**QUIC** — newer; runs on UDP, adds reliability + encryption + faster handshake.
HTTP/3 uses QUIC. Saves a lot of round-trips vs. TCP+TLS.

### Network layer
**Job:** move packets hop by hop across networks, anywhere on Earth.

Protocol: **IP** (v4 or v6). Traits:
- **Datagram** model — every packet independent, no "connection."
- **Best-effort** — no guarantees. Packets can be lost, reordered, or corrupted.
- **Works over anything** — fiber, WiFi, satellite. The "narrow waist" of the internet.

If you want guarantees, that's TCP's job above.

### Link layer
**Job:** move a frame across **one hop** — between two directly connected devices.

Two sub-jobs:
- **LLC** — flow & error checking (mostly invisible)
- **MAC** — who can talk on the shared medium, plus addressing via **MAC addresses**

> Crucial fact: MAC addresses only matter on a local segment.
> They don't survive crossing a router.

### Physical layer
**Job:** turn bits into actual signals (voltage, light, radio) and back.

Dominant technique today: **OFDM**. Used by WiFi, 4G/5G, DVB-T.

> **Course-relevant:** the physical and MAC layers can be modified without
> changing anything above them. This is exactly what **SDR** exploits — replace
> the physical layer with software, the rest of the stack doesn't notice.

---

## Theme 2 (part A): IP addressing, switches/routers, DHCP, ARP

### IPv4 addressing recap

```
IP Address:   192 . 168 . 1 . 50
Subnet Mask:  255 . 255 . 255 . 0
              |----network----| |host|
```

- The mask splits the address into **network bits** and **host bits**.
- `255.255.255.0` = `/24` = first 24 bits are the network.
- Two reserved hosts in every subnet:
  - `.0` = network address (refers to the whole subnet)
  - `.255` = broadcast address (sends to everyone in the subnet)

**Why it matters:** every device applies its mask to a destination IP and asks
"is this local or remote?" That decision drives whether to deliver directly or
hand off to a router.

### Switch vs Router

| | Switch | Router |
|---|---|---|
| Layer | 2 (Link / MAC) | 3 (Network / IP) |
| What does it look at? | **MAC** addresses | **IP** addresses |
| Scope | Inside one local network | Connects different networks |
| Speed | Very fast (hardware) | Slower (more thinking) |
| Cost per port | Cheap | Expensive |

- A **switch** is like a smart power strip: it learns which MAC is on which port.
  Unknown destination MAC -> flood; known -> filter (send only to right port).
- A **router** has a **routing table**: "network X is reachable via next-hop Y."

Modern enterprise gear often blurs the line, but keep them conceptually separate.

### DHCP — how a host joins a network

A freshly plugged-in host needs 4 things:

1. An **IP address**
2. A **default gateway** (the router to reach the rest of the world)
3. A **subnet mask**
4. A **DNS server**

The exchange is the **DORA** dance:

```
Host                                   DHCP server
  |-- Discover --"Anyone got an IP?" ----->|   (broadcast)
  |<- Offer    --"Here's 192.168.1.50"-----|
  |-- Request  --"OK, I'll take it" ------>|
  |<- Ack      --"Granted"-----------------|
```

> First message is a **broadcast** — the host has no IP yet, so it shouts to
> everyone. This is one of the legitimate uses of `255.255.255.255`.

### ARP — finding the next-hop MAC

You have an IP. You want to talk to `192.168.1.99` on your LAN. But the link
layer speaks **MAC**, not IP. You need to map the IP to a MAC.

```
Your laptop                       Everyone on the LAN
  |-- ARP Request --"Who has .99?" ----->|   (broadcast)
  |                                       |  most ignore
  |<- ARP Reply --"That's me, MAC=88:..."| (only .99 replies)
```

The result is cached in your **ARP table** so you don't broadcast every time.

### Putting it together — the sequence

1. **DHCP** gets you IP, mask, gateway, DNS.
2. For every packet, ask: "Is the destination on my local subnet?"
   - **Yes** -> ARP for destination's MAC, send directly.
   - **No**  -> ARP for **gateway's** MAC, send to the gateway and let it handle it.

This last bullet is the bridge to the cross-LAN walkthrough (Part 3, coming next session).

---

## Still pending (will be added as we cover them)

- Theme 2 (part B) — Cross-LAN packet walkthrough (the A -> R -> B example, slides 22-26)
- Theme 3 — Wireless & mobile network basics (RAN, Core, standards)
- Theme 4 — How we test/measure networks (analytical, simulation, emulation, measurement)
- Theme 5 — KPIs (throughput, goodput, latency, jitter, QoE, MOS)
