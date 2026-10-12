# Chapter 6 — The Link Layer and LANs
### (Kurose & Ross, *Computer Networking: A Top-Down Approach*, 7th Ed.)

---

## Chapter Overview

- Chapters 4–5 covered how datagrams get across the whole path. This chapter asks how a datagram crosses **one individual link** between adjacent nodes.
- Questions answered: How are datagrams encapsulated in link-layer frames? Do different links use different link-layer protocols? How are transmission conflicts on broadcast links resolved? How does link-layer addressing relate to IP addressing? What is the difference between a switch and a router?
- **Two fundamentally different types of link-layer channel:**
  1. **Broadcast channels**: many hosts share one channel (wireless LANs, satellite networks, hybrid fiber-coax cable access). Needs a **medium access (multiple access) protocol** to coordinate frame transmission — sometimes a central controller coordinates, sometimes the hosts coordinate themselves.
  2. **Point-to-point links**: e.g., between two routers over a long-distance link, or between an office PC and the nearby Ethernet switch. Coordination is simpler. (The book's website covers **PPP**, the Point-to-Point Protocol, used from dial-up to high-speed fiber frame transport.)
- Topics: error detection/correction, multiple access networks, switched LANs including **Ethernet** (by far the most prevalent wired LAN), VLANs, MPLS, data center networks. WiFi is a link-layer topic but is deferred to **Chapter 7**.

---

## 6.1 Introduction to the Link Layer

- **Terminology**: a **node** is any device that runs a link-layer (layer 2) protocol: hosts, routers, switches, WiFi access points. A **link** is a communication channel connecting adjacent nodes along the path.
- A datagram must be moved across **each individual link** on the end-to-end path. Over a given link, the transmitting node **encapsulates the datagram in a link-layer frame** and transmits it.
- **Example (Figure 6.1)**: a wireless host sending to a server in a company network crosses **six links**: WiFi (host → access point), Ethernet (AP → switch), switch → router, router → router, Ethernet (router → switch), Ethernet (switch → server).
- **Transportation analogy**: a tourist going Princeton → Lausanne via limousine (to JFK), plane (to Geneva), train (to Lausanne). Tourist = **datagram**; each transportation segment = **link**; transportation mode = **link-layer protocol**; travel agent = **routing protocol**. Each segment is run by a different company using a different mode, yet each moves the passenger between adjacent locations.

### 6.1.1 Services Provided by the Link Layer
Basic service: move a datagram from one node to an **adjacent** node over one link. Possible services (vary by protocol):
- **Framing**: almost all protocols encapsulate each datagram in a **frame** = data field (the datagram) + header fields. Frame structure is specified by the link-layer protocol.
- **Link access**: a **MAC (medium access control) protocol** specifies the rules for transmitting a frame onto the link.
  - Point-to-point (one sender, one receiver): MAC is simple or nonexistent (send whenever the link is idle).
  - Shared broadcast link: the **multiple access problem** — the MAC protocol coordinates transmissions of many nodes.
- **Reliable delivery**: guarantees to move each datagram across the link without error, using **acknowledgments and retransmissions** (as in Section 3.4). Used mainly on links with **high error rates (e.g., wireless)** to fix errors locally rather than forcing end-to-end retransmission. Considered unnecessary overhead on **low bit-error links** (fiber, coax, many twisted-pair), so many wired protocols don't provide it.
- **Error detection and correction**: bit errors come from **signal attenuation and electromagnetic noise**. The sender adds error-detection bits; the receiver checks. Link-layer detection is **more sophisticated than the Internet checksum and implemented in hardware**. **Error correction** also determines *where* the errors are and fixes them.

### 6.1.2 Where Is the Link Layer Implemented?
- Mostly in a **network adapter / network interface card (NIC)**. At its heart is the **link-layer controller**, usually a single special-purpose chip implementing framing, link access, error detection, etc. — much of it in **hardware**. (Examples: Intel 710 adapter → Ethernet; Atheros AR5006 → 802.11 WiFi.)
- Before the late 1990s, adapters were separate cards (PCMCIA or PCI); now often integrated on the motherboard (**LAN-on-motherboard**).
- **Sending side**: the controller takes a datagram from host memory, encapsulates it in a frame (filling in fields), and transmits it following the link-access protocol. **Receiving side**: the controller receives the whole frame, extracts the datagram. If error detection is used, the **sending controller sets the error-detection bits and the receiving controller checks them**.
- Part of the link layer is **software on the host CPU** (assembling link-layer addressing info, activating the controller hardware, responding to controller interrupts, handling errors, passing the datagram up). So the link layer is "**the place in the protocol stack where software meets hardware**."

---

## 6.2 Error-Detection and -Correction Techniques

- Setup (Figure 6.3): data **D** is augmented with **EDC** (error-detection-and-correction) bits. D covers not only the datagram but also link-level addressing, sequence numbers, and other header fields. The receiver gets **D′ and EDC′**, which may differ from the original because of bit flips.
- The receiver's task is to decide whether D′ equals D. Note the wording: the receiver asks whether an error is **detected**, not whether an error **has occurred** — undetected bit errors are still possible. More sophisticated schemes have a smaller chance of missing errors but cost more computation and more bits.
- Three techniques: **parity checks**, **checksumming**, **CRC**.

### 6.2.1 Parity Checks
- **Single parity bit**: with d data bits, add one bit. **Even parity**: choose the bit so the total number of 1s in the d+1 bits is **even**. **Odd parity**: total is odd.
  - Receiver counts 1s. If an odd number of 1s appears under even parity, it knows an **odd number of bit errors** occurred. An **even number of errors goes undetected**.
  - If errors were independent and rare, one bit might suffice, but errors often come in **bursts**; with burst errors the probability of undetected errors with single-bit parity can approach **50%**.
- **Two-dimensional parity**: arrange the d bits into **i rows and j columns**; compute a parity bit for **each row and each column**, giving **i + j + 1** parity bits.
  - A **single bit error** flips both its row parity and its column parity, so the receiver can **locate and correct** it (also works for an error in a parity bit itself).
  - It can also **detect (but not correct) any combination of two errors** in a packet.
- **Forward error correction (FEC)**: the ability of the receiver to both detect and correct errors. Used in audio CDs; valuable in networks because it reduces sender retransmissions and gives **immediate correction** without waiting a round trip for a NAK + retransmission — important for real-time applications and for links with **long propagation delay (e.g., deep space)**.

### 6.2.2 Checksumming Methods
- Treat the data as a sequence of **k-bit integers** and sum them. The **Internet checksum** treats bytes as **16-bit integers**, sums them, and stores the **1's complement of the sum**. The receiver sums the received data **including the checksum**; if the result is **all 1s**, no error is detected, otherwise an error is indicated (RFC 1071).
- In TCP/UDP the checksum covers header + data; in IP it covers the **IP header only**.
- Requires little overhead (16 bits) but gives **relatively weak protection** compared to CRC.
- **Why checksum at the transport layer but CRC at the link layer?** Transport-layer error detection is implemented in **software** (needs to be simple and fast); link-layer detection is in **dedicated hardware** in adapters, which can quickly do the more complex CRC.

### 6.2.3 Cyclic Redundancy Check (CRC)
- CRC codes are also called **polynomial codes** (the bit string is viewed as a polynomial with 0/1 coefficients).
- **Setup**: d-bit data **D**. Sender and receiver agree on an **(r+1)-bit generator G**, whose **leftmost bit is 1**. The sender picks **r additional bits R** and appends them so the resulting **d+r bit pattern is exactly divisible by G** using **modulo-2 arithmetic**. The receiver divides the d+r received bits by G: **nonzero remainder = error detected**.
- **Modulo-2 arithmetic**: no carries in addition, no borrows in subtraction, so **addition and subtraction are identical and equal XOR**. Examples: 1011 XOR 0101 = 1110; 1001 XOR 1101 = 0100. Multiplication by 2^k left-shifts the bits by k.
- **Computing R**: we want D·2^r XOR R = nG for some n. XORing R to both sides gives D·2^r = nG XOR R, so **R = remainder of (D·2^r) / G**.
- **Worked example (Figure 6.7)**: D = 101110 (d = 6), G = 1001 (r = 3) → R = **011**, so the 9 transmitted bits are **101110 011**.
- **Standards**: international standards exist for **8-, 12-, 16-, and 32-bit generators**. **CRC-32** (used in many IEEE link-level protocols) uses G = `100000100110000010001110110110111`.
- **Detection power**: each standard detects **burst errors of fewer than r+1 bits** (i.e., all bursts of r bits or fewer); a longer burst is detected with probability **1 − 0.5^r** (under appropriate assumptions); and each detects **any odd number of bit errors**.

---

## 6.3 Multiple Access Links and Protocols

- **Point-to-point link**: single sender and receiver (protocols: **PPP**, **HDLC**). **Broadcast link**: many sending and receiving nodes share one channel; when one node transmits, the channel **broadcasts** it and every other node gets a copy (Ethernet, wireless LANs). LANs are geographically concentrated (a building or campus).
- **The multiple access problem**: coordinating access of many nodes to one shared broadcast channel. **Cocktail party / classroom analogy**: who gets to talk and when? Human "protocols": give everyone a chance, don't monopolize, raise your hand, don't interrupt, etc.
- Because every node can transmit, **two or more simultaneous transmissions collide** at the receivers. Typically no receiver can make sense of any of the colliding frames, so **all frames in a collision are lost** and the channel is wasted during the collision. Hence the need for **multiple access protocols**.
- Every multiple access protocol falls into one of **three categories**: **channel partitioning**, **random access**, **taking turns**.

**Desirable properties** of a multiple access protocol on a channel of rate R bps:
1. With only **one** active node, that node gets throughput **R**.
2. With **M** active nodes, each gets throughput of about **R/M** (on average over a suitable interval).
3. **Decentralized** — no master node that is a single point of failure.
4. **Simple**, hence cheap to implement.

### 6.3.1 Channel Partitioning Protocols
- **TDM (time-division multiplexing)**: time is divided into **frames**, each divided into **N slots**; each of N nodes owns one slot per frame and transmits its packet in its slot. (The TDM time frame is different from a link-layer frame.)
- **FDM (frequency-division multiplexing)**: the R bps channel is split into N frequency bands, each of **R/N** bandwidth, one per node.
- Pros: **no collisions**, perfectly **fair** (R/N each).
- **Cons (shared by TDM and FDM)**: a node is limited to **R/N even when it is the only active node**; and (TDM) a node always **waits for its turn**.
- **CDMA (code division multiple access)**: each node gets a **unique code** to encode its bits. If codes are chosen well, **different nodes can transmit simultaneously** and receivers can still decode the intended sender (knowing its code). Used in military systems (anti-jamming) and widely in **cellular telephony**; technical details are in **Chapter 7**.

### 6.3.2 Random Access Protocols
- A transmitting node **always transmits at the full channel rate R**. On a **collision**, each involved node retransmits **repeatedly until its frame gets through**, but **waits a random delay** before each retransmission. Delays are chosen independently, so one node may pick a short enough delay to get through.
- Protocols covered: **ALOHA** (slotted and pure) and **CSMA** (and CSMA/CD).

#### Slotted ALOHA
**Assumptions**: all frames are exactly **L bits**; time is divided into slots of **L/R seconds** (one frame time); nodes start transmitting **only at slot beginnings**; nodes are **synchronized**; if two or more frames collide in a slot, **all nodes detect the collision before the slot ends**.

**Operation** (retransmission probability **p**, 0 ≤ p ≤ 1):
- With a fresh frame, wait for the next slot and transmit the whole frame.
- No collision → success. Collision → **retransmit in each subsequent slot with probability p** until it succeeds (like tossing a biased coin each slot; nodes toss independently).

**Pros**: a lone active node can transmit at full rate R; highly decentralized (only slot synchronization needed); very simple.
**Cons**: when many nodes are active, some slots have **collisions** (wasted) and some are **empty**; only slots with **exactly one transmitter** (**successful slots**) are useful.

**Efficiency** = long-run fraction of successful slots with many active nodes, each with many frames to send.
- Simplified model: each of N nodes transmits in a slot with probability p. Probability a given node succeeds = p(1−p)^(N−1). Probability that **some** node succeeds = **N·p·(1−p)^(N−1)**.
- Maximizing over p and letting N → ∞ gives maximum efficiency **1/e ≈ 0.37**. So at best **37% of slots do useful work** (effective rate 0.37R), **37% are empty**, and **26% have collisions**. (A 100 Mbps slotted ALOHA channel delivers well under 37 Mbps of useful throughput.)

#### (Pure) ALOHA
- The original ALOHA protocol was **unslotted and fully decentralized**: a node **immediately transmits** a frame when it arrives. If it collides, the node (after finishing the collided frame) **retransmits with probability p**; otherwise it waits one frame time and then transmits with probability p (or stays idle another frame time with probability 1−p).
- For a frame starting at t0 to succeed, no other node may start in **[t0−1, t0]** nor during the frame itself, so success probability for a node = **p(1−p)^(2(N−1))**.
- Maximum efficiency is **1/(2e) ≈ 0.18** — **exactly half** of slotted ALOHA. That is the price of full decentralization (no synchronization).
- **Case History — Norm Abramson and ALOHAnet**: at the University of Hawaii (1969), Abramson designed a radio packet-switching network over the Hawaiian islands with a central host and secondary nodes on two channels (downlink broadcast from the central host, with ACKs; uplink shared by secondary hosts). Collisions on the uplink led to **pure ALOHA**. In 1970 ALOHAnet was connected to the ARPAnet. It inspired **Bob Metcalfe**, who a few years later modified ALOHA to create **CSMA/CD and Ethernet**.

#### Carrier Sense Multiple Access (CSMA)
ALOHA ignores what other nodes are doing (like a boorish partygoer). Two polite-conversation rules:
1. **Listen before speaking** → **carrier sensing**: a node listens to the channel before transmitting; if busy, it waits until it detects no transmission for a short time.
2. **If someone else starts talking at the same time, stop talking** → **collision detection**: a transmitting node keeps listening; if it detects interference it **stops** and waits a random time before trying again.
- These give the **CSMA** and **CSMA/CD** families.

**Why do collisions still happen with carrier sensing?** (Space-time diagram, Figure 6.12) Because of **propagation delay**. Node B starts transmitting at t0; at t1 > t0 node D has a frame and senses the channel idle because B's bits **haven't reached D yet**, so D transmits and the frames collide. The **end-to-end channel propagation delay** is crucial: the longer it is, the greater the chance that a carrier-sensing node can't yet sense an ongoing transmission.

#### CSMA/CD
- Without collision detection both nodes transmit their whole damaged frames. **With collision detection, a node aborts shortly after detecting a collision**, saving channel time.
- **Adapter operation**:
  1. Get a datagram from the network layer, prepare a frame, put it in the adapter buffer.
  2. If the channel is **idle**, start transmitting; if **busy**, wait until no signal energy is sensed, then transmit.
  3. While transmitting, monitor for signal energy from other adapters.
  4. If the whole frame is sent with no interference detected, done; if interference is detected, **abort**.
  5. After aborting, wait a **random** amount of time and go back to step 2.
- **Why random (not fixed) wait?** Two nodes waiting the same fixed time would collide forever. What interval size? Should be **short when few nodes collide, long when many do**.
- **Binary exponential backoff** (used in **Ethernet** and **DOCSIS**): after a frame has suffered **n collisions**, choose K at random from **{0, 1, 2, …, 2^n − 1}**. For Ethernet the wait is **K × 512 bit times**, and n is **capped at 10**.
  - 1st collision: K ∈ {0,1} (each 0.5). K=1 → wait 512 bit times (**5.12 µs at 100 Mbps**).
  - 2nd collision: K ∈ {0,1,2,3}. 3rd: {0,…,7}. **≥10 collisions: {0,…,1023}**.
- Each **new frame** runs CSMA/CD afresh, ignoring recent collisions, so a node with a new frame can sneak in while others are backing off.
- **CSMA/CD efficiency** (long-run fraction of time frames are sent without collision, many active nodes). Let **d_prop** = max propagation time between any two adapters and **d_trans** = time to transmit a max-size frame (about 1.2 ms on 10 Mbps Ethernet). Approximation (derivation beyond the text):

$$\text{Efficiency} = \frac{1}{1 + 5\, d_{prop} / d_{trans}}$$

  - As **d_prop → 0**, efficiency → 1 (colliding nodes abort immediately). As **d_trans → very large**, efficiency → 1 (a frame holds the channel a long time).

### 6.3.3 Taking-Turns Protocols
ALOHA and CSMA have property 1 (full rate for a lone node) but not property 2 (about R/M each). Taking-turns protocols aim for both.
- **Polling protocol**: one node is the **master**, polling each node round-robin: it tells node 1 it may send up to some maximum number of frames, then node 2, etc. (it detects that a node has finished by the lack of signal).
  - Eliminates collisions and empty slots → higher efficiency.
  - **Drawbacks**: **polling delay** (a lone active node transmits below R because the master must poll inactive nodes too); and **if the master fails, the whole channel fails**. Examples: **802.15 and Bluetooth**.
- **Token-passing protocol**: **no master**. A small special **token** frame circulates among nodes in a fixed order. A node holding the token sends up to a maximum number of frames if it has any, then forwards the token (or forwards it immediately if it has none).
  - Decentralized and efficient, but **one node failing can crash the channel**, and if a node forgets to release the token, **recovery** is needed. Examples: **FDDI**, **IEEE 802.5 token ring**.

### 6.3.4 DOCSIS: Link-Layer Protocol for Cable Internet Access
A case study that combines **all three classes** of multiple access protocols.
- A cable access network connects thousands of **cable modems** to a **CMTS** at the head end. **DOCSIS** (Data-Over-Cable Service Interface Specifications) defines the architecture and protocols.
- **FDM** divides downstream (CMTS → modems) and upstream (modems → CMTS) into multiple frequency channels:
  - **Downstream** channel: **6 MHz wide, ~40 Mbps** max per channel.
  - **Upstream** channel: up to **6.4 MHz wide, ~30 Mbps** max.
  - Each channel is a **broadcast channel**.
- **Downstream**: only the CMTS transmits → **no multiple access problem**.
- **Upstream**: many modems share the same channel → collisions possible. Each upstream channel is divided into **time intervals (TDM-like) containing mini-slots**. The CMTS **explicitly grants** specific modems specific mini-slots via a control message called a **MAP message** sent on a downstream channel → **no collisions in granted mini-slots**.
- How does the CMTS learn who has data? Modems send **mini-slot-request frames** during a special set of mini-slots, using **random access**, so these **can collide**. A modem **cannot sense the channel or detect collisions**; it infers a collision if it gets **no response in the next downstream control message**, then uses **binary exponential backoff** to defer the retry. At low load a modem may even send data in slots nominally meant for requests.
- So cable access has **FDM + TDM + random access + centrally allocated slots** all in one network.

---

## 6.4 Switched Local Area Networks

- A switched LAN (Figure 6.15) interconnects departments, servers, and a router with several **link-layer switches**. These switch **link-layer frames** (not datagrams), **don't recognize network-layer addresses**, and **don't use routing algorithms like RIP/OSPF**; they use **link-layer (MAC) addresses**.
- Order of coverage: link-layer addressing and ARP (6.4.1) → Ethernet (6.4.2) → link-layer switches (6.4.3) → VLANs (6.4.4).

### 6.4.1 Link-Layer Addressing and ARP

#### MAC addresses
- It is **adapters (interfaces)**, not hosts/routers, that have link-layer addresses. A host/router with several interfaces has several MAC addresses (just as several IP addresses). **Link-layer switch interfaces don't have MAC addresses** (the switch is transparent: hosts never address frames to it).
- Called a **LAN address, physical address, or MAC address**. For Ethernet and 802.11 it is **6 bytes** (2^48 possible addresses), written in **hex**, one pair per byte (e.g., `49-BD-D2-C7-56-2A`). Designed to be permanent but can be changed in software.
- **Uniqueness**: managed by the **IEEE**. A manufacturer buys a block of **2^24 addresses** (the first 24 bits fixed by IEEE, the manufacturer assigns the last 24 bits).
- **Flat structure** (not hierarchical) and **does not change** wherever the adapter goes. **IP addresses are hierarchical** and must change when a host moves to another network. Analogy: **MAC = social security number**; **IP = postal address**.
- **Receiving**: an adapter compares the frame's destination MAC with its own; match → pass the datagram up; no match → discard (so the host isn't interrupted).
- **MAC broadcast address**: 48 consecutive 1s = **FF-FF-FF-FF-FF-FF**; delivered to all adapters on the LAN.
- *Principles in Practice — keeping layers independent*: separate MAC addresses exist because (1) LANs must support **arbitrary network-layer protocols** (not just IP; e.g., IPX, DECnet); (2) if adapters used network-layer addresses, those would have to be reconfigured every time the adapter moved or powered up; (3) without adapter addresses, **every frame would interrupt the host**. Three address types: **host names (application), IP addresses (network), MAC addresses (link)**.

#### Address Resolution Protocol (ARP, RFC 826)
- Translates **IP address → MAC address**. Analogous to DNS (hostname → IP), but with one big difference: **DNS resolves names anywhere in the Internet; ARP resolves only for hosts and router interfaces on the SAME subnet** (a node in California can't ARP for one in Mississippi).
- Each host/router has an **ARP table** of IP → MAC mappings with a **TTL** (typical expiration **20 minutes**). The table need not contain every node.
- **Operation** (host 222.222.222.220 wants to send to 222.222.222.222 on the same subnet, no table entry):
  1. The sender builds an **ARP packet** with sending and receiving IP and MAC addresses. Query and response have the **same format**.
  2. The ARP query is sent in a frame addressed to the **MAC broadcast address FF-FF-FF-FF-FF-FF** ("shouting in a crowded room").
  3. All adapters pass it up to ARP; only the node whose **IP address matches** replies with an ARP response containing the mapping.
  4. The sender updates its ARP table and sends the datagram in a frame addressed to the learned MAC.
- Notes: the **query is broadcast, but the response is sent in a standard (unicast) frame** (the responder knows the asker's MAC from the query). ARP is **plug-and-play** — tables build automatically, and entries for departed hosts eventually expire.
- **Layer position**: ARP packets ride inside link-layer frames, but contain both link-layer and network-layer addresses — ARP **straddles the boundary** between link and network layers.

#### Sending a datagram off the subnet (Figure 6.19)
- Two subnets joined by a router: Subnet 1 = 111.111.111/24, Subnet 2 = 222.222.222/24. A router has **one IP address, one ARP module, and one adapter per interface**.
- Host 111.111.111.111 sending to 222.222.222.222: the frame's destination MAC must be that of the **first-hop router's interface** (111.111.111.110, MAC E6-E9-00-17-BB-4B), **not** the final destination's MAC. (Using the destination's MAC would mean no adapter on Subnet 1 accepts the frame — "datagram heaven.") The sender gets the router's MAC via ARP.
- The router receives the frame, passes the datagram to its network layer, looks up its **forwarding table** to find the outgoing interface (222.222.222.220), and builds a **new frame** for Subnet 2 whose destination MAC *is* the final host's — obtained via ARP from that interface.
- Key takeaway: **IP addresses stay the same end to end; MAC addresses change hop by hop.**

### 6.4.2 Ethernet
- Ethernet has "pretty much taken over" wired LANs and is "to local area networking what the Internet has been to global networking."
- **Why it won**: (1) first widely deployed high-speed LAN, so admins knew it; (2) token ring, FDDI, ATM were **more complex and costly**; (3) Ethernet kept producing **equal or higher speed** versions; **switched Ethernet** (early 1990s) boosted effective rates; (4) huge popularity made hardware a **cheap commodity**.
- **History**: invented in the mid-1970s by **Bob Metcalfe and David Boggs**, originally on a **coaxial bus** (a broadcast LAN) — buses persisted into the mid-1990s. By the late 1990s: **hub-based star** topology. A **hub** is a **physical-layer device acting on individual bits**: it re-creates each bit, boosts its energy, and sends it on all other interfaces, so it is still a broadcast LAN where simultaneous frames collide. In the early 2000s the hub was replaced by a **switch** — **collision-less** and a true **store-and-forward** switch, operating only through layer 2.
- **Case History — Bob Metcalfe and Ethernet**: after studying ARPAnet and visiting Abramson's ALOHAnet group for three months, Metcalfe at **Xerox PARC** (with Boggs) invented Ethernet to network Alto computers cheaply. The original ran at **2.94 Mbps**, linking up to **256 hosts** up to **one mile** apart. He built an alliance of **Xerox, Digital, Intel** for a **10 Mbps** standard ratified by IEEE; in **1979** he founded **3Com**, which sold Ethernet cards for IBM PCs.

#### Ethernet frame structure (Figure 6.20)
Example: adapter A (AA-AA-AA-AA-AA-AA) sends an IP datagram to adapter B (BB-BB-BB-BB-BB-BB). Six fields:
| Field | Size | Purpose |
|---|---|---|
| **Preamble** | 8 bytes | First 7 bytes = `10101010`, last byte = `10101011`. The first 7 **wake up receivers and synchronize their clocks** to the sender's (rates drift from nominal 10 Mbps/100 Mbps/1 Gbps). The last two bits (consecutive 1s) signal that the important data is coming |
| **Destination address** | 6 bytes | MAC of the destination. Adapter B accepts the frame if this equals its MAC **or the broadcast address**; otherwise discards |
| **Source address** | 6 bytes | MAC of the sender |
| **Type** | 2 bytes | **Multiplexes network-layer protocols** (IP, IPX, AppleTalk, ARP, …). ARP's type is **0806 (hex)**. Analogous to the IP protocol field and transport port numbers — all "glue" a layer to the one above |
| **Data** | **46–1500 bytes** | Carries the datagram. **MTU = 1500 bytes** (larger datagrams must be fragmented). Minimum 46 bytes, so shorter datagrams are **stuffed (padded)**; the network layer removes the stuffing using the IP length field |
| **CRC** | 4 bytes | Lets the receiver detect bit errors |

- **Connectionless**: no handshake between adapters before sending (layer-2 analog of IP's datagram service and UDP).
- **Unreliable**: the receiver runs the CRC but sends **no ACK or NAK**; failed frames are simply **discarded**, so the datagram stream may have **gaps**. This keeps Ethernet simple and cheap. Whether the application sees gaps depends on transport: **UDP → yes; TCP → TCP retransmits** (so Ethernet effectively "retransmits," unaware).

#### Ethernet technologies (802.3)
- Names like **10BASE-T, 100BASE-T, 1000BASE-LX, 10GBASE-T, 40GBASE-T**: first part = **speed** (10 Mbps, 100 Mbps, 1 Gbps, 10G, 40G); **BASE** = **baseband** (media carries only Ethernet traffic); last part = **physical medium** (**T = twisted-pair copper**, also coax and fiber). Ethernet is both a link-layer and physical-layer specification.
- Early **10BASE-2 and 10BASE-5**: 10 Mbps over two types of **coaxial cable**, each limited to **500 m**; longer runs used **repeaters** (physical-layer devices that regenerate the signal). Coax suits the broadcast view; CSMA/CD solves the multiple access problem.
- Today, nodes connect to a switch by point-to-point twisted-pair or fiber.
- **100 Mbps** (mid-1990s): same MAC protocol and frame format, with new physical layers for copper (**100BASE-T**) and fiber (**100BASE-FX, -SX, -BX**). Limited to **100 m over twisted pair** and **several km over fiber**.
- **Gigabit Ethernet (IEEE 802.3z)**: same frame format, **backward compatible** with 10BASE-T/100BASE-T; supports **point-to-point links (switches)** and **shared broadcast channels (hubs, called "buffered distributors")**; uses **CSMA/CD on shared channels** (with severely limited node distance to stay efficient); allows **full-duplex** operation on point-to-point links; initially fiber, now also **category 5 UTP**. (The text also notes 40 Gigabit Ethernet keeps full compatibility.)
- **Is a MAC protocol still needed?** In switched Ethernet, a switch never forwards more than one frame onto an interface at a time and links are **full-duplex**, so there are **no collisions** and no need for a MAC protocol. Through all the changes (3 orders of magnitude of speed, switched, many media) the one **constant** has been **Ethernet's frame format**.

### 6.4.3 Link-Layer Switches
- A switch receives link-layer frames and forwards them onto outgoing links. It is **transparent** to hosts and routers (they address frames to other hosts, not to the switch). Output interfaces have **buffers** for temporary overload.

#### Filtering and forwarding
- **Filtering**: deciding whether to forward a frame to some interface or drop it. **Forwarding**: deciding which interface(s) a frame goes to and moving it there. Both use the **switch table**.
- **Switch table entry**: (1) **MAC address**, (2) **interface** leading toward that address, (3) **time** the entry was made. It need not contain all hosts.
- Switches forward on **MAC addresses**, and a traditional switch table is **built very differently** from a router's forwarding table.
- For a frame with destination MAC DD-DD-DD-DD-DD-DD arriving on interface x, **three cases**:
  1. **No entry** → **broadcast**: forward copies to all interfaces except x.
  2. **Entry says interface x** (same segment it arrived from) → **filter**: discard (the frame was already broadcast on that segment).
  3. **Entry says interface y ≠ x** → **forward** to the output buffer for y.

#### Self-learning
Switch tables are built **automatically, dynamically, autonomously** — no admin and no configuration protocol:
1. The table is **initially empty**.
2. For each incoming frame, the switch records its **source MAC address**, the **interface it arrived on**, and the **current time** (learns which segment the sender is on).
3. Entries are **deleted after an aging time** with no frames from that source (so a replaced PC's old MAC eventually disappears). Example: aging time 60 minutes; no frames from 62-FE-F7-11-89-A3 between 9:32 and 10:32 → removed at 10:32.
- Switches are **plug-and-play** and **full-duplex**.

#### Properties of switching (vs. broadcast buses/hubs)
- **No collisions** — switches buffer frames and never put more than one frame on a segment at a time; aggregate throughput = **sum of interface rates**.
- **Heterogeneous links** — links can differ in speed and medium (e.g., 1 Gbps copper, 100 Mbps fiber, 100BASE-T), good for mixing legacy and new equipment.
- **Management** — a switch can detect a **"jabbering adapter"** that endlessly sends frames and disconnect it; a cable cut disconnects only that host; switches gather statistics (bandwidth, collisions, traffic types).

**Focus on Security — switch poisoning**
- On a switched LAN a sniffer on host C normally can't see A-to-B frames if B is in the table. But the switch **broadcasts frames whose destinations aren't in its table**, and everyone sees **FF-FF-FF-FF-FF-FF broadcasts**.
- **Switch poisoning attack**: flood the switch with frames carrying **many bogus source MAC addresses**, filling the table with bogus entries so legit hosts get no room; the switch then **broadcasts most frames**, which the sniffer can pick up. It's an involved attack, so switches are significantly less vulnerable to sniffing than hubs and wireless LANs.

#### Switches versus routers
- Both are store-and-forward. **Router = layer 3** (network addresses); **switch = layer 2** (MAC addresses). (With match-plus-action/OpenFlow, a device can do both.)
- **Switch pros**: **plug-and-play**; **high filtering/forwarding rates** (process only up to layer 2). **Switch cons**: active topology restricted to a **spanning tree** (to prevent cycling of broadcast frames); large switched networks need **large ARP tables** and generate heavy ARP traffic; vulnerable to **broadcast storms** (a malfunctioning host sending endless broadcast frames can collapse the network).
- **Router pros**: hierarchical IP addressing means packets **don't cycle** on redundant paths and aren't limited to a spanning tree → can use the **best path** and build rich topologies; provide **firewall protection against layer-2 broadcast storms**. **Router cons**: **not plug-and-play** (IP addresses must be configured); **larger per-packet processing time**.
- **Table 6.1**:

| | Hubs | Routers | Switches |
|---|---|---|---|
| Traffic isolation | No | Yes | Yes |
| Plug and play | Yes | No | Yes |
| Optimal routing | No | Yes | No |

- **When to use which**: small networks (a few hundred hosts, a few LAN segments) → **switches** suffice. Larger networks (thousands of hosts) → **routers inside the network too**, for stronger traffic isolation, broadcast-storm control, and smarter routes.

### 6.4.4 Virtual Local Area Networks (VLANs)
**Three drawbacks of the hierarchical switched LAN in Figure 6.15:**
1. **Lack of traffic isolation**: broadcast traffic (ARP, DHCP, frames to not-yet-learned destinations) must traverse the whole institutional network — bad for performance and also for **security/privacy** (e.g., executives vs. employees running sniffers).
2. **Inefficient use of switches**: 10 small groups would need 10 switches, even though one 96-port switch would hold everyone (but without isolation).
3. **Managing users**: moving an employee between groups requires **re-cabling**; employees in two groups are even harder.

**VLANs** let a switch define **multiple virtual LANs over one physical LAN infrastructure**; hosts in a VLAN communicate as if only they were connected to the switch.
- **Port-based VLAN**: the manager divides switch **ports** into groups; each group is a VLAN and a **broadcast domain** (broadcast from one port only reaches ports in the same VLAN). Example (Figure 6.25): a 16-port switch with ports 2–8 in the **EE VLAN**, ports 9–15 in the **CS VLAN** (ports 1 and 16 unassigned). Moving the user on port 8 to CS only needs **reconfiguration**, no recabling. The switch keeps a **port-to-VLAN mapping table** and delivers frames only between ports of the same VLAN.
- **Connecting different VLANs**: a router is needed. Connect a switch port (e.g., port 1) to an **external router** and make it belong to both VLANs; EE → CS traffic goes up to the router and back down the other VLAN. Vendors sell **combined VLAN switch + router** devices.
- **Interconnecting VLAN switches**:
  - Naive way: one dedicated cable per VLAN between switches — **doesn't scale** (N VLANs need N ports on each switch).
  - **VLAN trunking**: one special **trunk port** on each switch belongs to **all VLANs** and carries frames of any VLAN. The **IEEE 802.1Q** extended frame format adds a **4-byte VLAN tag** to the Ethernet header, added by the sending-side switch and parsed/removed by the receiving-side switch.
  - **VLAN tag**: **2-byte TPID** (Tag Protocol Identifier, fixed hex value **81-00**) + **2-byte Tag Control Information** containing a **12-bit VLAN identifier** and a **3-bit priority field** (like IP TOS).
- **Other VLAN types**: **MAC-based VLANs** (manager lists the MAC addresses in each VLAN), VLANs based on **network-layer protocols**, and VLANs extended **across IP routers** (islands of LANs forming one global VLAN).

---

## 6.5 Link Virtualization: A Network as a Link Layer

- Evolution of the idea of a "link": a physical wire → a shared channel (radio, etc.) → a complex **switched infrastructure** (Ethernet, VLANs). Hosts still see it as just a link-layer channel.
- Example: in a **dial-up** connection the "link" is the whole **telephone network** — the Internet **virtualizes** it as a simple wire. (Overlay networks do the reverse: view the Internet as a link.)
- **MPLS** is treated as a link-layer technology that interconnects IP devices. Unlike circuit-switched telephone networks, MPLS is itself a **packet-switched, virtual-circuit network** with its own packet formats and forwarding behavior.

### 6.5.1 Multiprotocol Label Switching (MPLS)
- Evolved from mid-to-late-1990s industry efforts to **speed up IP router forwarding** by borrowing the **fixed-length label** idea from virtual-circuit networks. The goal was **not to replace** IP's destination-based forwarding but to **augment** it: selectively label datagrams and forward on labels when possible, working hand-in-hand with IP addressing and routing (IETF: **RFC 3031, 3032**).
- **MPLS header** sits **between the link-layer (e.g., Ethernet) header and the IP header**. Fields: the **label**, **3 experimental bits**, a single **S bit** (marks the end of a stack of MPLS headers), and a **TTL**.
- Only works between **MPLS-capable routers**, also called **label-switched routers**: they forward by **looking up the MPLS label** in their forwarding table and passing the frame to the output interface — **without extracting the IP destination or doing a longest-prefix match**.
- **Example (Figure 6.29)**: R1–R4 are MPLS-capable; R5, R6 are standard IP routers. R1 advertises it can reach destination A with MPLS label **6**; R3 advertises it can reach A (label **10**) and D (label **12**); R2 advertises A (label **8**). R4 therefore has **two MPLS paths to A**: via interface 0 with outbound label 10, and via interface 1 with label 8. The MPLS routers connect IP devices R5, R6, A, D "in much the same way a switched LAN or ATM network can," without touching the IP header.
- **Signaling**: label distribution uses an extension of RSVP called **RSVP-TE** (RFC 3209, RFC 3468). Link-state info (e.g., unreserved bandwidth) is flooded by **extended OSPF**. The actual path-computation algorithms are **not standardized (vendor-specific)**.
- **Real advantage**: not speed but **traffic management**. R4's two paths to A can't be expressed in plain IP routing (which gives one least-cost path). MPLS enables **traffic engineering** — forcing some traffic to a destination along one path and the rest along another. Other uses: **fast restoration** (precomputed failover paths on link failure) and **VPNs** (isolating a customer's resources and addressing across an ISP's MPLS network). MPLS is "the Swiss Army knife of Internet traffic engineering."

---

## 6.6 Data Center Networking

- Companies like Google, Microsoft, Facebook, Amazon build **massive data centers** with tens to hundreds of thousands of hosts, running many cloud applications (search, e-mail, social networking, e-commerce). Each data center has its own **data center network** linking its hosts to each other and to the Internet.
- **Cost** (100,000-host data center): over **$12 million/month**. Breakdown: ~**45% hosts** (replaced every 3–4 years), ~**25% infrastructure** (transformers, UPS, generators, cooling), ~**15% electric power**, ~**15% networking** (switches, routers, load balancers, external links, transit). Networking is not the biggest cost, but networking innovation is key to lowering cost and maximizing performance.
- **Hosts** ("blades", like pizza boxes) are commodity machines (CPU, memory, disk) stacked in **racks** of **20–40 blades**. Each rack has a **Top of Rack (TOR) switch** connecting its hosts to each other and to other switches; hosts typically connect to the TOR at **40 Gbps Ethernet**. Each host has its own **data-center-internal IP address**.
- **Two traffic types**: external clients ↔ internal hosts (through **border routers** connecting to the public Internet), and internal host ↔ internal host.

### Load balancing
- Each application has a **publicly visible IP address**. External requests go first to a **load balancer** that distributes requests across hosts according to current load. Large data centers have several, each for specific applications. Sometimes called a **"layer-4 switch"** since it decides on destination **port number (layer 4)** as well as destination IP.
- It relays the host's response back to the client and also does a **NAT-like translation** of public ↔ internal addresses, which **hides the internal structure** and prevents clients from contacting hosts directly (a security benefit).

### Hierarchical architecture (Figure 6.30)
- Small data center (few thousand hosts): border router + load balancer + one Ethernet switch could suffice. To scale to hundreds of thousands of hosts: a **hierarchy** — border router → **access routers** → **tier-1 switches** (connect to multiple tier-2 switches and a load balancer) → **tier-2 switches** → **TOR (tier-3) switches** → racks. All links use **Ethernet** (copper and fiber).
- **Redundancy** for high availability (e.g., each TOR connects to two tier-2 switches; access routers and tier-1/2 switches duplicated).
- The hosts under each access router form a **single subnet**; to localize **ARP broadcast traffic**, each subnet is further partitioned into smaller **VLAN subnets** of a few hundred hosts.
- **Limitation — limited host-to-host capacity**: with 1 Gbps host links and 10 Gbps inter-switch links, two hosts in the **same rack** get a full 1 Gbps, but with many simultaneous cross-rack flows rates collapse. Example: **40 simultaneous flows** (10 each between racks 1↔5, 2↔6, 3↔7, 4↔8) share the 10 Gbps A–B and B–C links, so each gets **10 Gbps / 40 = 250 Mbps**, far below the 1 Gbps NIC rate; it gets worse higher in the hierarchy. Buying faster switches/routers is very expensive.
- Why it matters: data centers need **flexibility in placing computation and services** (e.g., a search engine over thousands of hosts across racks; EC2 placing VMs on whichever hosts have capacity).

### Trends in data center networking
- **Fully connected / highly interconnected topologies** (Figure 6.31): every tier-1 switch connects to **every** tier-2 switch, so host-to-host traffic never needs to go above the switch tiers, and with n tier-1 switches there are **n disjoint paths** between any two tier-2 switches. For the 40-flow example: **4 paths → 40 Gbps** aggregate capacity. Any two racks not on the same switch are logically equivalent regardless of location.
- **Shipping-container modular data centers (MDCs)**: a factory builds a mini data center in a standard **12-meter shipping container** (up to a few thousand hosts, tens of racks), ships it, and containers are interconnected on site. Hard to service, so containers are designed for **graceful degradation** and are replaced when performance drops below a threshold. Two networks: **container-internal** (can be fully connected using cheap commodity Gigabit Ethernet switches) and the **core network** connecting containers (hard to design for high cross-container bandwidth; a hybrid electrical/optical switch architecture has been proposed).
- **Routing in highly interconnected topologies**: possibilities include **random routing**, or multiple NICs per host connected to multiple low-cost switches with **hosts themselves routing intelligently** among them.
- Providers increasingly **build or customize nearly everything** (NICs, switches, routers, TORs, software, protocols).
- **Availability zones** (pioneered by Amazon): replicate distinct data centers in nearby buildings (a few km apart) so transactional data can be synchronized for fault tolerance.

---

## 6.7 Retrospective: A Day in the Life of a Web Page Request

Scenario: **Bob** plugs his laptop into the school Ethernet switch (connected to the school router, which connects to ISP **comcast.net**; Comcast provides DNS, and the DHCP server runs in the router) and loads **www.google.com**. Steps:

### 6.7.1 Getting started: DHCP, UDP, IP, Ethernet
1. Bob's OS creates a **DHCP request** inside a **UDP segment** (dest port **67**, source port **68**), inside an **IP datagram** with dest **255.255.255.255** and source **0.0.0.0** (no address yet).
2. The datagram goes inside an **Ethernet frame** with dest MAC **FF:FF:FF:FF:FF:FF** and Bob's source MAC (00:16:D3:23:68:8A).
3. The switch **broadcasts** it on all ports, including the router's.
4. The router (MAC 00:22:6B:45:1F:1B) receives it; the broadcast IP destination means it is processed by upper layers → UDP → DHCP request extracted.
5. The router's DHCP server allocates **68.85.2.101** from block **68.85.2.0/24** and builds a **DHCP ACK** with: the IP address, the **DNS server** (68.87.71.226), the **default gateway** (68.85.2.1), and the subnet block/network mask. It is sent in UDP → IP → Ethernet, addressed to Bob's MAC.
6. The frame is **unicast** to the switch, which (being **self-learning** and having already seen Bob's frame) forwards it **only to Bob's port**.
7. Bob's laptop extracts the ACK, records its IP address and DNS server, and installs the default gateway in its forwarding table. (Only the **last two** of the four DHCP steps are really needed here.)

### 6.7.2 Still getting started: DNS and ARP
8. Typing www.google.com → needs the IP → a **DNS query** in **UDP (dest port 53)**, in an IP datagram to **68.87.71.226** from 68.85.2.101.
9. To frame it, Bob needs the **gateway's MAC** (he knows its IP, 68.85.2.1, from DHCP) → **ARP**.
10. ARP query for 68.85.2.1 in a **broadcast frame** (FF:FF:FF:FF:FF:FF) → switch delivers to everyone.
11. The gateway router sees its own IP in the query and replies with an **ARP reply** (MAC 00:22:6B:45:1F:1B), unicast to Bob.
12. Bob extracts the gateway MAC.
13. Bob sends the DNS-query frame addressed to the **gateway's MAC**, even though the **IP destination is the DNS server** (68.87.71.226) — the classic MAC-per-hop vs. IP-end-to-end distinction.

### 6.7.3 Intra-domain routing to the DNS server
14. The gateway router looks up 68.87.71.226 in its forwarding table and sends the datagram toward the **leftmost Comcast router** in a link-layer frame suited to that link.
15. Comcast routers forward using tables filled by the **intra-domain protocol (RIP, OSPF, or IS-IS)** and the **inter-domain protocol (BGP)**.
16. The DNS server finds www.google.com's record (**64.233.169.105**, assumed cached; originally from google.com's authoritative server) and returns a **DNS reply** (UDP → IP → ... → Bob).
17. Bob's laptop extracts the IP address of www.google.com.

### 6.7.4 Web client-server interaction: TCP and HTTP
18. Bob's browser creates a **TCP socket**; TCP does the **three-way handshake**: SYN segment to **port 80** in a datagram to 64.233.169.105, in a frame addressed to the gateway MAC.
19. Routers in the school, Comcast, and Google networks forward it using their forwarding tables (the Comcast–Google inter-domain entries come from **BGP**).
20. The SYN arrives at Google, is demultiplexed to the **welcome socket on port 80**, a **connection socket** is created, and a **SYNACK** is sent back.
21. The SYNACK reaches Bob's laptop; the socket becomes **connected**.
22. The browser writes the **HTTP GET** into the socket; it travels as a TCP segment in a datagram to Google.
23. Google's HTTP server reads the GET, builds an **HTTP response** with the page content, and writes it into the socket.
24. The response datagram returns through Google, Comcast, and school networks; the browser reads it and **displays the page**.
- Omitted for simplicity: NAT in the school's gateway, wireless access, security protocols, network management, Web caching, the DNS hierarchy.

---

## 6.8 Summary (Chapter Recap)

- The basic link-layer service is to move a datagram from one node to an **adjacent** node; all link-layer protocols **encapsulate datagrams in frames**, but differ in access, delivery, and transmission services because links differ (point-to-point, multiple access, and even **a whole network acting as a link**, as with MPLS or a telephone connection).
- Principles covered: **error detection/correction** (parity, checksum, CRC), **multiple access protocols** (partitioning, random access, taking turns), **link-layer addressing** (MAC, ARP), **virtualization (VLANs)**, extended **switched LANs**, and **data center networks**.
- Next: **Chapter 7 — Wireless and Mobile Networks**.

---

## Key Formulas and Facts Quick Reference
- **CRC**: R = remainder of (D·2^r) / G (modulo-2); receiver divides by G, nonzero remainder = error. Worked example: D = 101110, G = 1001, R = 011.
- **Two-dimensional parity**: i+j+1 parity bits; corrects 1 error, detects 2.
- **Slotted ALOHA** efficiency: N·p·(1−p)^(N−1) → max **1/e ≈ 0.37**. **Pure ALOHA**: p(1−p)^(2(N−1)) per node → max **1/(2e) ≈ 0.18**.
- **CSMA/CD efficiency** ≈ $1/(1 + 5\,d_{prop}/d_{trans})$
- **Binary exponential backoff**: after n collisions, K ∈ {0,…,2^n − 1}; Ethernet waits K × **512 bit times**; n capped at **10**
- **MAC address**: 6 bytes (48 bits); broadcast = **FF-FF-FF-FF-FF-FF**; IEEE sells blocks of **2^24**
- **ARP**: resolves IP → MAC **only within a subnet**; query broadcast, reply unicast; entry TTL typically **20 min**
- **Ethernet frame**: preamble 8 B, dest 6 B, source 6 B, type 2 B, data **46–1500 B**, CRC 4 B; ARP type = **0806**
- **Switch**: self-learning (source MAC + interface + time), aging time; unknown destination → **broadcast**; same interface → **filter**; else **forward**
- **802.1Q VLAN tag**: 4 bytes = TPID **81-00** (2 B) + TCI (2 B: **12-bit VLAN ID**, **3-bit priority**)
- **DOCSIS**: downstream 6 MHz (~40 Mbps), upstream up to 6.4 MHz (~30 Mbps); **MAP messages** grant mini-slots; request mini-slots use random access + exponential backoff
- **MPLS**: header between link and IP headers (label, 3 experimental bits, S bit, TTL); enables **traffic engineering, fast restoration, VPNs**
- **Data center cross-rack example**: 40 flows over a 10 Gbps link → **250 Mbps** each
