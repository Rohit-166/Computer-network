# Chapter 4 — The Network Layer: Data Plane
### (Kurose & Ross, *Computer Networking: A Top-Down Approach*, 7th Ed.)

---

## Introduction

- Unlike transport/application layers (which run only in end systems), the network layer has **a piece running in every host AND every router** — making it among the most challenging layers in the stack.
- The network layer is split into **two interacting parts**:
  - **Data plane**: per-router **local** functions determining how a datagram arriving on one input link is forwarded to an output link — covered in **Chapter 4**.
  - **Control plane**: **network-wide** logic controlling how datagrams are routed among routers along an end-to-end path — covered in **Chapter 5** (routing algorithms, OSPF, BGP, SDN controllers).
- Traditionally, control-plane routing and data-plane forwarding were implemented **together, monolithically**, within each router. **SDN (Software-Defined Networking)** explicitly **separates** them, implementing control-plane functions in a **remote controller**.

---

## 4.1 Overview of the Network Layer

- Network layer at the **sending host**: takes transport-layer segments, encapsulates each into a **datagram**, sends to its nearby router. At the **receiving host**: receives datagrams, extracts segments, delivers up to transport layer.
- **Routers** have a **truncated protocol stack** — no layers above the network layer, since routers don't run application/transport-layer protocols.

### 4.1.1 Forwarding and Routing: The Data and Control Planes
Two key network-layer functions:
- **Forwarding**: the **router-local** action of moving a packet from an input link interface to the appropriate output link interface. Happens at very **short timescales** (nanoseconds) — typically implemented in **hardware**. This is a data-plane function. (More generally, forwarding could also involve **blocking** a packet — e.g., from a known malicious source — or **duplicating** it across multiple outgoing links.)
- **Routing**: the **network-wide process** that determines the end-to-end paths packets take from source to destination, computed by **routing algorithms**. Happens at much **longer timescales** (seconds) — typically implemented in **software**. This is a control-plane function.
- **Driving analogy**: forwarding = the process of getting through a single interchange (car enters, determines which road to take to leave); routing = planning the entire trip (consulting a map beforehand, choosing among many possible paths).
- **Forwarding table**: every router has one — the router examines value(s) of one or more header fields in an arriving packet, indexes into the forwarding table using these values, and the table entry tells the router which outgoing link interface to use.

**Control Plane — The Traditional Approach:**
- A **routing algorithm runs in every router**; routing components in different routers communicate with each other (exchanging **routing messages** per a **routing protocol**) to collectively compute each router's forwarding table entries.
- Illustrative thought experiment: if forwarding tables were configured **manually** by human operators, **no routing protocol would be needed at all** — but this would be error-prone and far slower to adapt to topology changes than an automated protocol.

**Control Plane — The SDN Approach:**
- A **physically separate, remote controller** computes and distributes forwarding tables to every router; the router itself now performs **forwarding only**. Routers and the remote controller communicate by exchanging messages carrying forwarding-table/routing information.
- The remote controller might run in a data center with high reliability/redundancy, managed by the ISP or a third party. Increasingly implemented in **open-source software** (like Linux), letting ISPs/researchers/students innovate on network-layer control logic.

### 4.1.2 Network Service Model
- Defines **the characteristics of end-to-end delivery** of packets between sending and receiving hosts. Candidate services a network layer *could* provide: **guaranteed delivery**; **guaranteed delivery with bounded delay**; **in-order delivery**; **guaranteed minimal bandwidth** (emulating a fixed-bit-rate link); **security** (encrypt at source, decrypt at destination).
- **The Internet's actual network layer provides only ONE service: best-effort service.**
  - No guarantee of in-order delivery, no guarantee of eventual delivery at all, no delay bound, no bandwidth guarantee. (A network delivering **zero** packets would technically satisfy this definition!)
  - Other architectures do more: **ATM** provides guaranteed in-order delivery, bounded delay, and guaranteed minimal bandwidth. **Intserv** (RFC 1633) was a proposed Internet extension aiming for end-to-end delay guarantees and congestion-free communication.
  - Despite this minimalism, the Internet's best-effort model **combined with adequate bandwidth provisioning** has proven "more than good enough" to support Netflix, Skype, FaceTime, etc.
- **Terminology clarifications**: "forwarding" and "switching" are used interchangeably in this book. A **packet switch** = general term for any device transferring a packet from input to output interface based on header values. A **link-layer switch** (Ch. 6) bases decisions on **link-layer frame** fields (a layer-2 device). A **router** bases decisions on **network-layer datagram** fields (a layer-3 device).

---

## 4.2 What's Inside a Router?

Four components of a **generic router architecture**:
1. **Input ports**: terminate the incoming physical link (physical layer), perform link-layer functions to interoperate with the other end of the link, and perform the crucial **lookup function** (consulting the forwarding table to determine the output port). Control packets (e.g., routing-protocol packets) are forwarded to the **routing processor**. "Port" here means the **physical** interface — distinct from the software/socket "port" of Ch. 2–3. Real routers can have hundreds of 10 Gbps ports (e.g., Juniper MX2020: up to 960 10 Gbps ports, 80 Tbps overall capacity).
2. **Switching fabric**: connects input ports to output ports — "a network inside the network router."
3. **Output ports**: store packets received from the switching fabric and transmit them onto the outgoing link (performs link-layer and physical-layer functions). For a bidirectional link, an output port is typically paired with the input port for that same link, on the same **line card**.
4. **Routing processor**: performs **control-plane** functions. In traditional routers: executes routing protocols, maintains routing tables/link-state info, computes the forwarding table. In SDN routers: communicates with the remote controller to receive and install forwarding-table entries. Also performs network-management functions (§5.7).

- Input ports, output ports, and the switching fabric are **almost always implemented in hardware** — necessary because at 10 Gbps with 64-byte datagrams, an input port has only **51.2 ns** to process each datagram (far too fast for software). Control-plane functions operate at millisecond/second timescales → implemented in **software** on the routing processor (a traditional CPU).
- **Roundabout analogy**: entry road + entry station (with an attendant who looks up destination) = input port (with lookup); the roundabout itself = switching fabric; exit road = output port. **Destination-based forwarding** = attendant only considers final destination. **Generalized forwarding** = attendant could route based on many other factors (e.g., the car's state of origin/license plate) — foreshadowing §4.4.

### 4.2.1 Input Port Processing and Destination-Based Forwarding
- A brute-force forwarding table with **one entry per possible 32-bit destination address** (4+ billion) is completely infeasible.
- Solution: forwarding tables use **address-range/prefix entries**. The router matches the **prefix** of the packet's destination address against table entries.
- **Longest prefix matching rule**: when a destination address matches **multiple** entries (which is possible, since prefixes can overlap), the router selects the entry with the **longest matching prefix** and forwards accordingly. (Explained further with CIDR in §4.3.3.)
- Lookup must happen in **hardware**, using techniques faster than simple linear search — special attention to memory access (on-chip DRAM, SRAM caches). **TCAM (Ternary Content Addressable Memory)** is commonly used: presenting a 32-bit IP address returns the matching forwarding-table entry's content in essentially **constant time**. (Cisco Catalyst 6500/7600: can hold upwards of a million TCAM entries.)
- Once the output port is determined, the packet is sent into the switching fabric — but may be **temporarily blocked** (queued) if the fabric is currently busy with packets from other input ports.
- Other input-port tasks besides lookup: (1) physical/link-layer processing, (2) checking + rewriting the datagram's **version number, checksum, and TTL** fields, (3) updating network-management counters.
- **"Match plus action"**: the input-port process of looking up a destination address ("match") then sending the packet toward the specified output port ("action") is a special case of a more general abstraction used across many networked devices — link-layer switches (Ch. 6), firewalls (Ch. 8, match on header criteria → drop), NAT (rewrite port number → forward). This general abstraction underlies **generalized forwarding** (§4.4).

### 4.2.2 Switching
Three main switching-fabric technologies:

| Technique | Mechanism | Throughput Characteristics |
|---|---|---|
| **Switching via memory** | Earliest/simplest routers — traditional computers; CPU directly controls the copy from input port memory to output port buffer, under an interrupt-driven scheme. | Forwarding throughput must be **< B/2** where B = memory read/write bandwidth (since each packet requires one read + one write via the shared system bus); only **one** packet can be forwarded at a time (shared bus). Some modern routers still switch via memory, but lookup/storing is now done by input line-card processing rather than a central CPU (Cisco Catalyst 8500 series). |
| **Switching via a bus** | Input port pre-pends a switch-internal label indicating the target output port, then transmits onto a **shared bus**; all output ports see it, only the matching one keeps it (label removed after). | Only **one packet at a time** can cross the bus — switching speed limited to bus speed. Sufficient for small LAN/enterprise routers (Cisco 6500: 32-Gbps bus). |
| **Switching via an interconnection network** | E.g., a **crossbar switch**: 2N buses connecting N inputs to N outputs, with a **crosspoint** at each intersection that can be opened/closed by the fabric controller. | **Non-blocking**: multiple packets CAN be forwarded **in parallel**, as long as they're headed to **different** output ports (e.g., A→Y and B→X simultaneously use different bus pairs). If two packets from different inputs target the **same** output, one must wait. Cisco 12000 series uses crossbar; Cisco 7600 configurable as bus or crossbar. More sophisticated multi-stage interconnection networks (e.g., Cisco CRS's 3-stage design) allow even more parallelism; capacity can also be scaled by running **multiple fabrics in parallel**, with an input port "spraying" chunks of a packet across several fabrics for reassembly at the output. |

### 4.2.3 Output Port Processing
- Takes packets stored in the output port's memory and transmits them over the outgoing link — includes **selecting and de-queueing** packets for transmission, plus link-layer/physical-layer transmission functions.

### 4.2.4 Where Does Queuing Occur?
Both input AND output ports can develop queues — this is where packets are actually **dropped/lost** within a router.

- Setup: N input ports, N output ports, all lines at rate R_line; switching fabric transfer rate R_switch. If **R_switch ≥ N × R_line**, input-port queuing is **negligible** (each batch of N arriving packets clears the fabric before the next batch arrives).

**Input Queuing & HOL Blocking:**
- If the fabric is slower than N×R_line, packets must queue at input ports.
- With a crossbar fabric transferring only **one packet per output port at a time**, if two packets at the **front** of two different input queues are both headed to the **same** output port, one must wait — and (**crucially**) so must every packet queued **behind** it in that input queue, even ones destined for currently-free output ports.
- This is **Head-of-the-Line (HOL) blocking**. [Karol 1987] shows that HOL blocking can cause the input queue to grow **unboundedly** once the input arrival rate reaches just **58%** of capacity, under certain assumptions.

**Output Queuing:**
- Even with R_switch = N × R_line, if all N input ports simultaneously receive packets destined for the **same** output port, N packets arrive at that output port in the time it takes to transmit just **one** packet onward — so a queue forms regardless.
- When output memory is exhausted: **drop-tail** policy (drop the arriving packet) or drop already-queued packets to make room. Proactive dropping/marking before the buffer is completely full, to signal congestion early, falls under **AQM (Active Queue Management)** — the most widely studied AQM algorithm being **RED (Random Early Detection)**.
- A **packet scheduler** at the output port must choose, among queued packets, which to transmit next (§4.2.5).
- **Buffer sizing rule of thumb** (RFC 3439, older): buffering **B = RTT × C** (e.g., 10 Gbps link, 250ms RTT → B = 2.5 Gbits). **More recent research** [Appenzeller 2004] shows that with a **large number N of TCP flows** sharing the link, the needed buffering is actually **B = (RTT × C) / sqrt(N)** — substantially smaller in high-flow-count backbone scenarios.

### 4.2.5 Packet Scheduling
Determines the **order** in which queued packets are transmitted over an outgoing link.

**FIFO (First-In-First-Out / FCFS):** transmits packets in the exact order they arrived. Simplest; when the queue is full, incoming packets are either dropped or displace already-queued ones (discard policy).

**Priority Queuing:** packets classified into priority classes on arrival (e.g., by port number — network-management traffic or VoIP might get priority over SMTP/e-mail); each class typically has its own queue. Router always transmits from the **highest-priority non-empty queue**; within a class, FIFO. Illustrated as **non-preemptive**: a packet already being transmitted is never interrupted, even if a higher-priority packet arrives mid-transmission — it must wait.

**Round Robin (RR):** packets sorted into classes as in priority queuing, but service **alternates** among classes (class 1, then class 2, then class 1, ...) rather than strict priority. **Work-conserving**: if a class's queue is empty when its turn comes, the scheduler immediately moves to the next class rather than leaving the link idle.

**Weighted Fair Queuing (WFQ):** a generalized form of RR. Each class *i* is assigned a **weight w_i**. During any interval where class *i* has packets queued, it is **guaranteed** to receive a fraction of service equal to:
$$\frac{w_i}{\sum_j w_j}$$
(sum taken over all classes that currently have queued packets — or, in the worst case, over ALL classes). So for a link of rate R, class *i* is always guaranteed throughput of at least R times w_i / (sum of w_j). (Idealized description — doesn't address the fact that real packets are discrete/non-preemptible.)

- **Which disciplines preserve arrival order?** Only **FIFO** guarantees packets depart in the same order they arrived; priority/RR/WFQ reorder packets across classes.

---

## 4.3 The Internet Protocol (IP): IPv4, Addressing, IPv6, and More

Two IP versions in active use today: **IPv4** (RFC 791) and **IPv6** (RFC 2460, RFC 4291). "To master IP addressing is to master the Internet's network layer itself!"

### 4.3.1 IPv4 Datagram Format

| Field | Size | Purpose |
|---|---|---|
| **Version** | 4 bits | IP version (4 for IPv4) — tells the router how to interpret the rest of the datagram |
| **Header length** | 4 bits | Where the payload begins (datagrams can have a variable-length options field); typical header = **20 bytes** (no options) |
| **Type of service (TOS)** | 8 bits | Distinguishes datagram types (e.g., real-time vs. non-real-time traffic); 2 of these bits are used for **ECN** (Section 3.7.2) |
| **Datagram length** | 16 bits | Total length (header + data) in bytes; theoretical max = 65,535 bytes, though datagrams rarely exceed **1,500 bytes** (fits a max-size Ethernet frame payload) |
| **Identifier, flags, fragmentation offset** | — | Used for **IP fragmentation** (Section 4.3.2) — NOT present in IPv6 |
| **Time-to-live (TTL)** | 8 bits | Decremented by 1 at each router; datagram dropped when it hits 0 — prevents datagrams circulating forever (e.g., due to routing loops) |
| **Protocol** | 8 bits | Identifies the transport protocol to hand the payload to at the final destination (6 = TCP, 17 = UDP) — analogous to how a port number binds transport-application; the protocol field binds network-transport |
| **Header checksum** | 16 bits | Internet checksum (1's complement, as in Section 3.3) computed over the **header only**; must be **recomputed at every router** since TTL (and possibly options) change en route |
| **Source / destination IP address** | 32 bits each | — |
| **Options** | variable | Rarely used; extends the header but complicates fixed-offset parsing — **removed entirely in IPv6** |
| **Data (payload)** | variable | Typically a TCP or UDP segment; can also carry other data e.g. ICMP messages |

- With no options: 20-byte IP header. A datagram carrying a TCP segment (no fragmentation) -> **40 bytes total header** (20 IP + 20 TCP).
- **Why checksum at both transport AND network layers?** (1) Only the IP **header** is checksummed at the network layer, vs. the **entire segment** at the transport layer. (2) TCP/UDP need not always run over IP (e.g., TCP could in principle run over ATM), and IP can carry non-TCP/UDP payloads.

### 4.3.2 IPv4 Datagram Fragmentation
- Different link-layer protocols support different **MTUs** (Maximum Transmission Units — e.g., Ethernet: 1,500 bytes; some WAN links: as low as 576 bytes). Since each datagram is encapsulated in a link-layer frame hop-by-hop, differing MTUs along a path are a real problem.
- **Solution**: when an outgoing link's MTU is smaller than the arriving datagram, the router **fragments** the payload into two or more smaller datagrams (**fragments**), each independently encapsulated and sent onward.
- **Design decision**: IPv4 designers chose to do **reassembly only at the destination end system**, never at intermediate routers — keeping the network core simple (consistent with the general "keep the core simple" Internet design philosophy).
- Mechanism: sending host stamps each datagram with an **identification number** (incremented per datagram); when a router fragments a datagram, **all resulting fragments carry the same identification number** plus the original source/destination addresses. A **flag bit** = 1 on all fragments except the **last** one (flag = 0), letting the destination know when reassembly is complete. An **offset field** specifies where each fragment fits within the original datagram (also lets the receiver detect missing fragments and reorder them).
- **Worked example**: a 4,000-byte datagram (20-byte header + 3,980-byte payload) must go out over a link with MTU = 1,500 bytes -> the 3,980 payload bytes get split across **three fragments**.

### 4.3.3 IPv4 Addressing
- An **interface** = the boundary between a host/router and one of its physical links. IP addresses are technically associated with **interfaces**, not with the host/router itself. A typical host has **one** interface; a router (with multiple links) has **multiple** interfaces, each needing its own IP address.
- IP address = **32 bits (4 bytes)** -> ~4 billion possible addresses. Written in **dotted-decimal notation** (e.g., `193.32.216.9` = `11000001 00100000 11011000 00001001` in binary).
- Every interface on every Internet host/router must have a **globally unique** address (except interfaces behind a NAT).

**Subnets:**
- A **subnet** (a.k.a. "IP network" or just "network") = a set of device interfaces that can physically reach each other **without an intervening router** — typically interconnected by a switch (Ch. 6) or wireless access point (Ch. 7).
- Subnet addressing: `223.1.1.0/24` — the **"/24"** ("slash-24") notation (a.k.a. a **subnet mask**) means the **leftmost 24 bits** define the subnet address; the remaining bits distinguish devices within the subnet.
- **Recipe to identify all subnets in an interconnected system**: "detach each interface from its host or router, creating islands of isolated networks" — each isolated island is a subnet. (Applies both to broadcast/LAN-style subnets connecting hosts to a router, AND to the point-to-point links directly connecting two routers — each such link is itself its own /24 subnet in the book's examples.)

**CIDR (Classless Interdomain Routing)** — RFC 4632, pronounced "cider":
- The Internet's actual address-assignment strategy; generalizes subnet addressing. Address form `a.b.c.d/x`: the leftmost **x** bits = the **network portion / prefix**; the remaining **32-x** bits distinguish devices **within** the organization (and may have further internal subnetting structure).
- Routers **outside** the organization need only consider the leading **x** bits when forwarding to that organization — dramatically **shrinking external forwarding tables**, since one entry (`a.b.c.d/x`) suffices for the whole organization.
- **Before CIDR — classful addressing**: network portion constrained to exactly **8, 16, or 24 bits** -> **Class A / B / C** networks respectively. Problematic: Class C (/24) supports only 254 usable hosts (2^8 - 2, since 2 addresses are reserved) — too small for many orgs; Class B (/16) supports 65,534 — often wastefully large (e.g., an org with 2,000 hosts getting a /16 wastes >63,000 addresses). This drove rapid depletion of the Class B space and motivated CIDR.
- **Address aggregation / route summarization**: an ISP can advertise a **single prefix** (e.g., `200.23.16.0/20`) to the outside world covering **multiple** downstream client organizations, each with their own internal subnet, without the rest of the Internet needing to know those internal details.
  - **Non-hierarchical complication example**: if Organization 1 (originally under Fly-By-Night-ISP, address block `200.23.16.0/20`) connects instead through a **subsidiary ISP** (ISPs-R-Us, block `199.31.0.0/16`) but **keeps its old addresses** (`200.23.18.0/23`, outside ISPs-R-Us's normal block) — ISPs-R-Us must additionally and specifically advertise `200.23.18.0/23`. Routers elsewhere seeing both the broader `200.23.16.0/20` (Fly-By-Night) and the more specific `200.23.18.0/23` (ISPs-R-Us) use **longest prefix matching** to correctly route toward ISPs-R-Us for that sub-block.
- **IP broadcast address**: `255.255.255.255` — delivered to **all hosts on the same subnet**; routers *optionally* (but usually don't) forward it into neighboring subnets.

**Obtaining a Block of Addresses:**
- An organization typically gets a block from its **ISP**, which itself carves out a piece of a **larger block already allocated to the ISP** (e.g., an ISP with `200.23.16.0/20` splitting into eight `/23` blocks for eight client organizations).
- Ultimate authority: **ICANN** (Internet Corporation for Assigned Names and Numbers) — manages the IP address space AND the DNS root servers, and (contentiously) domain-name assignment/dispute resolution. ICANN allocates to **regional Internet registries** (ARIN, RIPE, APNIC, LACNIC — together the ASO-ICANN), which manage addresses within their regions.

**Obtaining a Host Address — DHCP (Dynamic Host Configuration Protocol, RFC 2131):**
- Lets a host **automatically** obtain an IP address (either a fixed address each time, or a temporary one that varies) — a **"plug-and-play" / "zeroconf"** protocol. Widely used in residential/enterprise/WiFi networks where hosts join/leave frequently.
- Besides the address itself, DHCP also lets a host learn its **subnet mask**, the address of its **first-hop router** (the "**default gateway**"), and its **local DNS server's** address.
- **Client-server protocol**. Ideally each subnet has its own DHCP server; if not, a **DHCP relay agent** (typically a router) forwards requests to a known DHCP server elsewhere.
- **Four-step process** for a newly arriving host (all messages use **UDP, port 67**; `yiaddr` = "your Internet address," the address being allocated):
  1. **DHCP discover**: client broadcasts (destination IP `255.255.255.255`, source "this host" `0.0.0.0`) to find any available DHCP server.
  2. **DHCP offer**: any server(s) receiving the discover message broadcast back an offer (also to `255.255.255.255`, since the client has no address yet) — including the discover message's transaction ID, a proposed IP address, subnet mask, and an **IP address lease time**. Client may receive multiple offers to choose from.
  3. **DHCP request**: client chooses one offer, replies echoing back the chosen configuration parameters.
  4. **DHCP ACK**: the chosen server confirms the requested parameters.
- Once ACKed, the client can use the address for the **lease duration**; DHCP also supports **renewing** a lease.
- **Mobility shortcoming**: since DHCP assigns a **new** address on each new subnet, an ongoing TCP connection **cannot survive** a mobile host moving between subnets — motivating **Mobile IP** (covered in Ch. 6).

### 4.3.4 Network Address Translation (NAT)
- Problem: rather than requiring an ISP to allocate a full range of public addresses to every small office/home network device, **NAT** lets an entire home/office network share **one single public IP address** as seen from the outside world.
- **Private address space** (RFC 1918): three reserved blocks with meaning **only within** a private network — the book's example uses **`10.0.0.0/8`**. Hundreds of thousands of separate home networks can all reuse the same `10.0.0.0/24` range internally without conflict, since these addresses never appear as source/destination on the public Internet.
- The NAT-enabled router presents **a single IP address** to the outside world (e.g., `138.76.29.7`); all outbound traffic gets this source address, all inbound traffic must target this destination address. (The router itself often gets *its* single public address via DHCP from the ISP, while simultaneously running its **own DHCP server** to assign private addresses within the home network.)
- **NAT translation table**: since all inbound datagrams share the same destination IP, the router uses **port numbers** to distinguish which internal host a datagram belongs to.
  - **Worked example**: host `10.0.0.1` (source port 3345) requests a page from `128.119.40.186:80`. The NAT router **rewrites** the source IP to its own public address `138.76.29.7` and generates a **new source port** (e.g., 5001), recording `(10.0.0.1, 3345) <-> (138.76.29.7, 5001)` in its translation table. The remote server, unaware of any translation, replies to `138.76.29.7:5001`; the NAT router looks up the table, **rewrites the destination back** to `10.0.0.1:3345`, and forwards it inward.
  - Since the port-number field is 16 bits, a NAT router can theoretically support **over 60,000 simultaneous connections** with just one public IP.
- **Criticisms of NAT**:
  1. **Port numbers are meant to address processes, not hosts** — this violation causes real problems for servers running on the home network (a well-known port needs to be reachable) and for P2P peers needing to accept incoming connections. Mitigations: **NAT traversal tools** (RFC 5389) and **UPnP** (Universal Plug and Play — lets a host discover/configure a nearby NAT).
  2. **Architectural-purist objection**: routers are meant to be pure layer-3 devices, and NAT violates the principle that hosts should communicate directly without interfering intermediate nodes modifying addresses/ports. Nonetheless, NAT has become firmly entrenched, alongside other **middleboxes** (load balancers, firewalls, etc.) that perform non-traditional network-layer functions — a trend that **generalized forwarding** (Section 4.4) aims to unify.

**Focus on Security — Firewalls and Intrusion Detection Systems (IDS):**
- **Firewalls**: inspect datagram/segment header fields, denying entry to suspicious datagrams — e.g., blocking all ICMP echo-request packets to prevent port-scan-style reconnaissance; can filter by source/destination IP and port, or track ongoing TCP connections and only admit datagrams belonging to approved connections.
- **IDS (Intrusion Detection System)**: performs **deep packet inspection** — examining payloads (including application-layer data), not just headers — matching against a continuously-updated **database of known attack signatures**; raises an alert on a match.
- **IPS (Intrusion Prevention System)**: like an IDS, but **actively blocks** matching packets rather than just alerting.
- Neither fully shields a network — attackers continually devise new attacks with no existing signature — but both are valuable against **known** attacks.

### 4.3.5 IPv6
Developed by the IETF starting in the early 1990s, primarily to address the impending **exhaustion of the 32-bit IPv4 address space**. (IANA allocated its last unassigned IPv4 address pool to a regional registry in **February 2011**.) The designers also used the opportunity to refine other aspects of IPv4 based on accumulated operational experience. ("What happened to IPv5?" — originally envisioned for the ST-2 protocol, which was later dropped.)

**IPv6 Datagram Format — Key Changes:**
- **Expanded addressing**: address size increased from **32 bits to 128 bits** — "every grain of sand on the planet can be IP-addressable." Also introduces a new address type, **anycast** — delivers a datagram to **any one** of a group of hosts (e.g., the nearest of several mirror sites).
- **Streamlined, fixed 40-byte header**: several IPv4 fields dropped or made optional -> faster router processing.
- **Flow labeling**: RFC 2460 describes this as allowing "labeling of packets belonging to particular flows for which the sender requests special handling" (e.g., non-default QoS or real-time service) — audio/video might be a flow; e-mail/file-transfer typically would not.

**IPv6 header fields**: **Version** (4 bits, =6), **Traffic class** (8 bits — like IPv4's TOS), **Flow label** (20 bits), **Payload length** (16 bits — bytes following the 40-byte fixed header), **Next header** (identifies the encapsulated protocol — same value space as IPv4's Protocol field), **Hop limit** (decremented per hop like TTL; datagram discarded at 0), **Source/destination address** (128 bits each, formats detailed in RFC 4291), **Data**.

**Fields REMOVED relative to IPv4:**
- **Fragmentation/reassembly fields**: IPv6 does **not** support fragmentation/reassembly at intermediate routers — only at source/destination. If a router receives a too-large IPv6 datagram for its outgoing link, it simply **drops it** and returns a **"Packet Too Big" ICMP** error to the sender, who must resend at a smaller size. This significantly speeds up router forwarding.
- **Header checksum**: removed since transport-layer (TCP/UDP) and link-layer (e.g., Ethernet) protocols already checksum; keeping a network-layer checksum too was seen as redundant, especially since (as in IPv4) it would need **recomputation at every hop** due to the changing hop-limit field.
- **Options**: no longer part of the standard header — instead handled as one of the possible "next header" types chained after the main header (same mechanism used for the transport-layer protocol itself). Removing it yields the fixed 40-byte header.

**Transitioning from IPv4 to IPv6:**
- A **"flag day"** (a single moment when every device worldwide switches over) is completely infeasible at Internet scale — even the much smaller 1980s NCP->TCP transition needed care, and today's transition involves billions of devices.
- **Tunneling** is the most widely adopted practical approach: when two IPv6 nodes need to communicate but are connected via intervening **IPv4** routers (the "**tunnel**"), the sending-side IPv6 node encapsulates the entire IPv6 datagram inside the **data/payload field of an IPv4 datagram**, addressed to the IPv6 node at the tunnel's far end. The intervening IPv4 routers route this as an ordinary IPv4 datagram, oblivious to its IPv6 payload. At the tunnel's far end, the receiving IPv6 node recognizes the IPv4 datagram carries an IPv6 payload (via **protocol field value 41**, RFC 4213), extracts the original IPv6 datagram, and continues routing it as IPv6.
- **Adoption trends (as of the book's writing)**: >1/3 of US government second-level domains were IPv6-enabled (NIST); only ~8% of clients accessed Google services via IPv6 (Google, 2015), though adoption was reported as accelerating; Europe's 3GPP had specified IPv6 as the standard addressing scheme for mobile multimedia.
- **Broader lesson**: changing a **network-layer** protocol is enormously difficult ("like replacing the foundation of a house") compared to introducing new **application-layer** protocols ("like adding a new coat of paint") — explaining why the Web, IM, streaming, and social media all deployed rapidly while IPv6, multicast, and resource-reservation protocols have seen much slower uptake.

---

## 4.4 Generalized Forwarding and SDN

- Motivation: the proliferation of **middleboxes** — NAT boxes (rewrite IP/port), firewalls (block/redirect via header inspection or deep packet inspection), load balancers (distribute requests across servers) — each historically with **its own specialized hardware/software/management interface**, creating operational headaches. SDN offers a **unified** approach.
- **Generalized "match-plus-action" paradigm**: extends destination-based forwarding's simple "match on destination IP -> forward" into matching on **multiple header fields across multiple protocol layers**, with actions including: forward to one/multiple ports, load-balance across interfaces, **rewrite** header values (NAT-like), **block/drop** (firewall-like), or **redirect to a special server** (DPI-like).
- Devices doing this generalized matching are more accurately called **packet switches** (not strictly "routers" or "switches" — the terminology adopted going forward in SDN literature and in Chapter 5).
- The match-plus-action table (called a **flow table** in **OpenFlow**, the standard this section is based on) is computed, installed, and updated by a **remote controller**.
- Each flow-table entry contains: (1) header-field values to match, (2) a set of **counters** (updated on each match — e.g., packet count, time since last update), (3) a set of **actions** to take on a match.

### 4.4.1 Match
- OpenFlow 1.0 allows matching on **eleven packet-header fields plus the incoming port ID**, spanning **three protocol layers**: **link layer (layer 2)** — source/destination MAC address, Ethernet type, VLAN fields; **network layer (layer 3)** — IP source/destination address, IP protocol, IP type-of-service; **transport layer (layer 4)** — source/destination port number. (This deliberately "defies" strict layering, per Section 1.5.) A switch matching on Ethernet addresses can act as a **layer-2 switch**; matching on IP addresses lets the same device act as a **layer-3 router**.
- Later OpenFlow specifications expanded this to **41 matchable values**.
- **Wildcards** are supported: e.g., `128.119.*.*` matches any address sharing the first 16 bits. Entries also have **priority** — if a packet matches multiple entries, the **highest-priority** match wins.
- **Not all fields can be matched** — e.g., TTL and datagram-length are excluded, reflecting a deliberate functionality-vs-complexity tradeoff. Quoted design principle from Butler Lampson: *"Do one thing at a time, and do it well. An interface should capture the minimum essentials of an abstraction. Don't generalize; generalizations are generally wrong."*

### 4.4.2 Action
Each flow-table entry has a **list of zero-or-more actions**, executed in the listed order if there are multiple:
- **Forwarding**: send out a specific port, broadcast over all ports (except the arrival port), multicast to a selected set, or send to the **remote controller** for further processing (the controller may install new flow-table rules and/or return the packet for forwarding under the updated rules).
- **Dropping**: a flow entry with **no action** means drop any matched packet.
- **Modify-field**: rewrite up to **ten** of the matchable header fields (all layer 2/3/4 fields except the IP Protocol field itself) before forwarding.

### 4.4.3 OpenFlow Examples of Match-Plus-Action in Action
Illustrated on a sample network of 6 hosts (h1-h6) and 3 packet switches (s1, s2, s3), each with 4 interfaces:

- **Example 1 — Simple forwarding**: route packets from h5/h6 (destined to h3/h4) via s3->s1->s2, deliberately avoiding the direct s3-s2 link — accomplished purely with per-switch match (ingress port + IP src/dst prefix) -> forward(specific-port) entries at s1, s2, and s3.
- **Example 2 — Load balancing**: traffic from **h3** destined to `10.1.*.*` is forwarded over one path (direct s2-s1 link), while traffic from **h4** to the same destination range is forwarded over a **different** path (via s2-s3-s1) — a behavior **impossible to achieve** with plain destination-based IP forwarding, since it discriminates by **source** host, not just destination.
- **Example 3 — Firewalling**: s2's flow table is configured to accept traffic **only** from hosts attached to s3 (matching `IP Src = 10.3.*.*`) destined to specific local hosts — any other source is implicitly dropped since it matches no entry.
- The chapter notes that flow tables can even implement **virtual networks** — multiple logically-separate networks, each with independent forwarding behavior, sharing the **same physical** switches/links (explored further in homework problems and revisited with SDN controllers in Section 5.5).

---

## 4.5 Summary

- Covered the **data-plane functions** of the network layer — the per-router mechanisms determining how a datagram arriving on one input link gets forwarded to an output link.
- Studied **router internals** in detail: input/output port functionality, destination-based forwarding, the router's internal switching fabric (memory/bus/interconnection-network), packet queuing (where and why it occurs, including **HOL blocking**), and **packet scheduling** disciplines (FIFO, priority, RR, WFQ).
- Covered both **traditional IP forwarding** (destination-address-based) and **generalized forwarding** (matching on many header fields across multiple layers) — the latter's versatility being central to modern SDN data planes.
- Studied **IPv4 and IPv6** in detail, and **Internet addressing** — "much deeper, subtler, and more interesting than we might have expected."
- Next: **Chapter 5**, the network layer's **control plane** — routing algorithms, routing protocols (OSPF, BGP), and SDN controllers.

---

## Interview Sidebar — Vinton G. Cerf
- Co-designer of TCP/IP and the Internet's architecture; worked at UCLA in the late 1960s under **Leonard Kleinrock**, on the ARPAnet's first node (installed at UCLA, **September 1, 1969**), instrumenting performance measurement.
- With **Bob Kahn**, began working on internetworking in **1973**, focused on the central question of making heterogeneous packet networks interoperate **without modifying the networks themselves** — leading to what became TCP/IP.
- Reflects that he and Kahn "doubted [they] had a clear image" of a world with hundreds of millions of interconnected computers.
- On future challenges: mobility support, battery life, access-link capacity, scaling the optical core, and the **IPv4->IPv6 transition** — plus his ongoing work at NASA's Jet Propulsion Laboratory on an **interplanetary extension** of the Internet.
- Received the **US Presidential Medal of Freedom (2005)** and the **US National Medal of Technology (1997)**.

---

## Key Concepts Quick Reference
- **Forwarding** = local/hardware/nanosecond-scale; **Routing** = network-wide/software/second-scale
- **Longest prefix matching**: used whenever a destination address matches multiple forwarding-table entries
- **HOL blocking**: input-queue packets stuck behind a blocked head-of-line packet, even if their own output port is free; grows unbounded past ~58% input load (Karol 1987)
- Buffer sizing: classic rule **B = RTT x C**; refined rule for N large flows: **B = (RTT x C) / sqrt(N)**
- IPv4 header = 20 bytes (no options); IPv6 header = fixed 40 bytes (no options field at all)
- CIDR notation: `a.b.c.d/x` — leftmost x bits = network prefix
- Classful addressing (pre-CIDR): Class A = /8, Class B = /16, Class C = /24
- Private address ranges (RFC 1918) — book's example: **10.0.0.0/8**
- DHCP: UDP port 67; 4-step handshake = **Discover -> Offer -> Request -> ACK**
- NAT: rewrites (source IP, source port) using a translation table; supports up to ~60,000 simultaneous connections per public IP (16-bit port field)
- IPv4 -> IPv6 transition mechanism: **tunneling** (IPv6 datagram encapsulated inside IPv4 payload; protocol field = 41)
- OpenFlow 1.0: matches on 11 header fields + ingress port, across layers 2/3/4; actions = forward / drop / modify-field
