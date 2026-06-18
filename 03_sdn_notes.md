# PDF 4 — SDN (Software Defined Networking)

> The single most important PDF in the course. Most exam questions and 3 of the
> project options come from here.

## The fundamental distinction: data plane vs control plane

Every router/switch does **two completely different jobs**:

**Data plane (forwarding plane):**
- "Packet arrived. Look at it, send it out the right port. Fast."
- Done in dedicated chips. Microseconds per packet. The muscle.

**Control plane:**
- "How do I know which port to send it out? Build a forwarding table by talking
  to neighbors, running shortest-path algorithms, reacting to failures."
- Done in software. Slow and thoughtful. The brain.

```
+----------------------------------+
|              ROUTER              |
|  +-----------------------+       |
|  | Control plane (brain) |       |
|  |  - OSPF, BGP, etc.    |       |
|  |  - builds the table   |       |
|  +-----------+-----------+       |
|              | writes            |
|              v                   |
|  +-----------------------+       |
|  | Data plane (muscle)   |       |
|  | Forwarding table:     |       |
|  |  "if dst=X -> port=3" |       |
|  +-----------------------+       |
+----------------------------------+
```

**Postal analogy:**
- Data plane = workers sorting mail into pigeonholes by ZIP code.
- Control plane = the planning dept deciding which ZIPs go through which hub.

---

## Traditional networking (the per-router model)

Every router runs BOTH jobs in one box. Routers gossip with neighbors via
OSPF/BGP/RIP to figure out the topology independently.

Problems:
1. **Vendor lock-in.** Cisco OS only runs on Cisco gear. Change behavior = wait for vendor.
2. **Hard to control behavior network-wide.** Standard protocols don't give fine-grained control.
3. **Middleboxes everywhere.** Firewalls, load balancers, NAT each are their own special device.

> Segata's framing: monolithic router with proprietary HW + proprietary OS.
> Closed stack, slow innovation.

---

## The SDN idea (in one sentence)

> **Rip the control plane out of every router. Put it in one centralized
> "controller" instead. Let the controller program every switch's data plane
> directly.**

```
       +---------------------+
       |   SDN Controller    |  <- THE central brain
       |   (software on a    |
       |    commodity server)|
       +--+-----+-----+------+
          |     |     |       (OpenFlow protocol)
          v     v     v
       +----+ +----+ +----+
       |SW 1| |SW 2| |SW 3|  <- dumb fast forwarders
       +----+ +----+ +----+
```

Three consequences:
1. Controller has a **god's-eye view** -- sees whole topology, every link, every flow.
2. Switches become **commodity hardware**. Smart logic lives in software elsewhere. Cheap.
3. You can **program network behavior directly** via the controller's API.

---

## The mainframe -> PC analogy

| | Traditional networking | SDN |
|---|---|---|
| Hardware | Proprietary, vertical | Commodity, horizontal |
| Software | Vendor-locked | Open, swappable |
| Innovation | Slow | Fast |
| Industry | Few players | Big ecosystem |

| | Mainframes (1970s) | PCs (1990s+) |
|---|---|---|
| Hardware | IBM proprietary | Intel/AMD commodity |
| Software | IBM only | Windows/Linux/Mac |
| Innovation | Slow | Fast (anyone writes apps) |

**SDN is to networking what the PC revolution was to computing.**
Closed -> open. Slow -> fast. Few players -> many.

---

## Why now (why ~2010)

- Cheap x86 servers became powerful enough to run network control at scale.
- Commodity switching chips (merchant silicon) became available.
- Standardized protocol -- **OpenFlow** -- emerged so controllers/switches can talk across vendors.

First big real-world success: **Google's B4** (2013), running global backbone
on SDN, reportedly hitting 70-95% link utilization vs the industry norm of 30-40%.

---

## Why centralization wins: 3 things traditional routing CAN'T do

Setup: small network. Operator wants control over how u-to-z traffic flows.

### Problem 1: Force a specific (non-shortest) path
"Route u->z via uvwz instead of uxyz."
- Only knob in traditional routing is **link weights**.
- Weights are global -- fudging them affects unrelated flows too.
- You wanted per-flow control. You're stuck.

> Segata: *"link weights are only control 'knobs': not much control!"*

### Problem 2: Load-balance one flow across multiple paths
"Send 50% of u->z via uvwz and 50% via uxyz."
- Distributed routing algorithms pick ONE shortest path.
- ECMP works only when paths are *exactly* equal weight; can only split by hash, not policy.
- 30/70 split? Not possible.

### Problem 3: Route different kinds of traffic differently
"Send 'blue' traffic on cheap path, 'red' traffic on encrypted path -- same destination z."
- Traditional IP routing is **destination-based** -- routers only look at dst IP.
- Two packets to same destination get same treatment. Period.
- Can't differentiate flows by their type / source / classification.

> Real networks have very different traffic types (latency-sensitive, throughput-hungry,
> confidential, internal vs guest). One-size-fits-all routing doesn't serve them.

---

## How SDN solves all three

SDN switches do **"generalized flow-based forwarding"** -- a match/action model.
The controller installs rules like:

```
if (src=10.1.1.5  AND  dst=10.2.2.10  AND  TCP port=8080)
   -> send out port 3

if (VLAN=10  AND  packet color=blue)
   -> send out port 5
```

Match criteria can be ANY header field (src/dst IP, MAC, VLAN, TCP/UDP ports, ...).

| Problem | SDN solution |
|---|---|
| Force u->z via uvwz | Controller installs rule "if src=u and dst=z, output toward v" at u. Other flows unaffected. |
| Load-balance 50/50 | Two rules with different match criteria (odd vs even src ports) -> different paths. |
| Differentiate blue vs red | Rules match on traffic type (src subnet, DSCP) and steer differently. |

> Slide quote: **"Generalized forwarding and SDN can be used to achieve any routing desired."**

---

## The deeper mental shift: persuasion -> command

- **Traditional:** network behavior is the *emergent result* of distributed protocols. You shape it indirectly via knobs (weights, timers).
- **SDN:** network behavior is *directly programmed*. Controller installs the exact rule you want.

It's the difference between *persuading* a system and *commanding* it.
You finally have a **programmable network** the way you've always had programmable computers.

---

## Connection to your projects

Two project options live directly in this lesson:
- **Network slicing in SDN** -- your controller does the traffic engineering.
- **SDN-based firewall** -- your controller installs match/action rules dynamically.

---

## SDN Architecture: 3 layers + 2 APIs

```
+----------------------------------+
|  Apps (routing, firewall, LB)    |  Layer 3 -- "the apps"
+--------------+-------------------+
               | Northbound API (REST, intent)
+--------------+-------------------+
|  SDN Controller (Network OS)     |  Layer 2 -- "the brain"
+--------------+-------------------+
               | Southbound API (OpenFlow, NETCONF, OVSDB)
+--------------+-------------------+
|  SDN-Controlled Switches         |  Layer 1 -- "the muscle"
|  (flow tables, fast forwarding)  |
+----------------------------------+
```

### Layer 1: Data plane (switches)
- Dumb fast forwarders with flow tables (`if match X -> action Y`).
- Commodity hardware. Intelligence is in the controller above.
- Buy from any vendor that supports OpenFlow.

### Layer 2: Controller (network OS)
"Network OS" = same role as Windows/Linux but for the network.

Three sub-components inside:
1. **Communication** -- speaks southbound protocols to switches.
2. **State management** -- the network database: topology, links, hosts, flow tables, stats.
3. **App abstractions** -- exposes a clean API (REST/intent) hiding the messy details.

Real controllers: **OpenDaylight (ODL)**, **ONOS**, **Ryu** (used in your lab), **Floodlight**.

### Layer 3: Apps (the actual smarts)
SDN app = a program using the controller's API to make the network do something.
Examples: routing, firewall, load balancer, traffic engineering, monitoring.

**Crucial:** apps can come from anyone -- the controller vendor, third parties, you.
This is the unbundling.

---

### The two APIs

| | Direction | Who talks | Examples |
|---|---|---|---|
| **Northbound** | Apps <-> Controller | Programmers write apps | REST API, intent (ONOS), language bindings |
| **Southbound** | Controller <-> Switches | Controller programs switches | **OpenFlow** (dominant), NETCONF, OVSDB, P4Runtime, SNMP |

Names refer to direction in the architecture diagram. North = up (toward apps). South = down (toward switches).

---

### "Logically" centralized != physically centralized

Production controllers are a **cluster** of instances sharing state.

Reasons:
- **Fault tolerance** -- one dies, others take over.
- **Performance scale** -- different controllers handle different parts of network.
- **Geographic distribution** -- low latency to the switches they manage.

To apps and switches it still looks like one logical brain.

---

### The OS analogy

| | Laptop OS | SDN |
|---|---|---|
| Hardware | CPU, RAM, NIC | Switches, links |
| Kernel | Manages hw, exposes syscalls | Controller, manages switches, exposes northbound API |
| Driver | Talks to specific hw | Southbound plugin (OpenFlow) |
| Apps | Chrome, Word | Routing, firewall apps |
| API | POSIX, Win32 | REST, intent |

SDN consciously tries to give the network the same "platform with apps on top" structure that computers have had for decades.

---

### Slide quote (exam-ready)

> "SDN controller (network OS): maintains network state information, interacts with network
> control applications 'above' via northbound API, interacts with network switches 'below'
> via southbound API, implemented as distributed system for performance, scalability,
> fault-tolerance, robustness."

---

## OpenFlow protocol

The dominant southbound protocol. Runs over TCP (TLS optional). Switch initiates connection at boot. Controller is boss, switch is worker.

### Flow table structure

Every switch has one or more flow tables. Each entry has 3 pieces:

```
+---------------------+----------------+-----------+
| Match fields        | Actions        | Counters  |
+---------------------+----------------+-----------+
| src/dst IP, MAC,    | output port,   | packets,  |
| VLAN, TCP ports, ...| drop, modify,  | bytes     |
|                     | send-to-ctrl   |           |
+---------------------+----------------+-----------+
```

- **Match:** any header field, wildcards allowed.
- **Action:** output port N, drop, modify header, send-to-controller (chainable).
- **Counters:** stats for monitoring/billing.

Packet arrives -> walk table by priority -> first match wins. **No match -> usually send to controller** (triggers a `packet-in`).

### Three message classes

**(1) Controller -> switch**
| Message | Purpose |
|---|---|
| `features` | Query switch capabilities |
| `configure` | Set switch-wide config |
| `modify-state` | **Install/delete/modify flow entries** (the big one) |
| `packet-out` | Inject a packet into the network from controller |

**(2) Async (switch -> controller)**
| Message | Purpose |
|---|---|
| `packet-in` | "Got a packet I can't handle. Help." |
| `flow-removed` | A flow entry expired/was deleted |
| `port-status` | A port went up/down (failure notification) |

**(3) Symmetric**
| Message | Purpose |
|---|---|
| `hello` | Initial handshake |
| `echo` | Keepalive |

> As an app developer you call the controller's northbound API (`install_flow(...)`),
> not OpenFlow messages directly. Controller translates under the hood.

---

## The big walkthrough: link failure -> automatic reroute

Tying it all together. Network of S1-S4. Dijkstra app registered for link-change callbacks.

### Step 1 — S1 detects the failure
S1's port goes down (cable cut, neighbor dead). S1 sends `port-status` to controller.

### Step 2 — Controller updates state
State management component updates link database: link S1<->S2 = DOWN.

### Step 3 — Subscribed apps notified
Dijkstra app's callback fires (it had said "call me on link changes").

### Step 4 — Dijkstra computes new routes
Reads updated topology, runs shortest path, produces new paths.
- Old: A -> S1 -> S2 -> S4 -> B
- New: A -> S1 -> S3 -> S4 -> B

### Step 5 — App tells controller to install new rules
Via northbound API: `controller.update_flows({...})`.

### Step 6 — Controller pushes via OpenFlow
Sends `modify-state` to each affected switch. Switches update tables.
Next packet for B takes the new path.

### Why this is dramatically better than traditional

Traditional (OSPF):
1. S1 announces failure to all neighbors
2. Announcement propagates
3. Every router recomputes
4. Every router installs new routes
5. Eventually they converge

Time: **seconds to tens of seconds**. Network in inconsistent state during convergence.

SDN:
- One controller, recomputes once, pushes to all switches.
- Time: **milliseconds**. No inconsistent intermediate states.

---

## Reactive vs proactive flow installation

**Reactive**
- Unknown packet -> `packet-in` to controller -> rule installed.
- Pros: small tables (only active flows have rules).
- Cons: first packet slow; "packet-in storm" if too many new flows.

**Proactive**
- Controller pre-installs rules at boot based on policy.
- Pros: consistent low latency, no controller bottleneck.
- Cons: flow tables can grow huge.

Real systems mix both.

> **Project relevance:** SDN firewall = reactive port-scan detection + proactive static rules.

---

## Real-world controllers

### OpenDaylight (ODL)
- Backed by Linux Foundation, big-vendor consortium.
- Heavily modular, plugin-based.
- **Service Abstraction Layer (SAL)** -- glue between built-in services and southbound plugins.
- **Multi-protocol southbound:** OpenFlow, NETCONF, OVSDB, SNMP -> can manage legacy + modern gear together.
- Targets: datacenter, enterprise.

### ONOS (Open Network Operating System)
- Born at Stanford, now Linux Foundation.
- Carrier-grade focus (telcos, ISPs).
- **Intent framework** -- declarative API: tell controller "connect A to B with 1 Gbps", it figures out rules.
- Heavy distributed-core emphasis for reliability.

### Comparison

| | OpenDaylight | ONOS |
|---|---|---|
| Origin | Linux Foundation consortium | Stanford -> LF |
| Target | DC, enterprise | Carrier-grade |
| Distinctive | Multi-protocol SAL | Intent framework |
| Architecture | Modular plugins | Distributed core |

> **Lab uses Ryu** -- simpler Python-based controller. Great for learning since
> controller logic is just a Python program.

---

## SDN Challenges (likely exam material)

### 1. Hardening the control plane
Controller = brain of network. Must be reliable, performant, secure, distributed.
Distributed systems problem applied to networking.

### 2. Scalability
Reactive flow installation can overwhelm controller. Mitigations: proactive rules,
distributed controller clusters, hierarchical controllers.

### 3. Security
- Controller compromise = whole network owned.
- Spoofed controllers (attacker pretends to be controller).
- Southbound DoS attacks.

Mitigations: TLS with mutual auth, intrusion detection, defense in depth.
ONF has security working group.

### 4. Mission-specific requirements
Real-time, ultra-reliable, ultra-secure -- no one-size-fits-all SDN.
Must design controller + apps for the specific requirement.

### 5. Internet-scale operation
Within one domain: fine. Across the whole public internet (thousands of independent
orgs): unsolved. BGP works precisely because no single party controls.
SDN's central brain doesn't fit that political reality.

### Bonus: SDN in 5G
5G separates user plane and control plane (CUPS) -- same idea applied to cellular core.
One of the biggest practical deployments of SDN today.

---

## Network management (SNMP, NETCONF, YANG)

Different from SDN's control plane, but related: how do you actually *run* the network
day-to-day? Monitor, configure, alert.

### CLI
Engineer SSH-es into devices, types commands. Old-school, scriptable, painful.

### SNMP (Simple Network Management Protocol)
- Legacy standard for monitoring.
- Two modes: **request/response** (server polls) and **trap** (device pushes alert).
- **MIB** = Management Information Base = tree of named values on the device.
- **OID** = Object Identifier, the dotted-number address of a MIB value
  (e.g. `1.3.6.1.2.1.7.1` = UDP input datagrams).
- Message types: GetRequest, GetNextRequest, SetRequest, Response, Trap.
- Today: still widely used for monitoring (CPU, counters, alerts). Rarely for serious config.

### NETCONF
Modern replacement for SNMP for configuration.

| | SNMP | NETCONF |
|---|---|---|
| Encoding | Binary | **XML** |
| Transport | UDP | TLS/SSH |
| Operations | Get/Set | Rich (get-config, edit-config, lock, commit) |
| Atomic multi-device commits | No | **Yes** |
| Strongly typed | No | **Yes (via YANG)** |
| Use | Monitoring | Config + monitoring |

Uses RPC (Remote Procedure Call) model. Messages are XML over secure transport.

### YANG
Data-modeling language used by NETCONF to describe what fields exist on a device, with types and constraints. Lets both sides validate config before pushing.

Example: "an interface has a string name, an optional boolean enabled, and a uint16 mtu."

### One-line summaries (exam-ready)

- **SNMP** = old, simple, mostly monitoring; tree of OIDs called the MIB.
- **NETCONF** = XML over secure transport, RPC, supports atomic config changes.
- **YANG** = data-modeling language used to describe NETCONF data.

### Relationship to SDN

Two views:
1. **Alternatives** -- network management = device-by-device; SDN = network-as-a-whole.
2. **Complementary** -- SDN controllers can USE NETCONF as a southbound protocol (ODL does).

Most modern networks use a mix.

---

# PDF 4 — Final Recap

| Concept | One-liner |
|---|---|
| Data vs control plane | Muscle vs brain. Forwarding vs deciding. |
| Per-router control | Every router runs both. Closed, slow. |
| SDN idea | Centralize control, switches become commodity. |
| 3 things SDN does that traditional can't | Force a path; load-balance; differentiate flows. |
| 3-layer architecture | Apps -> Controller -> Switches |
| 2 APIs | Northbound (apps<->ctrl) / Southbound (ctrl<->switches) |
| OpenFlow | Dominant southbound protocol. Match/action flow tables. |
| Failure walkthrough | port-status -> controller updates -> app fires -> rules pushed. ms convergence. |
| Reactive vs proactive | Install on first packet vs pre-install. Real systems mix. |
| Real controllers | OpenDaylight (multi-protocol), ONOS (intent), Ryu (lab). |
| 5 challenges | Hardening, scaling, security, mission-specific, internet-scale. |
| Network management | SNMP (old monitoring), NETCONF (modern XML/RPC), YANG (data model). |

**Big theme:** SDN = PC revolution applied to networking. Hardware -> commodity, software -> innovation, network -> platform with apps on top.

This same pattern will reappear in:
- **SDR** (next big PDF) -- same idea applied to radios.
- **NFV** (virtualization PDF) -- same idea applied to network functions.

Internalize this and the rest of the course is repeated applications of the same trick.
