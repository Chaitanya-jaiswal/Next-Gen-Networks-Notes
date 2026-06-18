# PDF 7 — SDR (Software Defined Radio)

> The professor's research home turf. Goes deep.
> SDR is "SDN for the wireless physical layer" — same softwarization pattern, different layer.

## The fundamental idea

| Layer | Traditional | Softwarized |
|---|---|---|
| Computer hardware | Bare-metal app | VM / Container |
| Network control | Per-router proprietary OS | SDN controller |
| Radio waveform | Custom RF chip per protocol | **SDR** |

> Take something previously locked in dedicated hardware.
> Replace it with software on a commodity processor + minimal generic hardware front-end.
> Now you can change behavior with code instead of building new silicon.

---

## The motivation, in two laws

### Moore's law (1965)
"Transistors per IC double approximately every 2 years."

Implication: **the cost of running a fixed set of signal processing tasks halves every 2 years.**
Eventually general processors become fast enough to do in software what required hardware.

### Mitola's vision (1993)
Joseph Mitola III: "If software is flexible and easy to update, why do we keep building radios in hardware?"

His ideal radio (1993 paper):
```
antenna --- [A/D + D/A] --- [SOFTWARE]
              conversion    does everything:
                            - modulation
                            - demodulation
                            - error correction
                            - signal generation
```

One radio, infinite functions. Phone = WiFi card today, LTE modem tomorrow, FM radio next week. Just load different software.

### Is Mitola's dream fully realized today?

Not quite. Sampling at multi-GHz rates is still impractical at consumer prices.
A 2 GHz LTE signal would need ~40 Gbps data rate -- melts any bus.

Real SDR is a **compromise**: minimal analog front-end (RF stage), everything below RF in software.

---

## SDR definition

> A class of reconfigurable/reprogrammable radios whose physical-layer characteristics
> can be significantly modified **via software**.

### Five distinctive features

| Feature | Means |
|---|---|
| **Multi-functionality** | Same HW becomes WiFi / LTE / GPS / walkie-talkie -- just load different SW |
| **Global mobility** | Talk to any network on Earth without new hardware |
| **Compactness & power efficiency** | Should be small and battery-friendly (not always achieved) |
| **Ease of manufacturing** | New radio = new software, not new chip |
| **Ease of upgrading** | Firmware update for new protocols. No hardware swap. |

> Goal: end user doesn't even know SDR is underneath. Just sees their device gain new bands/features after an update.

---

## Anatomy of an SDR system

```
Analog                                       Software
<--------------- HW ---------------> <-------------------------- SW -------------------->

[Antenna]--[RF Chain]--[A/D & D/A]--[Symbol mod/demod]--[Channel dec / Source dec]--[App]
            (filter,    (sampling)
             amp,
             mix to
             baseband)

HARDWARE                          SOFTWARE on a PC / general processor
("tunable")                       ("programmable")
```

Receive path (left to right):
1. **Antenna** picks up waves
2. **RF chain** = analog: amplify, filter noise, shift GHz down to digital-friendly rate
3. **A/D** = sample analog into digital
4. Everything after this is software (demod, error correct, decompress, deliver)

Transmit path: run it backwards.

> Only the high-frequency analog stuff is physically required.
> Everything that doesn't strictly need to be analog is now software = reprogrammable.

---

## Before vs after SDR (the practical revolution)

**Before SDR (building a new radio):**
- Hire RF engineering team
- Design custom chips
- Foundry, 6-month wait
- Tens of millions $, years of time
- A handful of companies on Earth could play

**After SDR:**
- Buy USRP/BladeRF/HackRF
- Plug into laptop
- Write Python/GNU Radio code
- Hundreds to low thousands of $
- Weeks of time
- Anyone, including students

> This is why one project option here is "implement OTFS modulation in GNU Radio."
> Used to be a multimillion-dollar industry project. Now it's a student project.

---

## How a radio actually works (intuition, no math)

### Why high frequencies?

Antennas work best when their length is a fraction of the wavelength of the signal.

| Signal | Freq | Wavelength | Half-wavelength antenna |
|---|---|---|---|
| Voice | 1 kHz | 300 km | 150 km !! |
| AM radio | 1 MHz | 300 m | 150 m |
| FM radio | 100 MHz | 3 m | 1.5 m |
| WiFi | 2.4 GHz | 12.5 cm | 6 cm |
| 5G mmWave | 30 GHz | 1 cm | 5 mm |

To send voice with a tiny antenna -> shift voice signal UP to GHz. Receiver shifts it back down.

### Two vocabulary words

- **Baseband** = your raw data signal, near 0 Hz.
- **Passband** = same signal shifted up to high frequencies, what flies through the air.

```
[baseband] -- upconvert --> [passband] -- transmitted -->
                                                          v
                                                      received
                                                          |
[baseband] <-- downconvert -- [passband] <----------------+
(recovered)
```

### The mixer = multiplication = frequency shift

The fundamental trick: **multiplying two signals shifts their frequencies.**

A mixer multiplies the input by a high-frequency **carrier** (pure sine wave at, say, 2 GHz):

```
input (baseband)
"hello world"  ---->-- [X] ----> output (passband, around 2 GHz)
                       |
carrier (2 GHz) ------+
pure sine wave
```

Same signal energy, just relocated on the frequency axis.

### Downconversion = multiply + filter

Multiply the received signal by the same carrier frequency. Math: two sine waves multiplied give you:
- their **sum** (high junk, e.g. 2 GHz)
- their **difference** (back to baseband, 0 Hz)

Low-pass filter (LPF) throws away the junk:

```
input (passband ~1 GHz)
   ---->-- [X] ---- (0 Hz + 2 GHz junk) ---> [LPF] ---> baseband recovered
            |
carrier (1 GHz)
```

Receive path = **multiply, then filter**. Original baseband recovered.

### Full RF chain

```
TX side:                              RX side:

baseband data                                       recovered baseband
     |                                                       ^
     v                                                       |
   [MIXER] -- passband --> [ANTENNA] -)))- ((( -- [ANT] [MIXER + LPF]
     |                                                       ^
carrier @ f0                                          carrier @ f0
(high freq)                                           (high freq)
```

Everything between the two mixers = analog RF world.
Everything outside the mixers = software territory.

### Why Mitola's dream still isn't fully realized

To sample directly at the antenna requires multi-GHz A/D rates. At 8 bits per sample, ~32 Gbps data flowing into the computer. No commodity bus handles that.

So real SDRs:
- Do RF up/down conversion in **analog hardware** (cheap, mature)
- Sample only after reaching baseband
- Everything from baseband onward = **software** (modulation choice, error correction, framing, protocol)

This is the **practical compromise** -- still revolutionary, just not "all software."

---

## SDR hardware

An SDR system = peripheral (RF analog work) + computer (software). They talk over USB or Ethernet.

### Specs that matter (what they mean)

| Spec | Controls | Why you care |
|---|---|---|
| **Frequency range** | Which radio bands you can tune to | Need GPS / FM / mobile? Check it covers them |
| **Sample rate (MS/s)** | Bandwidth you can capture/transmit | 20 MHz WiFi needs ~20 MS/s |
| **A/D bit depth** | Fineness of samples | Higher = better SNR = hear weaker signals |
| **Host interface** | USB 2 / USB 3 / Gig Eth | The data-rate bottleneck |
| **TX power (EIRP)** | Transmit strength | Indoor demo: 10 dBm fine. Outdoor: need more |
| **MIMO** | Multi-antenna support | Needed for 4G/5G |

---

### USRP family (Ettus Research, the dominant player)

Created by **Matt Ettus**. Design philosophy: minimal peripheral, host computer does heavy lifting.

#### USRP1 (the original)
- Sample rate: 8 MS/s | A/D: 12-bit | USB 2.0 | TX: 17 dBm
- Now historic. Still in old papers.

#### USRP N210 (the academic workhorse)
- Sample rate: 25-50 MS/s | A/D: 14-bit | **Gigabit Ethernet** | TX: 15 dBm | MIMO
- The Ethernet upgrade is the big jump -- enough for ~25 MHz bandwidth.
- **What this course's labs use.**

#### USRP X310 (high-end)
- Sample rate: 200 MS/s full-duplex | A/D: 14-bit | 10G Eth / PCIe | Xilinx Kintex FPGA
- Optional: GPS-disciplined OCXO for nanosecond timing.
- For 5G research, advanced MIMO. Expensive (5-figures).

Why USRPs dominate: open hardware, excellent GNU Radio support, MIMO sync, ecosystem of daughterboards.

---

### Cheaper alternatives

| | BladeRF | LimeSDR | HackRF | Matchstiq |
|---|---|---|---|---|
| Sample rate | 61.44 MS/s | 60 MS/s | 20 MS/s | 28 MHz BW |
| Interface | USB 3.0 | USB 3.0 | USB 2.0 | (self-contained) |
| RF range | 300 MHz – 3.8 GHz | 100 kHz – 3.8 GHz | 1 MHz – 6 GHz | 300 MHz – 3.8 GHz |
| MIMO | Yes | -- | Yes (2 units) | -- |
| Notes | Solid mid-range | Wide range incl. HF | Hacker favorite, half-duplex | Has own CPU + GPS, mobile/vehicular |

---

### Quick-pick guide

| | USRP N210 | USRP X310 | BladeRF | LimeSDR | HackRF | Matchstiq |
|---|---|---|---|---|---|---|
| Best for | Course labs, research | High-end research, 5G | Mid-range projects | Wide-range hobbyist | Learning, hacking | Mobile/vehicular |
| Bandwidth | ~25 MHz | ~200 MHz | ~60 MHz | ~60 MHz | ~20 MHz | ~28 MHz |
| Lowest freq | 50 MHz | 10 MHz | 300 MHz | 100 kHz | 1 MHz | 300 MHz |

**USRP N210 = academic sweet spot. What the course uses.**

---

### Practical "not on spec sheet" stuff

- **Daughterboards** -- USRPs ship without antennas/RF tuners. Buy the right daughterboard for your band. Same USRP body works for all.
- **Antennas matter** -- bad antenna = useless SDR.
- **Clock stability** -- needed for LTE / GPS. X310 + GPS OCXO is gold standard.
- **Latency vs throughput** -- Ethernet gives high bandwidth but few microsec extra latency.

---

### Project hardware fit

| Project | Hardware |
|---|---|
| 802.11p comparison | CUBE EVK + USRP (lab provides) |
| WiFi minichannels | USRP + GNU Radio |
| Cognitive walkie-talkie | USRP + GNU Radio |
| OTFS modulation | Mostly software + simulation |

You don't buy hardware -- the lab provides USRPs.

---

## SDR software stack

### GNU Radio (the "Linux of SDR")

> Free, open-source toolkit for building/deploying SDR systems.

- Started 2001. Backed by academics + hobbyists + industry.
- **C++** core, **Python** user API.
- Runs natively on Linux (Ubuntu). macOS okay. Windows painful.
- **Standard** way to talk to USRPs.

### The flowgraph concept (KEY)

Chain signal processing blocks together, data flows through:

```
[Source] -> [Filter] -> [Demodulator] -> [Decoder] -> [Sink]
```

Hundreds of built-in blocks. Connect with arrows. Like Lego.

Two ways to build a flowgraph:
1. **Write Python code** -- instantiate blocks, call `connect()`.
2. **GNU Radio Companion (GRC)** -- drag-and-drop GUI.

Both produce the same thing -- GRC actually generates Python.

### GNU Radio Companion (GRC)

Visual front-end. Similar to Simulink or LabVIEW.

Pros:
- Visual -- see whole signal chain at a glance.
- Live tuning -- sliders adjust parameters while running.
- Built-in scopes and spectrum displays.
- No coding to get started.

When to graduate to raw Python:
- Better Git diffs (text vs XML).
- Need custom blocks (no built-in matches what you want).
- Integration with web/DB/ML libs.

**Course labs start in GRC.** Projects may end up in Python/C++.

---

### Pre-built protocol stacks

| Project | Implements | Notes |
|---|---|---|
| **OpenBTS** | GSM (2G) + some 4G | Cheap remote/disaster cell networks. C++ + USRP. |
| **OpenAirInterface (OAI)** | 4G + 5G (RAN + Core) | The big one for modern cellular research. Universities use heavily. |
| **srsRAN** (formerly srsLTE) | LTE/LTE-A full stack | Production-quality. Some real operators use it. Needs 30.72 MS/s. |
| **OpenWebRX** | Browser-based SDR receiver | Listen to someone else's SDR over the web. Hobbyist. |

### FutureSDR (the new modern alternative)

Written in **Rust**. By Bastian Bloessl.
- Better GPU/FPGA accelerator support
- Asynchronous programming
- Runs on Linux, macOS, Windows, Android, **WASM**
- Tagline: "SDR go brrr!"

If your project needs raw performance, consider FutureSDR over GNU Radio.

---

### Closed-source alternatives

| Tool | Vendor | Use case |
|---|---|---|
| MATLAB + Simulink | MathWorks | Research prototyping, education (most students already know MATLAB) |
| LabVIEW | National Instruments | Instrument control, real-time. NI owns Ettus -> "official" USRP path. |

Most academic SDR research uses GNU Radio anyway (free + open).

---

### The stack picture

```
+-----------------------------------------------+
| Pre-built protocol stacks                     |
| OpenBTS (GSM) | OAI (4G/5G) | srsRAN (LTE)    |
+-----------------------------------------------+
| General-purpose SDR frameworks                |
| GNU Radio (+ GRC) | FutureSDR (Rust)          |
+-----------------------------------------------+
| Hardware drivers (UHD for USRP)               |
+-----------------------------------------------+
| Hardware: USRP, BladeRF, HackRF               |
+-----------------------------------------------+
```

---

### What you'll actually do in this course

| Project | Software |
|---|---|
| 802.11p comparison | GNU Radio's open-source 802.11p implementation |
| WiFi minichannels | Modify GNU Radio 802.11 |
| Cognitive walkie-talkie | GNU Radio flowgraph + spectrum sensing |
| OTFS modulation | Write new GNU Radio blocks (C++/Python) |

In all cases: **GNU Radio + GRC**.

---

## Three pillars of software-defined communications

```
       FUTURE NETWORKS (roof)
       /                \
    SDN                Wireless Virtualization
    (pillar 1)         (pillar 2)
       \                /
        SDR (foundation)
```

- **SDR** turns physical layer into software.
- **SDN** decouples control plane from data plane (from PDF 4).
- **Wireless virtualization** turns spectrum/infrastructure into shareable slices.

Together: every layer above the antenna is malleable software.
Foundation of 5G, O-RAN, future 6G.

---

## Wireless Network Virtualization

Server virtualization (PDF 6) but for radio infrastructure.

### Why?

A telco wants to rent slices of their towers + spectrum + core to:
- **MVNOs** (virtual operators with no infrastructure)
- **Specific apps** (low-latency car slice, massive-scale IoT slice)
- **Different internal services** (voice / streaming / gaming)

Same physical network, multiple virtual networks. Like VLAN, but for cellular.

### The four players

| Acronym | Stands for | What they own |
|---|---|---|
| **InP** | Infrastructure Provider | Physical hardware (towers, fiber) |
| **MNO** | Mobile Network Operator | Spectrum license + sometimes infrastructure |
| **MVNO** | Mobile Virtual Network Operator | Customers, brand — **no spectrum, no towers**, rents everything |
| **SP** | Service Provider | A specific app on top |

### The four levels of slicing

| Level | What gets sliced | Notes |
|---|---|---|
| 1. **Spectrum-level** | Frequency bands | Simple concept, political reality is hard |
| 2. **Infrastructure-level** | Physical hardware (multiple virtual base stations on one tower) | Type-1 hypervisor for towers |
| 3. **Network-level** | Everything — spectrum + RAN + Core (end-to-end virtual network) | The ideal. Real **5G network slicing**. |
| 4. **Flow-level** | Individual data flows within an existing network | SDN traffic engineering, applied to wireless |

### Wireless virtualization controller

Like SDN controller. Two parts:
- **Substrate controller** — used by InP/MNO to manage physical network.
- **Virtual controller** — used by MVNO/SP to manage their slice.

Underlay (physical) vs overlay (sliced virtual) — same pattern as VXLAN.

---

## Five challenges (same as SDN)

| Challenge | Why it's hard |
|---|---|
| Performance vs flexibility | GPP slow, custom chips rigid. FPGAs usually the sweet spot for SDR. |
| Scalability | One controller for millions of users -> distribute it |
| Security | Central controller = juicy target. TLS, mutual auth, IDS. |
| Mission-specific requirements | Real-time, ultra-reliable, ultra-secure -- one size doesn't fit all |
| Internet-scale | Cross-operator slicing needs economic + political cooperation |

---

## Real application examples

### 1. SALICE — emergency communications recovery
SDR-based system for rescue teams after disasters.
- Detects what radio standards are available (GSM, WiFi, WiMAX) and adapts on the fly.
- Provides assisted localization when satellites are blocked.
- Multi-standard SDR at its best.

### 2. LTE-A cooperative relaying
Use spare base stations as relays to extend coverage in dead zones.
- Emulated in lab with GNU Radio + USRPs.
- Measured real coverage gains cheaply, no real cellular gear needed.

### 3. SDR satellite gateway for IoT
Bridge wireless sensor networks (Zigbee/WiFi/LoRa) to satellite (DVB-S2x).
- Multi-protocol gateway deployable anywhere.
- Perfect SDR use case: flexible + adaptable + cheap commodity hardware.

**Pattern:** SDR enables flexible, multi-protocol, software-controlled radios that adapt to scenarios.

---

# PDF 7 Final Recap

| Concept | Bottom line |
|---|---|
| SDR | Replace dedicated RF hardware with software + minimal RF front-end |
| Moore's law | Made it possible (cheap general processors) |
| Mitola's vision (1993) | "Put the whole radio in software." Still aspirational. |
| Baseband vs passband | Data at low freq; transmission needs high freq |
| Mixer = multiplication | Multiplying by a carrier shifts signal in frequency |
| Upconversion | Multiply -> shift up to RF for TX |
| Downconversion | Multiply + LPF -> recover baseband at RX |
| SDR hardware | USRP (course default = N210), BladeRF, LimeSDR, HackRF, Matchstiq |
| SDR software | GNU Radio + GRC; OpenBTS/OAI/srsRAN; FutureSDR (Rust) |
| Three pillars | SDR + SDN + Wireless Virt = software-defined communications |
| Wireless virtualization | 4 slicing levels: spectrum, infrastructure, network, flow |
| 5 challenges | Same as SDN: performance, scaling, security, mission-spec, internet-scale |

**The big theme:** SDR completes the softwarization trilogy. After SDN (control plane) + Containers/VMs (compute) + SDR (radio), **every layer is malleable software**. The "next-generation network" = behavior changes via software update, not hardware swap.
