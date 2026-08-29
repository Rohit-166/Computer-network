# Chapter 1 — Computer Networks and the Internet
### (Kurose & Ross, *Computer Networking: A Top-Down Approach*, 7th Ed.)

---

## 1.1 What Is the Internet?

Two ways to describe the Internet:

### 1.1.1 Nuts-and-Bolts Description
- The Internet interconnects **billions of computing devices** worldwide — called **hosts** or **end systems** (PCs, servers, smartphones, IoT devices, etc.).
  - ~5 billion devices connected in 2015; projected 25 billion by 2020 (Gartner).
  - ~3.2 billion Internet users in 2015 (~40% of world population).
- End systems are connected by **communication links** (coax cable, copper wire, fiber, radio spectrum) and **packet switches**.
  - Links have a **transmission rate** measured in **bits/second**.
- **Packets**: When a sending end system has data to send, it **segments** the data and adds **header bytes**, forming packets sent through the network to be reassembled at the destination.
- **Packet switch**: takes a packet on an incoming link and forwards it on an outgoing link.
  - Two main types: **routers** (used in the network core) and **link-layer switches** (used in access networks).
- **Route/path**: the sequence of links and packet switches a packet traverses from source to destination.
- Cisco predicted annual global IP traffic would pass 1 zettabyte (10²¹ bytes) by end of 2016, reaching 2 zettabytes/year by 2019.
- **Analogy**: packet-switched networks ~ transportation networks. Packets ≈ trucks; communication links ≈ highways/roads; packet switches ≈ intersections; end systems ≈ buildings.
- **ISPs (Internet Service Providers)**: End systems access the Internet through ISPs — residential (cable/telco), corporate, university, WiFi-hotspot, and cellular ISPs.
  - Each ISP is itself a network of packet switches and links.
  - Lower-tier ISPs interconnect via national/international **upper-tier ISPs** (e.g., Level 3, AT&T, Sprint, NTT).
- **Protocols**: End systems, switches, and other Internet pieces run protocols that **control sending and receiving of information**.
  - **TCP** (Transmission Control Protocol) and **IP** (Internet Protocol) are the two most important — collectively **TCP/IP**.
  - IP specifies the packet format sent/received among routers and end systems.
- **Standards**: Internet standards developed by the **IETF** (Internet Engineering Task Force).
  - Standards documents = **RFCs** (Request For Comments) — over 7,000 exist; define protocols like TCP, IP, HTTP, SMTP.
  - **IEEE 802 LAN/MAN Standards Committee** specifies Ethernet and WiFi standards.

### 1.1.2 Services Description
- Alternatively, the Internet is an **infrastructure that provides services to distributed applications** (e-mail, Web, messaging, streaming, social networks, video conferencing, gaming, etc.).
- Applications are **distributed applications** — they run on **end systems**, not inside packet switches in the core.
- **Socket interface**: a set of rules a program must follow so the Internet infrastructure can deliver data from a program on one end system to a program on another end system.
  - **Analogy**: Alice sending a letter to Bob via postal service — must use envelope, address, stamp, mailbox = the "postal service interface." Similarly, apps use the Internet's socket interface.
- Like the postal service (express, confirmation, ordinary), the Internet offers **multiple types of services** to applications.

### 1.1.3 What Is a Protocol?
- **Human analogy**: e.g., asking someone the time —
  1. "Hi" (initiate) → "Hi" (response) → question → answer.
  2. A different/no response indicates unwillingness/inability, so the protocol dictates not to proceed.
- **Definition**:
  > A protocol defines the **format and order of messages** exchanged between two or more communicating entities, as well as the **actions taken** on transmission/receipt of a message or other event.
- **Network protocol** = same idea, but the entities are hardware/software components (computers, routers, smartphones, etc.).
- Example — fetching a Web page:
  1. Computer sends a **connection request** message to server, waits for reply.
  2. Server returns a **connection reply** message.
  3. Computer sends a **GET message** naming the desired page.
  4. Server returns the Web page (file).
- All activity involving two or more remote communicating entities in the Internet is governed by a protocol.

---

## 1.2 The Network Edge

- **End systems** = **hosts** (because they *host* application programs) — sit at the "edge" of the Internet.
  - Divided informally into **clients** (desktops, mobiles, smartphones) and **servers** (more powerful machines storing/distributing Web pages, video, e-mail).
  - Most servers today reside in large **data centers** (e.g., Google has 50–100 data centers, ~15 large ones with 100,000+ servers each).
- **Case History – Internet of Things (IoT)**: wearables, smart thermostats, connected toys, body scales, etc. ~5 billion "things" connected in 2015, projected 25 billion by 2020. Raises major **security and privacy risks** (e.g., hijacked dolls, hacked health-data servers).

### 1.2.1 Access Networks
The **access network** physically connects an end system to the first router ("**edge router**") on the path to any other end system.

**Home Access:**
| Technology | Key Points |
|---|---|
| **DSL** (Digital Subscriber Line) | Uses existing telephone line via telco (telco = ISP). Modem ↔ **DSLAM** at telco Central Office (CO). Line carries 3 bands: downstream data (50 kHz–1 MHz), upstream data (4–50 kHz), voice (0–4 kHz) — simultaneous data + phone call via **frequency-division multiplexing**. **Asymmetric** (down ≠ up rate): e.g., 12 Mbps down / 1.8 Mbps up, or 55 Mbps down / 15 Mbps up. Limited by distance to CO (~5–10 miles). |
| **Cable** | Uses cable-TV infrastructure. Fiber connects head end to neighborhood junctions (500–5,000 homes); coax reaches individual homes → **Hybrid Fiber Coax (HFC)**. Needs a **cable modem**; head end has a **CMTS** (Cable Modem Termination System). **Shared broadcast medium** — every packet from head end travels to every home; upstream also shared, so needs a **multiple access protocol** to avoid collisions. DOCSIS 2.0: up to 42.8 Mbps down / 30.7 Mbps up. |
| **FTTH** (Fiber To The Home) | Optical fiber directly from CO to home. Distribution options: **AON** (Active Optical Network, essentially switched Ethernet) and **PON** (Passive Optical Network — used by Verizon FiOS). PON: home **ONT** → neighborhood **splitter** (<100 homes) → **OLT** at telco CO → Internet. Potential Gbps-range speeds; ~20 Mbps average US downstream (2011) vs 13 Mbps cable, <5 Mbps DSL. |
| **Satellite** | Used where DSL/cable/FTTH unavailable (rural). >1 Mbps. |
| **Dial-up** | Same model as DSL (modem over phone line) but **56 kbps**, very slow. |

**Enterprise/Home LAN Access:**
- **Ethernet**: most prevalent LAN access technology; uses twisted-pair copper to an **Ethernet switch**, connected onward to the Internet. Users: 100 Mbps–1 Gbps; servers: 1–10 Gbps.
- **WiFi (IEEE 802.11)**: wireless users transmit to/from an **access point** (connected via wired Ethernet to the enterprise network). User must be within a few tens of meters. Shared rate >100 Mbps (as of writing).
- A typical **home network** combines broadband access (cable/DSL modem) + wireless router/access point + wired PC + roaming laptop.

**Wide-Area Wireless Access:**
- **3G**: packet-switched wide-area wireless, >1 Mbps.
- **4G / LTE** (Long-Term Evolution): rates >10 Mbps, tens of Mbps reported in deployments. User needs to be within a few tens of **kilometers** of a base station (vs. tens of meters for WiFi).

### 1.2.2 Physical Media
- A **bit** travels via a series of **transmitter-receiver pairs**, each over some physical medium.
- **Guided media**: waves guided along a solid medium (fiber, twisted pair, coax).
- **Unguided media**: waves propagate through atmosphere/space (wireless LAN, satellite).
- Labor cost of installing physical links often exceeds material cost — hence buildings are wired with multiple media types in advance.

| Medium | Notes |
|---|---|
| **Twisted-pair copper wire** | Cheapest, most common. Two insulated wires twisted together (reduces interference). **UTP** (unshielded twisted pair) common for LANs. Data rates: 10 Mbps – 10 Gbps depending on wire gauge/distance. Category 6a cable: 10 Gbps up to ~100 m. Also used for dial-up (56 kbps) and DSL (tens of Mbps). |
| **Coaxial cable** | Two *concentric* copper conductors (vs. parallel in twisted pair). High data rates; common in cable TV/cable Internet. Can be a **shared** medium — multiple end systems connected directly, each receives what others send. |
| **Fiber optics** | Thin flexible medium carrying light pulses (1 pulse = 1 bit). Very high bit rates (tens–hundreds of Gbps), immune to EM interference, low attenuation up to 100 km, hard to tap. Preferred for long-haul/backbone links; costly for short-haul (LAN/home) due to optical device cost. **OC (Optical Carrier)** standard: OC-n = n × 51.8 Mbps (OC-1 to OC-768). |
| **Terrestrial radio** | No physical wire needed; penetrates walls; mobile connectivity. Affected by path loss, shadow fading, multipath fading, interference. Three classes: very short distance (headsets etc.), local area (WiFi), wide area (cellular). |
| **Satellite radio** | **Geostationary satellites**: fixed above one spot, orbit at 36,000 km — introduces ~280 ms propagation delay; speeds up to hundreds of Mbps; used where no DSL/cable. **LEO (Low-Earth Orbiting) satellites**: closer to Earth, don't stay fixed, need multiple satellites for continuous coverage; being developed for future Internet access. |

---

## 1.3 The Network Core

The **network core** = the mesh of packet switches and links interconnecting the Internet's end systems.

### 1.3.1 Packet Switching
- End systems exchange **messages** (control or data — e-mail, JPEG, MP3, etc.).
- Source breaks long messages into **packets**. Packet of L bits sent over a link of rate R bits/sec takes **L/R seconds** to transmit.

**Store-and-Forward Transmission:**
- A packet switch must **receive the entire packet** before transmitting the first bit onward.
- For one packet over one router (ignoring propagation delay): total delay = **2L/R**.
- General case: N links each of rate R (N−1 routers) →
  $$d_{end\text{-}to\text{-}end} = \frac{NL}{R} \quad (1.1)$$

**Queuing Delays & Packet Loss:**
- Each packet switch has an **output buffer/queue** per attached link.
- If a link is busy, arriving packets wait in the output buffer → **queuing delay** (variable, depends on congestion).
- If the buffer is full → **packet loss** (arriving or already-queued packet dropped).

**Forwarding Tables and Routing Protocols:**
- Every end system has an **IP address** (hierarchical, like postal addresses).
- Each router has a **forwarding table** mapping (portions of) destination addresses → outbound link.
- **Analogy**: Joe driving cross-country asking gas station attendants for directions (each attendant only knows the "next hop"), like routers directing packets hop-by-hop.
- **Routing protocols** automatically configure forwarding tables (e.g., by computing shortest paths) — studied in depth in Chapter 5.
- **Traceroute** program (via www.traceroute.org) reveals the actual end-to-end route packets take.

### 1.3.2 Circuit Switching
Two fundamental approaches to moving data: **circuit switching** and **packet switching**.

- **Circuit switching**: resources (buffers, link transmission rate) needed along a path are **reserved for the duration** of the communication session. Classic example: **traditional telephone networks**.
  - A **circuit** is established before data transfer; switches maintain **connection state**.
  - **Analogy**: restaurant with reservations (must call ahead, but seated immediately) vs. one without (no advance hassle, but may wait).
- **Packet switching**: resources are **not reserved**; a session's messages use resources **on demand**, so may have to queue.

**Multiplexing in Circuit-Switched Networks:**
- **FDM (Frequency-Division Multiplexing)**: link's frequency spectrum divided among connections; each gets a dedicated frequency band (e.g., 4 kHz per telephone connection) for the connection's duration.
- **TDM (Time-Division Multiplexing)**: time divided into **frames**, each frame into fixed **time slots**; each connection gets one dedicated slot per frame.
  - Circuit transmission rate = frame rate × bits per slot (e.g., 8,000 frames/sec × 8 bits/slot = 64 kbps).

**Numerical example (from text):** Sending a 640,000-bit file, TDM with 24 slots, link rate 1.536 Mbps, circuit-setup time 500 ms.
- Per-circuit rate = 1.536 Mbps / 24 = 64 kbps.
- Transmission time = 640,000 / 64,000 = 10 sec.
- Total = 10 + 0.5 = **10.5 seconds** (independent of number of links!).

**Packet Switching vs. Circuit Switching:**
- Critics of packet switching: unsuitable for real-time services due to variable/unpredictable delay (queuing delay).
- Proponents: (1) better sharing of transmission capacity, (2) simpler/cheaper/more efficient to implement.
- **Example 1**: 1 Mbps link, users active 10% of time at 100 kbps.
  - Circuit switching (TDM): only **10** simultaneous users supported.
  - Packet switching: with **35** users, P(more than 10 active simultaneously) ≈ 0.0004 — so packet switching supports **3.5× more users** with essentially the same performance.
- **Example 2**: 10 users, one user suddenly sends 1000 packets of 1000 bits while others are idle.
  - Circuit switching (TDM, 10 slots): user stuck using only its 1 slot/frame → takes **10 seconds**.
  - Packet switching: user can use the **full 1 Mbps** link (no contention) → takes **1 second**.
- **Conclusion**: Circuit switching **pre-allocates** capacity regardless of demand (wasteful when idle); packet switching allocates **on demand**, sharing among only those with data to send.
- Trend is toward packet switching, even in traditional telephone networks (e.g., for expensive overseas call segments).

### 1.3.3 A Network of Networks
Building up the real Internet structure incrementally:

1. **Structure 1**: All access ISPs connect to one single global transit ISP — too costly/unrealistic (a mesh of every access ISP to every other would need too many links).
2. **Structure 2**: Multiple competing **global transit ISPs**; access ISPs are **customers**, global ISPs are **providers**. Global ISPs themselves must interconnect.
3. **Structure 3**: Since no ISP has global presence everywhere, add **regional ISPs**; **Tier-1 ISPs** (~a dozen, e.g., Level 3, AT&T, Sprint, NTT — no group officially sanctions Tier-1 status) sit at the top. Multi-tier hierarchy (e.g., China: city access ISPs → provincial ISPs → national ISPs → Tier-1).
4. **Structure 4**: Add **PoPs** (Points of Presence — groups of routers where customer ISPs connect), **multi-homing** (an ISP connects to 2+ providers for redundancy), **peering** (two ISPs at the same level directly interconnect, typically **settlement-free** — neither pays the other), and **IXPs** (Internet Exchange Points — third-party-run meeting points for multiple ISPs to peer; 400+ IXPs exist).
5. **Structure 5 (today's Internet)**: Adds **content-provider networks** (e.g., Google's private global TCP/IP network connecting its 50–100 data centers). These networks **bypass upper-tier ISPs** by peering directly (settlement-free) with lower-tier ISPs/IXPs where possible, but still connect to (and pay) Tier-1 ISPs where necessary.

> **Summary**: Today's Internet = a dozen-ish Tier-1 ISPs + hundreds of thousands of lower-tier ISPs + PoPs + multi-homing + peering + IXPs + content-provider networks. Lower tiers are customers of higher tiers; higher tiers interconnect with each other.

---

## 1.4 Delay, Loss, and Throughput in Packet-Switched Networks

### 1.4.1 Overview of Delay
A packet suffers delay at **each node** (host/router) along its path. Four components of **nodal delay**:

$$d_{nodal} = d_{proc} + d_{queue} + d_{trans} + d_{prop}$$

| Delay | Description | Typical Magnitude |
|---|---|---|
| **Processing delay** (d_proc) | Time to examine packet header, decide outbound link, check bit errors | microseconds or less |
| **Queuing delay** (d_queue) | Time waiting in the output buffer before transmission | microseconds to milliseconds (varies with congestion) |
| **Transmission delay** (d_trans) | Time to push all L bits of the packet onto the link = **L/R** | microseconds to milliseconds |
| **Propagation delay** (d_prop) | Time for a bit to physically travel the link = **d/s** (distance/propagation speed) | milliseconds (WANs); propagation speed ≈ 2×10⁸ – 3×10⁸ m/s |

- **Transmission delay vs. propagation delay**: transmission delay depends on packet length & link rate (not distance); propagation delay depends on distance (not packet length or rate).
- **Caravan analogy**: 10 cars (bits) as a caravan (packet) passing tollbooths (routers) spaced 100 km apart. Time to push the whole caravan through a tollbooth ≈ transmission delay; time to travel between tollbooths ≈ propagation delay.

### 1.4.2 Queuing Delay and Packet Loss
- **Traffic intensity** = **La/R**, where *a* = average packet arrival rate (packets/sec), L = packet size (bits), R = link transmission rate (bits/sec).
  - If **La/R > 1** → queue grows **unboundedly**, delay → infinity. **Golden rule**: design so traffic intensity ≤ 1.
  - If **La/R ≤ 1**: delay depends on the *nature* of arrivals (periodic → little/no delay; bursty → higher delay).
  - As traffic intensity → 1, average queuing delay increases **very rapidly** (non-linearly).
- **Packet loss**: real queues have **finite capacity** — when full, a router **drops** incoming (or queued) packets. Loss fraction increases as traffic intensity increases. Lost packets may be **retransmitted** end-to-end.

### 1.4.3 End-to-End Delay
For N−1 routers, uncongested network (negligible queuing), same d_proc, R, d_prop at each hop:

$$d_{end-end} = N(d_{proc} + d_{trans} + d_{prop}) \quad (1.2)$$

- **Traceroute**: utility that sends packets with increasing TTL-like markers so each intermediate router sends back a message; reveals route + round-trip delay to each hop. (Uses 3 packets per hop by default; described in RFC 1393.) Round-trip delay to router *n* can occasionally exceed that of router *n+1* due to varying queuing delay.
- Additional delays: media access delay (e.g., sharing WiFi/cable medium) and **media packetization delay** (e.g., in VoIP — time to fill a packet with digitized speech before sending).

### 1.4.4 Throughput
- **Instantaneous throughput**: rate (bits/sec) at which the receiving host is receiving data at a given instant.
- **Average throughput**: for a file of F bits taking T seconds, average throughput = **F/T** bits/sec.
- **Two-link example**: server→router (rate Rs), router→client (rate Rc). Throughput = **min{Rs, Rc}** — the **bottleneck link's** rate.
- **N-link generalization**: throughput = **min{R1, R2, …, RN}**.
- **Core network example**: if core links are much faster than access links, throughput = min{Rs, Rc} — i.e., **access network is typically the constraining factor** in today's Internet.
- **Shared-link example**: 10 simultaneous client-server downloads sharing one common core link of rate R divided equally among them. If R is comparable to Rs/Rc, this shared link becomes the bottleneck — e.g., Rs=2 Mbps, Rc=1 Mbps, R=5 Mbps shared 10 ways → each gets only **500 kbps** (rather than the access-link-limited 1 Mbps).
- **Conclusion**: throughput depends both on the transmission rates of links along the path AND on the intervening/competing traffic.

---

## 1.5 Protocol Layers and Their Service Models

### 1.5.1 Layered Architecture
- **Airline analogy**: airline functionality can be organized into horizontal layers — ticketing, baggage, gate, takeoff/landing — each layer providing a service using the layer below it.
- **Benefits of layering**: modularity — a layer's *implementation* can change without affecting other layers, as long as the *service* (interface) it offers stays the same.
- **Layer n's service model**: what services a layer offers to the layer above; implemented by (1) actions within that layer, and (2) using services of the layer below.
- Layers can be implemented in **software, hardware, or both**:
  - Application & transport layers: almost always software (end systems).
  - Physical & link layers: typically implemented in **network interface cards** (e.g., Ethernet/WiFi cards).
  - Network layer: mixed hardware/software.
- **Drawbacks of layering**: possible duplication of functionality across layers (e.g., error recovery at multiple layers); a layer may need info only available at another layer, violating separation.

**The Internet Protocol Stack — 5 layers (bottom to top):**

| Layer | Function | Packet name | Example protocols |
|---|---|---|---|
| **Application** | Supports network applications | **Message** | HTTP, SMTP, FTP, DNS |
| **Transport** | Process-to-process data transfer between application endpoints | **Segment** | TCP, UDP |
| **Network** | Routing datagrams from source to destination host | **Datagram** | IP (+ routing protocols) |
| **Link** | Data transfer between neighboring network elements | **Frame** | Ethernet, WiFi, DOCSIS, PPP |
| **Physical** | Moving individual bits within a frame from one node to next | bits | medium-dependent (twisted pair, fiber, etc.) |

- **Application layer**: HTTP (Web), SMTP (e-mail), FTP (file transfer), DNS (name→IP translation).
- **Transport layer**: **TCP** = connection-oriented, guarantees delivery, flow control, breaks messages into **segments**, congestion control. **UDP** = connectionless, no reliability/flow/congestion control ("no-frills").
- **Network layer**: moves **datagrams**; contains **IP protocol** (only one IP protocol — all Internet devices with a network layer run it) + various **routing protocols**; often just called "the IP layer."
- **Link layer**: moves datagrams between neighboring nodes as **frames**; different links along a path can use different link-layer protocols (e.g., Ethernet on one hop, PPP on the next); may offer reliable delivery **per-link** (different from TCP's end-to-end reliability).
- **Physical layer**: moves individual **bits**; protocol depends on actual transmission medium (twisted pair vs. fiber, etc., even for the same link-layer protocol like Ethernet).

**OSI Model (7 layers)**: application, **presentation**, **session**, transport, network, data link, physical.
- Proposed by ISO in the late 1970s, before Internet protocols matured. Still referenced in some courses/textbooks due to historical influence.
- **Presentation layer**: lets communicating apps interpret data meaning — data compression, encryption, data description (format differences between machines).
- **Session layer**: delimiting/synchronizing data exchange, checkpointing/recovery.
- The Internet **lacks these two layers** — it's up to the **application developer** to decide if such services are needed and to build them into the application if so.

### 1.5.2 Encapsulation
- Data flows **down** the stack at the sender, across, and **up** the stack at the receiver; at intermediate switches/routers, it flows partway up and back down.
- **Routers implement layers 1–3** (physical, link, network); **link-layer switches implement only layers 1–2**; **hosts implement all 5 layers**. (This reflects the Internet's design philosophy of pushing complexity to the network **edge**.)
- At each layer, a packet = **header** + **payload** (the payload is typically the entire packet handed down from the layer above).
- **Encapsulation chain**:
  - Application message **M** → Transport layer adds header **Ht** → **segment**.
  - Segment → Network layer adds header **Hn** (source/destination addresses) → **datagram**.
  - Datagram → Link layer adds its header → **frame**.
- **Analogy**: interoffice memo (message) → put in interoffice envelope with recipient's name (segment, transport header) → placed inside a postal envelope with full postal address (datagram, network header) → delivered → **de-encapsulated** in reverse at the destination mailroom → memo delivered to Bob.
- A large message may be split into multiple segments (each possibly split into multiple datagrams); receiver reconstructs from constituent pieces.

---

## 1.6 Networks Under Attack

Network security is central to modern networking. Key threat categories:

### Malware
- **Malware**: malicious software that can delete files, install spyware (stealing SSNs, passwords, keystrokes), or enlist the host into a **botnet** (used for spam or DDoS attacks).
- Often **self-replicating** — spreads exponentially.
  - **Virus**: requires user interaction (e.g., opening an infected e-mail attachment) to infect a device.
  - **Worm**: enters a device **without** explicit user interaction (exploits a vulnerable network application), then scans the Internet for other vulnerable hosts to infect.

### Attacks on Servers and Infrastructure — Denial of Service (DoS)
Renders a network/host/infrastructure unusable to legitimate users. Three main categories:
1. **Vulnerability attack**: sending a few well-crafted messages exploiting a bug in a vulnerable application/OS, potentially crashing it.
2. **Bandwidth flooding**: sending a deluge of packets to clog the target's access link.
3. **Connection flooding**: establishing many half-open/open TCP connections to overwhelm the target.
- **DDoS (Distributed DoS)**: attacker controls **multiple sources** (often a botnet) to jointly flood the target — harder to detect/block than a single-source attack, since blocking one upstream source isn't sufficient.

### Packet Sniffing
- A **passive receiver** placed near a wireless transmitter (or in a wired **broadcast** environment like some Ethernet/cable networks) can capture copies of **every packet** transmitted — a **packet sniffer**.
- Can expose passwords, SSNs, trade secrets, private messages.
- Hard to detect because sniffers are **passive** (don't inject traffic).
- Best defense: **cryptography** (covered in Ch. 8).

### IP Spoofing / Masquerading
- It's easy to craft a packet with an **arbitrary (false) source address** and send it — recipients may trust the (false) source and act on the packet's contents.
- This is called **IP spoofing** — one of many ways to masquerade as another user.
- Defense requires **end-point authentication** — a mechanism to verify a message truly originates from the claimed source (Ch. 8).

### Why the Internet Is Insecure by Design
- Originally built on a model of "**a group of mutually trusting users attached to a transparent network**" — i.e., no inherent need for security by design.
  - E.g., sending a packet to anyone is the **default** capability (not something requested/granted); sender identity is taken at **face value**, not authenticated.
- Today's Internet no longer fits that trust model — hence the need for defenses against sniffing, masquerading, man-in-the-middle attacks, DDoS, malware, etc.

---

## 1.7 History of Computer Networking and the Internet

### 1.7.1 Development of Packet Switching: 1961–1972
- Telephone network (circuit-switched) was dominant in early 1960s; unsuited to **bursty** computer traffic.
- Three independent research groups pioneered **packet switching**:
  - **Leonard Kleinrock** (MIT, 1961/1964) — first published work, using queuing theory to show packet switching's effectiveness for bursty traffic.
  - **Paul Baran** (Rand Institute, 1964) — packet switching for secure military voice.
  - **Donald Davies & Roger Scantlebury** (National Physical Laboratory, UK).
- **J.C.R. Licklider** and **Lawrence Roberts** (MIT colleagues of Kleinrock) led ARPA's computer science program; Roberts published the plan for the **ARPAnet** — first packet-switched network, direct ancestor of the Internet.
- **1969 (Labor Day)**: first packet switch installed at **UCLA** (under Kleinrock); three more soon after at Stanford Research Institute (SRI), UC Santa Barbara, and University of Utah → **4-node network** by end of 1969.
  - First use: a remote login from UCLA to SRI — **crashed the system**.
- **1972**: ARPAnet grew to ~15 nodes; first public demonstration (Robert Kahn); first host-to-host protocol, **NCP** (Network Control Protocol), completed. Ray Tomlinson wrote the **first e-mail program**.

### 1.7.2 Proprietary Networks and Internetworking: 1972–1980
- Other packet-switching networks emerged: **ALOHAnet** (Hawaii, microwave), DARPA's packet-satellite & packet-radio networks, **Telenet** (BBN, commercial), **Cyclades** (France, Louis Pouzin), Tymnet, GE Information Services, **IBM's SNA**.
- **Vinton Cerf and Robert Kahn** (under DARPA sponsorship) pioneered **internetworking** — connecting networks together; coined the term "**internetting**." These principles became **TCP**.
- Early TCP combined reliable delivery + forwarding; later, **IP was split out of TCP**, and **UDP** was developed for unreliable/non-flow-controlled needs (e.g., packetized voice). By end of the 1970s, **TCP, UDP, and IP** were conceptually in place.
- **Norman Abramson** developed **ALOHAnet** — the **ALOHA protocol**, the first **multiple-access protocol**.
- **Metcalfe and Boggs** built on ALOHA to develop the **Ethernet protocol**, motivated by connecting PCs, printers, and shared disks.

### 1.7.3 A Proliferation of Networks: 1980–1990
- End of 1970s: ~200 hosts on ARPAnet. End of 1980s: ~100,000 hosts on the public Internet.
- **BITNET** (e-mail/file transfer among Northeast US universities), **CSNET** (linking researchers without ARPAnet access), **NSFNET** (1986 — access to NSF supercomputing centers; backbone grew from 56 kbps to 1.5 Mbps by decade's end).
- **January 1, 1983**: official deployment of **TCP/IP** as ARPAnet's standard host protocol, replacing NCP — a "**flag day**" event (all hosts had to switch that day).
- Late 1980s: **host-based congestion control** added to TCP (Jacobson, 1988); **DNS** developed to map human-readable names to 32-bit IP addresses.
- **Minitel** (France, early 1980s): government-sponsored public packet-switched network (X.25-based) with free terminals distributed from 1984; huge success — 20,000+ services at its peak (mid-1990s), popular in French homes a decade before most Americans knew of the Internet.

### 1.7.4 The Internet Explosion: The 1990s
- **ARPAnet decommissioned**; **1991**: NSFNET lifts commercial-use restrictions; **1995**: NSFNET itself decommissioned, backbone traffic shifts to commercial ISPs.
- **World Wide Web** invented at **CERN** by **Tim Berners-Lee** (1989–1991), building on hypertext ideas from Vannevar Bush (1945) and Ted Nelson (1960s). Developed initial **HTML, HTTP**, a Web server, and a browser.
- ~200 Web servers by end of 1993. **Marc Andreessen** & **Jim Clark** formed Mosaic Communications → **Netscape**. **1996**: Microsoft entered with its own browser → the "**browser war**" (Microsoft eventually won).
- **Four "killer applications"** by end of the millennium:
  1. **E-mail** (with attachments, Web-accessible)
  2. **The Web** (browsing + e-commerce)
  3. **Instant messaging** (with contact lists)
  4. **Peer-to-peer file sharing** (MP3s, pioneered by **Napster**)
  - (First two from the research community; last two from young entrepreneurs.)
- **1995–2001**: dot-com boom and bust — many startups IPO'd without profits/revenue, then collapsed in 2000–2001. Big winners emerged: Microsoft, Cisco, Yahoo, eBay, Google, Amazon.

### 1.7.5 The New Millennium
- Aggressive broadband deployment (cable, DSL, FTTH) → enabled video applications (YouTube, Netflix, Skype/FaceTime/Google Hangouts).
- Ubiquitous high-speed **public WiFi** and **4G cellular** → constant connectivity + location-based apps (Yelp, Tinder, Yik Yak, Waze). **Wireless devices surpassed wired devices** connecting to the Internet in **2011**.
- **Online social networks** (Facebook, Instagram, Twitter, WeChat) — massive "people networks" atop the Internet; many provide APIs as platforms for new apps/games.
- **Content/online service providers** (Google, Microsoft) built their own **private global networks**, peering directly with lower-tier ISPs to bypass the public Internet where possible → near-instantaneous search/e-mail.
- **Cloud computing**: companies run applications in the cloud (Amazon EC2, Google App Engine, Microsoft Azure) — gaining scalable compute/storage plus implicit access to high-performance private networks.

---

## 1.8 Summary (Chapter Recap)

- Covered the hardware/software components of the Internet: **network edge** (end systems, access networks, physical media) and **network core** (packet switching vs. circuit switching, network structure/hierarchy of ISPs).
- Quantified **delay** (processing, queuing, transmission, propagation), **packet loss**, and **throughput** (bottleneck-link concept).
- Introduced **protocol layering** (5-layer Internet stack: application, transport, network, link, physical) and **encapsulation** (message → segment → datagram → frame).
- Surveyed major **network security threats**: malware (viruses/worms), DoS/DDoS attacks, packet sniffing, IP spoofing — motivating why security is now central to networking, despite the Internet's original "mutual trust" design assumption.
- Traced the **history** of computer networking from 1960s packet-switching research through ARPAnet, TCP/IP standardization, the Web, the dot-com era, and today's mobile/cloud/social Internet.

> This chapter is meant to give the "forest" view before the book dives into each layer in detail, top-down: **Application (Ch. 2) → Transport (Ch. 3) → Network (Ch. 4–5) → Link (Ch. 6) → Wireless/Mobile (Ch. 7) → Security (Ch. 8)**.

---

## Key Formulas Quick Reference
- Store-and-forward delay (N links, rate R): $d_{end-to-end} = \dfrac{NL}{R}$
- Nodal delay: $d_{nodal} = d_{proc} + d_{queue} + d_{trans} + d_{prop}$
- Transmission delay: $d_{trans} = L/R$
- Propagation delay: $d_{prop} = d/s$
- Traffic intensity: $La/R$ (must be ≤ 1 for stability)
- End-to-end delay (no queuing): $d_{end-end} = N(d_{proc} + d_{trans} + d_{prop})$
- Throughput (single path): $\min\{R_1, R_2, \ldots, R_N\}$
- Average throughput: $F/T$ (F = file bits, T = transfer time)
