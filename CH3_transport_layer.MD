# Chapter 3 — Transport Layer
### (Kurose & Ross, *Computer Networking: A Top-Down Approach*, 7th Ed.)

---

## 3.1 Introduction and Transport-Layer Services

- The transport layer provides **logical communication** between application **processes** running on different hosts — from the application's perspective, it's as if the hosts were directly connected, even though they may be on opposite sides of the planet.
- Transport-layer protocols are implemented **only in end systems, not in routers**. Sender side: converts app messages into transport-layer **segments** (breaking messages into chunks + adding a transport header). Segment is passed to the network layer, encapsulated in a **datagram**. Routers act only on network-layer fields — they never examine the transport-layer segment.
- The Internet offers **two** transport protocols to applications: **TCP** and **UDP**.

### 3.1.1 Relationship Between Transport and Network Layers
- **Network-layer protocol**: logical communication between **hosts**. **Transport-layer protocol**: logical communication between **processes** — relies on, and extends, network-layer services.
- **Household analogy**: Two houses (East Coast/West Coast), each with a dozen kids (cousins) who write to each other weekly. In each house, one kid (Ann/Bill) collects mail from siblings and hands it to the postal carrier, and distributes incoming mail to siblings.
  - application messages = letters in envelopes; processes = cousins; hosts = houses; **transport-layer protocol = Ann and Bill**; **network-layer protocol = the postal service**.
  - Ann and Bill work only **within their own house** — they have no say in how mail moves between mail centers (i.e., transport-layer protocols live only in end systems, unaware of the network core).
- The services a transport protocol can offer are **constrained by the underlying network layer's service model** — e.g., if IP can't guarantee bandwidth/delay, TCP/UDP can't either.
- **However**, some services CAN be added even when the network layer lacks them — e.g., TCP offers **reliable data transfer** even over an unreliable (best-effort) IP network layer; similarly, encryption can be added at the transport layer even though the network layer offers no confidentiality (Ch. 8).

### 3.1.2 Overview of the Transport Layer in the Internet
- **UDP** (User Datagram Protocol): unreliable, connectionless service.
- **TCP** (Transmission Control Protocol): reliable, connection-oriented service.
- **Terminology note**: this book calls both TCP's and UDP's transport-layer packets "**segments**" (though RFCs sometimes call UDP's packet a "datagram" — a term this book reserves for the **network-layer** packet, to avoid confusion).
- **IP (Internet Protocol)** = the Internet's network-layer protocol. IP's service model is **best-effort delivery**: makes its "best effort" but gives **no guarantees** — no guaranteed delivery, no guaranteed ordering, no guaranteed data integrity → IP is an **unreliable** service. Every host has at least one **IP address**.
- **Multiplexing and demultiplexing**: the most fundamental job of UDP/TCP — extending IP's host-to-host delivery to process-to-process delivery (detailed in §3.2).
- UDP's only two services: process-to-process delivery + light error checking (via a checksum) — otherwise **just as unreliable as IP**.
- TCP additionally provides: **reliable data transfer** (via flow control, sequence numbers, acknowledgments, timers) and **congestion control** — the latter being a service to the **network as a whole**, not just the two endpoints, preventing any one connection from swamping links/routers; TCP tries to give each connection an equal share of link bandwidth. UDP traffic is **unregulated** — an app can send at any rate for as long as it likes.

---

## 3.2 Multiplexing and Demultiplexing

- **Demultiplexing**: delivering data in a received transport-layer segment to the **correct socket**.
- **Multiplexing**: gathering data chunks from different sockets at the source host, encapsulating each with header info, and passing the resulting segments down to the network layer.
- **Household analogy continued**: Bill demultiplexes mail by reading the addressee names and delivering to siblings; Ann multiplexes by collecting mail from siblings for the postal carrier.
- Multiplexing/demultiplexing require: (1) sockets have **unique identifiers**, and (2) each segment carries special fields identifying the target socket — the **source port number** and **destination port number** fields.
  - Each port number is a **16-bit number, 0–65535**. Ports **0–1023** are **well-known port numbers**, reserved for well-known protocols (e.g., HTTP=80, FTP=21) — listed in RFC 1700, updated at iana.org.

### Connectionless Multiplexing and Demultiplexing (UDP)
- `socket(AF_INET, SOCK_DGRAM)` — transport layer auto-assigns a port (range **1024–65535**) unless the app explicitly `bind()`s a specific port (typically done on the **server** side for well-known protocols; client side usually lets the OS auto-assign).
- **A UDP socket is fully identified by a two-tuple: (destination IP address, destination port number).** So two UDP segments with **different source** IP/port but the **same destination** IP/port are delivered to the **same socket**.
- **Source port number's purpose**: serves as part of the "**return address**" — when B replies to A, B's destination port = A's original source port.

### Connection-Oriented Multiplexing and Demultiplexing (TCP)
- **A TCP socket is identified by a four-tuple: (source IP, source port, destination IP, destination port).** Incoming TCP segments are demultiplexed using **all four** values.
- Consequently, unlike UDP, **two arriving TCP segments with different source IP/port** are directed to **different sockets** (except the very first connection-establishment segment).
- **TCP server mechanics**: the server has a **"welcoming socket"** listening for connection requests on a well-known port (e.g., 12000). A client's `connect()` call sends a connection-establishment segment (destination port 12000 + a special connection bit set) with a **client-chosen source port**. The server's OS locates the listening process, which creates a **new socket** via `accept()`; this new socket is identified by the four-tuple (client IP, client port, server IP, server port) — **all subsequent segments matching these four values** demultiplex to this new socket. The server can support **many simultaneous connection sockets**, each with its own four-tuple.
- **Focus on Security — Port Scanning**: Attackers (or sysadmins) use **port scanners** (e.g., **nmap**, free at nmap.org) to discover which ports are open on a target host, mapping open ports to specific (possibly vulnerable) applications — e.g., the Slammer worm exploited a known buffer-overflow flaw in a SQL server listening on UDP port 1434. For TCP, nmap sequentially probes for ports accepting connections; for UDP, it probes for ports responding to UDP segments — either way, returns open/closed/unreachable.
- **Web servers and TCP**: A Web server (e.g., Apache) on port 80 — all segments from all clients arrive with **destination port 80**; the server distinguishes different clients purely via **source IP + source port**. Modern high-performance servers typically use **one process + one thread per connection** (rather than a full new process per connection), each thread with its own connection socket. With **persistent HTTP**, the client and server reuse the same socket throughout; with **non-persistent HTTP**, a new socket is created and closed per request/response — this frequent open/close can hurt performance on busy servers.

---

## 3.3 Connectionless Transport: UDP

- UDP (**RFC 768**) does "just about as little as a transport protocol can do" — only multiplexing/demultiplexing + light error checking, adding almost nothing to IP. With UDP, the app is almost talking directly to IP.
- UDP is **connectionless**: no handshaking between sender and receiver before sending a segment.
- **Example**: DNS typically uses UDP — the DNS app constructs a query, hands it to UDP (no handshake), UDP adds headers and passes to the network layer; if no reply arrives, the querying app can retry, try another server, or give up.

**Why would a developer choose UDP over TCP** (given TCP is "more reliable")?
1. **Finer application-level control over what/when to send**: UDP passes data to the network layer **immediately**, with no congestion-control throttling. TCP delays sending and continues resending until acknowledged, which is poorly matched to real-time apps that tolerate some loss but need a floor on sending rate/latency.
2. **No connection establishment**: TCP uses a **three-way handshake** before transferring data; UDP "just blasts away." This is a principal reason **DNS runs over UDP** (would be much slower over TCP). HTTP uses TCP because reliability matters for text, but TCP's setup delay is a known contributor to Web-download delay — hence **QUIC** (Quick UDP Internet Connection), used by Google's Chrome, builds reliability into an app-layer protocol **on top of UDP** to avoid this.
3. **No connection state**: TCP maintains **connection state** in end systems (buffers, congestion-control parameters, sequence/ack numbers) — needed for reliable transfer + congestion control. UDP maintains **no such state**, so a UDP-based server can typically support **many more active clients** than an equivalent TCP-based server.
4. **Small packet header overhead**: TCP header = **20 bytes**; UDP header = only **8 bytes**.

- Applications using UDP: DNS, network management (**SNMP**) — since management often must run precisely when the network is stressed, when reliable congestion-controlled transfer is hard to achieve anyway. Many multimedia apps (Internet phone, video conferencing, streaming) can use either UDP or TCP, though real-time apps react poorly to TCP's congestion control and tolerate some packet loss, favoring UDP — though with low loss rates and firewalls sometimes blocking UDP, TCP is increasingly used for streaming too.
- **Running multimedia over UDP is controversial**: UDP has **no congestion control**. If everyone streamed high-bit-rate video uncontrolled, router buffers would overflow badly, and this would also force TCP senders (who **do** back off under loss) to dramatically cut their rates — UDP traffic could "crowd out" TCP sessions. Researchers have proposed mechanisms to force adaptive congestion control even on UDP sources.
- It's possible to build **reliable data transfer on top of UDP** — by implementing acknowledgments/retransmissions in the application itself (as QUIC does) — a nontrivial task, but it lets the app be reliable **without** being subjected to TCP's congestion-control throttling.

### 3.3.1 UDP Segment Structure
- Defined in RFC 768. Header has only **four fields, 2 bytes each** (8 bytes total): **source port, destination port, length** (bytes in segment — header + data; needed since data-field size varies), **checksum**.
- The **application data** fills the data field (e.g., DNS query/response, or audio samples for streaming).

### 3.3.2 UDP Checksum
- Used for **error detection** — determines whether bits in the segment were altered in transit (e.g., by link noise or while buffered in a router).
- **Computation**: sender takes the **1's complement of the sum** of all 16-bit words in the segment (with any overflow **wrapped around**), and this result becomes the checksum field.
- **Worked example**: three 16-bit words `0110011001100000`, `0101010101010101`, `1000111100001100`.
  - Sum of first two = `1011101110110101`.
  - Adding the third (with overflow wraparound) = `0100101011000010`.
  - **1's complement** (flip all bits) = `1011010100111101` → this is the checksum.
  - At the receiver: sum **all four** 16-bit words including the checksum. If no errors, the result should be **all 1s** (`1111111111111111`); if any bit is 0, an error has been introduced.
- **Why does UDP checksum at all**, given link-layer protocols (e.g., Ethernet) often already do error checking? Because **not all links** along a path are guaranteed to perform error checking, and bit errors can also occur while a segment sits in a **router's memory**. Since neither link-by-link reliability nor in-memory error detection is guaranteed, UDP provides **end-to-end** error detection at the transport layer — an example of the **end-end principle** [Saltzer 1984]: "functions placed at lower levels may be redundant or of little value compared to the cost of providing them at a higher level."
- UDP detects but **does not recover** from errors — some implementations simply **discard** the corrupted segment; others pass it up to the application **with a warning**.

---

## 3.4 Principles of Reliable Data Transfer

- A central, "top-ten" fundamental problem in all of networking — relevant at the link, network, transport, AND application layers, not just transport.
- **Service abstraction**: a **reliable channel** through which data is transferred with **no corruption, no loss, and correct ordering** — exactly TCP's service model to applications.
- The task is difficult because the layer **below** the reliable protocol may itself be unreliable (e.g., TCP runs on top of unreliable, best-effort IP).
- Notation: `rdt` = reliable data transfer protocol; `rdt_send()` (sender's upper-layer call), `rdt_rcv()` (called when a packet arrives), `deliver_data()` (delivers to upper layer), `udt_send()` (send via the **u**nreliable underlying channel).
- Discussion is restricted to **unidirectional** data transfer for simplicity (though both sides still exchange packets — data packets one way, and control packets like ACKs the other way).
- Assumption throughout: the underlying channel **does not reorder** packets (though it may lose or corrupt them).

### 3.4.1 Building a Reliable Data Transfer Protocol

**rdt1.0 — reliable transfer over a perfectly reliable channel**
- Trivial: sender just packages data and sends; receiver just extracts data and delivers it up. Both sender and receiver FSMs have a **single state**. No feedback needed since nothing can go wrong.

**rdt2.0 — a channel with bit errors (no loss)**
- **ARQ (Automatic Repeat reQuest)** protocols use three additional capabilities:
  1. **Error detection** (extra bits — a checksum).
  2. **Receiver feedback**: ACK (positive) / NAK (negative) acknowledgments.
  3. **Retransmission**: sender resends packets that were reported in error.
- **Stop-and-wait protocol**: sender sends one packet, then waits for ACK/NAK before sending the next — cannot accept new upper-layer data while waiting.
- **Fatal flaw**: doesn't account for the ACK/NAK itself getting **corrupted**! If a NAK/ACK is garbled, the sender can't know what happened.

**rdt2.1 — fixing corrupted ACK/NAK via sequence numbers**
- Simple fix: add a **1-bit sequence number** to each data packet. Receiver checks the sequence number to tell whether an arriving packet is a **retransmission** (same seq #) or **new data** (seq # flips 0↔1). Since the channel doesn't lose packets (yet), ACK/NAK don't need to carry a sequence number — the sender knows any ACK/NAK it gets corresponds to its most recently sent packet.
- Sender and receiver FSMs now have **twice as many states** (must track whether the current/expected packet is seq# 0 or 1).

**rdt2.2 — a NAK-free protocol**
- Same effect as a NAK can be achieved by having the receiver send an **ACK for the last correctly received packet** instead of a NAK. A sender receiving **two ACKs for the same packet** (duplicate ACKs) knows the packet **following** the doubly-ACKed one was not received correctly.
- Now the ACK message itself must carry the sequence number of the packet being acknowledged.

**rdt3.0 — a lossy channel with bit errors (alternating-bit protocol)**
- Now the channel can also **lose** packets (data OR ACK/NAK). Burden of detecting/recovering from loss is placed on the **sender**, using a **countdown timer**.
- Sender must: (1) start the timer whenever a packet (new or retransmitted) is sent, (2) respond to a timer interrupt (retransmit), (3) stop the timer once an ACK arrives.
- **How long to wait?** At least an RTT + processing time — but this worst case is hard to know precisely, so in practice the sender picks a value where loss is **likely but not guaranteed** to have occurred; a too-early timeout causes an unnecessary retransmission (and thus **duplicate packets**), which rdt2.2's machinery (sequence numbers) already handles correctly.
- From the sender's view, **retransmission is a panacea**: whether the data packet was lost, the ACK was lost, or either was just overly delayed — the response is always the same: **retransmit**.
- Because sequence numbers alternate 0/1, rdt3.0 is also called the **alternating-bit protocol**.

### 3.4.2 Pipelined Reliable Data Transfer Protocols
- rdt3.0 is correct but has **terrible performance** because it's **stop-and-wait**.
- **Worked example** (West Coast ↔ East Coast, RTT ≈ 30 ms, link rate R = 1 Gbps, packet L = 1,000 bytes = 8,000 bits):
  - $d_{trans} = L/R = 8000/10^9 = 8\ \mu s$.
  - Sender utilization: $U_{sender} = \dfrac{L/R}{RTT + L/R} = \dfrac{0.008}{30.008} \approx 0.00027$ — the sender is busy only **2.7 hundredths of one percent** of the time!
  - Effective throughput: only **267 kbps**, despite a 1 Gbps link — a graphic illustration of how a poorly designed protocol can bottleneck powerful hardware.
- **Solution: pipelining** — sender transmits **multiple packets** without waiting for each ACK. E.g., allowing 3 packets in flight roughly **triples** utilization.
- Consequences of pipelining:
  1. Range of **sequence numbers** must increase (multiple unACKed in-flight packets, each needs a unique number).
  2. Sender/receiver must **buffer more than one packet**.
  3. Two basic approaches to pipelined error recovery: **Go-Back-N (GBN)** and **Selective Repeat (SR)**.

### 3.4.3 Go-Back-N (GBN)
- Sender may transmit multiple packets without waiting for ACKs, but is limited to **N** unacknowledged packets in the pipeline at once (**window size N**) — hence GBN is a **sliding-window protocol**.
- Sequence-number ranges (sender's view): **[0, base−1]** = already ACKed; **[base, nextseqnum−1]** = sent, not yet ACKed; **[nextseqnum, base+N−1]** = can be sent immediately; **≥ base+N** = cannot be used until the packet at `base` is ACKed.
- With a k-bit sequence number field, the range is [0, 2^k − 1], and arithmetic is done **modulo 2^k**.
- **Sender responds to 3 event types**:
  1. **Invocation from above**: if window not full, send a new packet; if full, refuse (return data to upper layer, to be retried later).
  2. **Receipt of an ACK**: GBN uses **cumulative acknowledgments** — an ACK for packet n means all packets up to and including n have been correctly received.
  3. **Timeout**: uses only **a single timer** (for the oldest unACKed packet). On timeout, the sender **resends ALL** packets that have been sent but not yet ACKed — hence the name "Go-**Back**-N."
- **Receiver behavior**: If packet n arrives correctly **and in order**, ACK it and deliver to the upper layer. **Any other case** (out-of-order or corrupted) → **discard the packet** and re-send an ACK for the most recently received in-order packet.
  - **Why discard out-of-order packets rather than buffer them?** Simplicity — the receiver only needs to track `expectedseqnum`, no buffering of out-of-order data needed. Downside: a lost packet forces retransmission of it AND all subsequent packets already sent (even ones that arrived correctly) — wasteful when error rates are non-trivial.
- GBN's mechanisms (sequence numbers, cumulative ACKs, checksums, timeout/retransmit) map closely onto how **TCP** implements reliable data transfer (§3.5).

### 3.4.4 Selective Repeat (SR)
- GBN's weakness: with a large window and high bandwidth-delay product, **one lost packet triggers retransmission of many packets**, many unnecessarily.
- **SR**: sender retransmits **only those packets it suspects were lost/corrupted** — requires the receiver to **individually acknowledge** each correctly received packet (not just cumulatively).
- Receiver **buffers out-of-order packets**, delivering a batch to the upper layer once the gap is filled.
- **Reacknowledgment**: the SR receiver must **re-ACK already-received packets below the current window base** — otherwise, if the corresponding ACK is lost, the sender's window would never advance, since the sender would eventually retransmit that already-received packet and needs an ACK for it to move on.
- **Sender/receiver window desynchronization**: because sender and receiver don't always have an identical view of what's been received, their windows won't always coincide. This creates a subtlety with **finite sequence-number ranges**: if the window size is too large relative to the sequence-number space, the receiver **cannot distinguish** a retransmission of an old packet from a brand-new packet carrying new data (a worked example with 4 sequence numbers {0,1,2,3} and window size 3 shows exactly this ambiguity). **Rule**: for SR, **window size must be ≤ half** the size of the sequence-number space.
- **Summary table of RDT mechanisms** (from Table 3.1):

| Mechanism | Use |
|---|---|
| **Checksum** | Detect bit errors in a transmitted packet |
| **Timer** | Timeout/retransmit a packet possibly lost; can cause duplicate packets on premature timeout |
| **Sequence number** | Sequential numbering lets receiver detect gaps (loss) and duplicates |
| **Acknowledgment** | Receiver tells sender a packet (or set of packets) was received correctly; may be individual or cumulative |
| **Negative acknowledgment** | Receiver tells sender a packet was NOT received correctly |
| **Window, pipelining** | Sender may send only packets within a given sequence-number range, allowing multiple outstanding unACKed packets — improves utilization over stop-and-wait |

- **Packet reordering caveat**: real networks (unlike a single physical wire) CAN reorder packets, meaning old duplicate packets with a reused sequence number could resurface unexpectedly. Practice avoids this by ensuring sequence numbers aren't reused until the sender is sure any earlier packet with that number is no longer in the network — assuming a maximum packet lifetime (~3 minutes is assumed in TCP extensions for high-speed networks, RFC 1323).

---

## 3.5 Connection-Oriented Transport: TCP

TCP is defined across **RFC 793, 1122, 1323, 2018, 2581**.

### 3.5.1 The TCP Connection
- **Connection-oriented**: requires a **handshaking** procedure before data transfer, initializing state variables on both sides.
- The TCP "connection" is **not** a circuit-switched TDM/FDM circuit — it's a **logical** connection, with state existing **only in the two end systems**; intermediate routers are completely oblivious to TCP connections (they see datagrams, not connections).
- TCP provides **full-duplex** service (data flows both directions simultaneously) and is always **point-to-point** (one sender, one receiver — no TCP multicasting: "two hosts are company and three are a crowd").
- **Case History — Vinton Cerf, Robert Kahn, and TCP/IP**: In the early 1970s, many incompatible packet-switched networks existed. Cerf and Kahn recognized the need to interconnect them and invented **TCP/IP** (originally one protocol, later split into TCP + IP). Published in **May 1974** (IEEE Trans. on Communications Technology). Received the **ACM Turing Award in 2004** for this and related work.
- **Connection establishment**: client's `connect()` triggers a **three-way handshake**: (1) client sends a special segment, (2) server responds with a second special segment, (3) client responds with a third segment — the first two carry no payload; the third **may** carry payload.
- **Send/receive buffers**: once connected, application data passes through the socket into a **send buffer** (allocated during the handshake). TCP grabs chunks from this buffer "at its own convenience" (per RFC 793) and passes them to the network layer. Each side has **its own send buffer and receive buffer**. No buffers/variables are allocated in intermediate network elements.
- **MSS (Maximum Segment Size)**: the max amount of app-layer data TCP will grab per segment — set based on the **MTU** (Maximum Transmission Unit, the largest link-layer frame the local host can send) so that a TCP segment + TCP/IP headers (typically 40 bytes) fits into a single link-layer frame. Ethernet/PPP MTU = 1,500 bytes → typical **MSS = 1,460 bytes**. (Path MTU discovery, RFC 1191, can also inform MSS selection.) **MSS is the max app-data size, NOT the total segment size including headers** — a commonly confused point.

### 3.5.2 TCP Segment Structure
- Header + data field; data field size ≤ MSS. Interactive apps (e.g., Telnet) often send chunks much smaller than MSS — sometimes just **1 byte**, making the total segment (with a 20-byte header) only 21 bytes.
- **Header fields**: source/destination port (multiplex/demultiplex), **32-bit sequence number**, **32-bit acknowledgment number** (both central to reliable data transfer, discussed below), **16-bit receive window** (flow control — bytes receiver is willing to accept), **4-bit header length** field (header length in 32-bit words; typically **20 bytes** when the options field is empty), **options field** (variable-length; used to negotiate MSS or a window-scaling factor for high-speed networks; also a timestamping option — RFC 854, RFC 1323), **checksum**, **flag field (6 bits)**:
  - **ACK**: indicates the acknowledgment field's value is valid.
  - **RST, SYN, FIN**: connection setup/teardown.
  - **CWR, ECE**: used in **Explicit Congestion Notification** (§3.7.2).
  - **PSH**: tells the receiver to pass data to the upper layer immediately.
  - **URG** + **16-bit urgent data pointer**: marks "urgent" data (in practice, PSH/URG/urgent-pointer are **rarely used**, but exist for completeness).

**Sequence Numbers and Acknowledgment Numbers**
- TCP views data as an **unstructured but ordered stream of bytes** — sequence numbers are over the **byte stream**, NOT over segments.
- The **sequence number of a segment** = the byte-stream number of the **first byte** in that segment.
  - Example: 500,000-byte file, MSS=1,000 bytes, byte 0 is first → segment 1 gets seq# 0, segment 2 gets seq# 1000, segment 3 gets seq# 2000, etc.
- **Acknowledgment number** that Host A puts in a segment to B = the sequence number of the **next byte** A is expecting from B.
  - Example: A has received bytes 0–535 from B → A's next segment to B carries **ACK = 536**.
  - Because TCP only ACKs **up to the first missing byte** in the stream, TCP provides **cumulative acknowledgments**.
  - Out-of-order segments: TCP RFCs leave the choice to implementers — either **discard** them (simpler) or **buffer** them and wait for gaps to fill (more bandwidth-efficient — the approach taken in practice).
  - Both sides choose their **initial sequence number randomly** (not 0) — to minimize the chance an old segment from an earlier, already-terminated connection (using the same port numbers) is mistaken for valid data in a new connection.
- **Telnet case study** (illustrates seq/ack numbers): client starting seq# = 42, server starting seq# = 79. User types 'C' → 3 segments exchanged:
  1. Client → Server: seq=42 (data='C'), ack=79 (client hasn't received anything from server yet).
  2. Server → Client: seq=79 (data='C', echoed back), ack=43 (**piggybacked** acknowledgment of client's byte 42).
  3. Client → Server: seq=43, ack=80, **empty data field** (pure acknowledgment of server's echoed byte).

### 3.5.3 Round-Trip Time Estimation and Timeout
Based on [Jacobson 1988] and RFC 6298.

- **SampleRTT**: time between sending a segment and receiving its ACK. Most implementations take **only one SampleRTT measurement at a time** (roughly once per RTT) and **never** compute SampleRTT for a **retransmitted** segment (Karn's algorithm — ambiguous which transmission the ACK corresponds to).
- **EstimatedRTT**: an **EWMA** (Exponentially Weighted Moving Average) of SampleRTT values:
$$EstimatedRTT = (1-\alpha) \cdot EstimatedRTT + \alpha \cdot SampleRTT$$
  - Recommended **α = 0.125** (1/8), i.e., $EstimatedRTT = 0.875 \cdot EstimatedRTT + 0.125 \cdot SampleRTT$ — puts more weight on **recent** samples.
- **DevRTT**: estimate of how much SampleRTT typically **deviates** from EstimatedRTT:
$$DevRTT = (1-\beta) \cdot DevRTT + \beta \cdot |SampleRTT - EstimatedRTT|$$
  - Recommended **β = 0.25**. Small DevRTT = little fluctuation; large DevRTT = lots of fluctuation.
- **TimeoutInterval**:
$$TimeoutInterval = EstimatedRTT + 4 \cdot DevRTT$$
  - Initial value recommended: **1 second**. On a timeout event, `TimeoutInterval` is **doubled** (rather than recomputed) to avoid premature timeouts on the next attempt; once a fresh SampleRTT/ACK arrives, it reverts to the formula above.

### 3.5.4 Reliable Data Transfer
- TCP builds reliable transfer atop IP's unreliable, best-effort service. TCP's guarantee: the byte stream read by the receiving app is **uncorrupted, without gaps, without duplication, and in sequence**.
- TCP uses only **a single retransmission timer** even with multiple outstanding unACKed segments (unlike the "one timer per unACKed packet" idealization).
- **Simplified TCP sender — three major events**:
  1. **Data received from application above**: encapsulate, pass to IP; start the timer if it isn't already running (conceptually, the timer tracks the **oldest** unACKed segment).
  2. **Timeout**: retransmit the segment with the smallest sequence number that hasn't been ACKed yet; restart the timer.
  3. **ACK receipt** (with value y): compare y to `SendBase` (seq# of oldest unACKed byte). If `y > SendBase`, this ACK covers new data → advance `SendBase`; restart the timer if any segments remain unACKed.
- **Scenario walkthroughs** in the text:
  - **Lost ACK**: A sends a segment; B's ACK is lost; A's timer expires; A retransmits; B discards the duplicate (recognized via seq#).
  - **Two back-to-back segments, both ACKs delayed past timeout**: only the **first** (oldest unACKed) segment gets retransmitted on timeout; if the second segment's ACK arrives before its own timeout, it isn't resent.
  - **Cumulative ACK avoids retransmission**: if the ACK for the first of two segments is lost but a **later cumulative ACK** (covering both) arrives before timeout, **neither segment is retransmitted**.
- **Doubling the timeout interval**: on each retransmission-triggered timeout, TCP **doubles** the next timeout value (rather than recomputing from EstimatedRTT/DevRTT) — giving **limited congestion control**: politely backing off with longer and longer intervals when the network seems congested, instead of persistently hammering it. (Ethernet's CSMA/CD, Ch. 6, uses a similar idea.)
- **Fast Retransmit**: waiting for a full timeout to detect loss can be slow. A **duplicate ACK** (re-ACKing a segment already ACKed) signals a possible gap in the data stream. TCP's ACK generation rules (RFC 5681, summarized):

| Event | Receiver Action |
|---|---|
| In-order segment, expected seq#, all prior data already ACKed | **Delayed ACK**: wait up to 500ms for another in-order segment; if none arrives, send ACK |
| In-order segment, expected seq#, one other in-order segment already waiting for ACK | Immediately send a single **cumulative ACK** for both |
| Out-of-order segment, higher-than-expected seq# (gap detected) | Immediately send a **duplicate ACK** for the next expected byte |
| Segment that partially/completely fills a gap | Immediately ACK, provided it starts at the low end of the gap |

  - If the sender receives **3 duplicate ACKs** for the same data, it infers the following segment was **lost**, and performs a **fast retransmit** — resending the missing segment **before** its timer even expires.
- **Is TCP GBN or SR?** TCP uses **cumulative ACKs**, no individual ACKs for out-of-order segments → looks like **GBN**. But many implementations **buffer** correctly-received out-of-order segments (unlike strict GBN), and TCP retransmits **at most the single lost segment** (not all subsequent ones) when only one ACK is lost but later cumulative ACKs arrive in time. **Selective ACKnowledgment (SACK)** (RFC 2018) — an optional TCP extension — lets a receiver acknowledge out-of-order segments selectively, and combined with selective retransmission, makes TCP behave more like **SR**. Overall: **TCP's error recovery is best categorized as a hybrid of GBN and SR**.

### 3.5.5 Flow Control
- **Flow control** = a **speed-matching service** — ensuring the sender doesn't overflow the receiver's buffer, distinct from congestion control (which throttles based on **network** congestion, not receiver buffer capacity — though the mechanisms/actions can look similar and are often conflated).
- TCP implements flow control via the sender-maintained **receive window (rwnd)** variable. Each side of a (full-duplex) connection maintains its own distinct receive window.
- Definitions: `LastByteRead` (last byte read from the buffer by the app), `LastByteRcvd` (last byte placed in the receive buffer). Constraint (buffer can't overflow):
$$LastByteRcvd - LastByteRead \le RcvBuffer$$
- **rwnd** = spare room in the buffer:
$$rwnd = RcvBuffer - [LastByteRcvd - LastByteRead]$$
- Host B places its current `rwnd` value in **every segment it sends** to A. Host A tracks `LastByteSent` and `LastByteAcked`; ensures:
$$LastByteSent - LastByteAcked \le rwnd$$
- **Zero-window deadlock problem**: if B's buffer fills (`rwnd=0`) and B has nothing further to send to A, A never learns when space frees up (TCP only sends a segment when it has data or an ACK to send) — A would be **permanently blocked**. **Fix**: TCP requires Host A to keep sending **1-byte segments** while B's window is 0; B's ACKs to these probes eventually carry a nonzero `rwnd` once space frees up.
- **UDP provides NO flow control** — segments can simply be **lost at the receiver due to buffer overflow** if the receiving process doesn't read fast enough.

### 3.5.6 TCP Connection Management
- **Three-way handshake** (establishment):
  1. **SYN**: client sends a segment with **SYN bit = 1**, no payload, and a randomly-chosen initial sequence number `client_isn`.
  2. **SYNACK**: server extracts the SYN, **allocates buffers/variables** for the connection, replies with SYN=1, ACK field = `client_isn+1`, and its own randomly-chosen `server_isn` in the sequence field. (Allocating resources at this step, before the handshake completes, is what makes TCP vulnerable to **SYN flooding**, discussed below.)
  3. **ACK**: client also allocates buffers/variables, sends a final segment with SYN=0 and ACK field = `server_isn+1` — this segment **may carry client-to-server data**.
- **Connection teardown** (either side can initiate): e.g., client sends **FIN=1** → server ACKs it → server sends its **own FIN=1** → client ACKs that → resources on both sides deallocated.
- **TCP client-side states**: `CLOSED` → (app opens socket) → `SYN_SENT` (after sending SYN) → `ESTABLISHED` (after receiving SYNACK) → (app closes) `FIN_WAIT_1` (after sending FIN) → `FIN_WAIT_2` (after receiving an ACK of the FIN) → `TIME_WAIT` (after receiving the server's FIN and ACKing it) → after a timeout (typical: **30 sec, 1 min, or 2 min**) the connection fully closes and resources (incl. port numbers) are released. **`TIME_WAIT`** exists so the client can **re-send its final ACK** in case that ACK is lost.
- **Server-side states**: mirror sequence, not detailed line-by-line here but analogous (LISTEN → SYN_RCVD → ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED).
- **RST segments**: if a host receives a TCP segment whose port/IP doesn't match any ongoing socket (e.g., a SYN to a port with no listening server), it replies with a segment having the **RST flag = 1** — "I don't have a socket for that; please don't resend."
- **nmap port-scanning outcomes** for a probed TCP port: (1) **SYNACK received** → port reported **"open"**; (2) **RST received** → target reachable but port not in use (also reveals no firewall blocking that path); (3) **nothing received** → likely blocked by an intervening **firewall**.
- **Focus on Security — SYN Flood Attack**: exploits the fact that a server **allocates resources upon receiving a SYN**, before the handshake completes. Attacker(s) flood the server with SYNs and never complete step 3 → server's connection resources get exhausted servicing **half-open connections**, denying service to legitimate clients — one of the **first documented DoS attacks**.
  - **Defense — SYN cookies** (RFC 4987): instead of allocating state on receiving a SYN, the server computes an **initial sequence number as a hash function** of (source/destination IP, source/destination port, a secret number known only to the server) — this is the "cookie" — and sends it in the SYNACK, **without storing any state**. A legitimate client's returning ACK (value = cookie+1) lets the server **recompute the same hash** and verify the ACK corresponds to a real earlier SYN — only then does it allocate a full connection. If no ACK ever returns (i.e., it was a bogus/attack SYN), **no resources were ever wasted** in the first place.

---

## 3.6 Principles of Congestion Control

- Packet loss (from router buffer overflow) is a **symptom**; the underlying **cause** is too many sources sending at too high a rate. Congestion control mechanisms address the **cause**, not just retransmit lost packets.

### 3.6.1 The Causes and the Costs of Congestion — Three Scenarios

**Scenario 1 — two senders, a router with infinite buffers**
- Two hosts A, B send at average rate λ_in into a shared link of capacity R (via one router with **infinite** buffer space).
- For λ_in ≤ R/2: throughput = sending rate (no loss). For λ_in > R/2: throughput **caps at R/2** — the link simply cannot deliver more, no matter how high the senders push.
- **Cost of congestion #1**: as sending rate approaches R/2, **average delay grows without bound** (with infinite buffering, queue length → ∞ as rate → R/2⁺) — large queuing delays near link capacity.

**Scenario 2 — two senders, a router with finite buffers (with retransmission)**
- Now buffers are **finite** → packets can be **dropped**; connections are **reliable** (lost segments get retransmitted). Let λ'_in = the "**offered load**" (original + retransmitted data).
- **Idealized case** (sender magically knows when a buffer is free, never causes loss): throughput = λ_in exactly (up to R/2) — no waste, ideal.
- **More realistic case** (sender retransmits only when a segment is *known for certain* lost): at offered load λ'_in = R/2, actual **delivered throughput is only R/3** — i.e., of the 0.5R sent, ~0.333R is original data and ~0.166R is retransmitted data.
  - **Cost of congestion #2**: the sender must perform **retransmissions** to compensate for buffer-overflow losses — reduces useful throughput.
- **Even more realistic case** (sender sometimes times out **prematurely**, retransmitting an un-lost, merely-delayed packet — so the receiver gets duplicate copies, discarding the redundant one): if each packet is (on average) forwarded **twice** by the router, throughput asymptotes at only **R/4** as offered load approaches R/2.
  - **Cost of congestion #3**: the router wastes link capacity forwarding **unneeded duplicate copies** of a packet that could have instead carried a different, useful packet.

**Scenario 3 — four senders, routers with finite buffers, multihop paths**
- Four hosts, overlapping two-hop paths through routers R1 and R2. Consider the A→C connection through R1 and R2, competing with the D→B connection (at R1) and the B→D connection (at R2).
- For very small λ_in: throughput ≈ offered load (rare overflow). For very large λ_in: A–C traffic arriving at R2 (after already consuming R1's capacity to get there) competes with B–D traffic filling R2's buffer — as offered load → ∞, **A–C's end-to-end throughput → 0**.
- **Cost of congestion #4 (the most important)**: when a packet is dropped **downstream** (e.g., at R2), all the transmission capacity used by **upstream** routers (e.g., R1) to forward that now-doomed packet toward R2 was **completely wasted** — the network would have been equally (bad) off, and better used that capacity for a different packet, if R1 had simply discarded the packet up front instead of forwarding it.

### 3.6.2 Approaches to Congestion Control
Two broad approaches:
1. **End-to-end congestion control**: network layer provides **no explicit support**; end systems must **infer** congestion purely from observed behavior (packet loss and/or delay). **TCP takes this approach** — since IP gives no feedback about congestion, TCP infers congestion from timeouts / triple-duplicate-ACKs (and, more recently, from rising RTT delay as an alternative signal).
2. **Network-assisted congestion control**: routers provide **explicit feedback** to the sender/receiver about the network's congestion state.
   - Simplest form: a **single bit** indicating link congestion (early IBM SNA, DEC DECnet, ATM architectures).
   - More sophisticated: e.g., ATM ABR (Available Bit Rate) lets a router tell the sender the **maximum rate** it can support on an outgoing link.
   - Internet's default IP/TCP is end-to-end, but **may optionally** implement network-assisted congestion control (ECN, §3.7.2).
   - Two feedback pathways: **(a) direct feedback** from a router to the sender (a "**choke packet**," essentially "I'm congested!"), or **(b) indirect feedback** — a router marks a field in a packet flowing sender→receiver, and the receiver then notifies the sender — this takes a full RTT to reach the sender.

---

## 3.7 TCP Congestion Control

TCP must use **end-to-end** congestion control since IP gives no explicit congestion feedback.

- **Congestion window (`cwnd`)**: a sender-side variable imposing a hard constraint:
$$LastByteSent - LastByteAcked \le \min\{cwnd,\ rwnd\}$$
- Assuming the receive buffer is large enough that `rwnd` isn't the binding constraint, and that the sender always has data ready to send: the constraint on **unacknowledged data** indirectly limits the sender's **rate**. Roughly, the sender's rate ≈ **cwnd / RTT** bytes/sec — so adjusting `cwnd` adjusts the send rate.
- **"Loss event"**: defined as either a **timeout** OR the **receipt of three duplicate ACKs**.
- **Self-clocking**: TCP uses the **arrival of ACKs** to trigger/pace its increases in `cwnd` — if ACKs arrive slowly (high-delay or low-bandwidth path), `cwnd` grows slowly; if ACKs arrive fast, it grows fast.
- **Three guiding principles**:
  1. A **lost segment implies congestion** → decrease sending rate.
  2. An **acknowledged segment implies the network is delivering data successfully** → increase sending rate.
  3. **Bandwidth probing**: TCP repeatedly increases its rate (probing for more available bandwidth) until loss occurs, backs off, then probes again — a purely **distributed, local-information-only** process; senders act asynchronously with **no explicit coordination**.

**Three major algorithm components** (first described in [Jacobson 1988], standardized in RFC 5681): **slow start**, **congestion avoidance**, **fast recovery**. Slow start & congestion avoidance are **mandatory**; fast recovery is **recommended but not required**.

### Slow Start
- `cwnd` initialized to a small value: **1 MSS** (RFC 3390) → initial rate ≈ MSS/RTT (e.g., MSS=500B, RTT=200ms → only ~20 kbps).
- During slow start, `cwnd` **increases by 1 MSS for every newly-ACKed segment** — so `cwnd` effectively **doubles every RTT** (**exponential growth** — "slow" start is a bit of a misnomer, since it actually grows very fast relative to congestion avoidance).
- **Slow start ends** in one of three ways:
  1. **Timeout occurs** → `cwnd` reset to **1 MSS**; `ssthresh` (slow-start threshold) set to **cwnd/2** (half the value when congestion was detected); slow start restarts.
  2. `cwnd` reaches (or exceeds) `ssthresh` → TCP transitions to **congestion avoidance**.
  3. **Three duplicate ACKs detected** → TCP performs **fast retransmit** and enters **fast recovery**.

### Congestion Avoidance
- Entered with `cwnd` ≈ half its pre-loss value — congestion could be "just around the corner," so TCP is now more cautious: increases `cwnd` by just **1 MSS per RTT** (**linear/additive increase**), typically implemented as **cwnd += MSS × (MSS/cwnd)** per newly-ACKed segment.
- **On timeout**: same as slow start — `cwnd → 1 MSS`, `ssthresh → cwnd/2`, back to slow start.
- **On triple-duplicate-ACK**: less drastic response than a timeout, since the network is clearly still delivering some segments — `cwnd` is **halved** (plus 3 MSS to account for the 3 duplicate ACKs received), `ssthresh` set to half of `cwnd` at the loss point → enters **fast recovery**.

### Fast Recovery
- `cwnd` is increased by **1 MSS for each further duplicate ACK** received for the missing segment.
- Once an ACK finally arrives for the missing segment, TCP **deflates `cwnd`** and moves to **congestion avoidance**.
- If a **timeout** occurs instead while in fast recovery, TCP transitions to **slow start** (same as the other two states: `cwnd→1 MSS`, `ssthresh→cwnd/2`).
- **TCP Tahoe vs. TCP Reno**: Tahoe (older) **unconditionally** cuts `cwnd` to 1 MSS and re-enters slow start after **either** a timeout **or** a triple-duplicate-ACK loss event. **Reno** (newer) added **fast recovery**, handling triple-duplicate-ACK loss more gently (as described above) while still falling back to slow start on a true timeout.

### TCP Congestion Control — Retrospective (AIMD)
- Ignoring the initial slow-start ramp-up and assuming losses are signaled by triple-dup-ACKs (not timeouts): TCP's steady-state behavior is **linear (additive) increase of 1 MSS per RTT, then a halving (multiplicative decrease) on loss** → **AIMD (Additive-Increase, Multiplicative-Decrease)** — produces the characteristic **"saw-tooth"** throughput pattern, embodying the bandwidth-probing intuition.
- Other TCP variants exist: **TCP Vegas** — tries to detect congestion **before** loss occurs, by watching for **rising RTT** as an early congestion signal, then linearly lowering rate preemptively. **BIC** is another example. Ubuntu Linux (as of late 2015) defaults to slow start + congestion avoidance + fast recovery + fast retransmit + SACK, with Vegas/BIC available as alternatives.
- **Theoretical grounding**: ~10 years after TCP's original development, theory [Kelly 1998] showed TCP's AIMD congestion control functions as a **distributed asynchronous optimization algorithm**, simultaneously optimizing several aspects of network/user performance — spawning a rich subsequent theory of congestion control.

**Macroscopic Throughput Formula**
- Ignoring post-timeout slow-start phases: if `w` = current window and `RTT` = round-trip time, sending rate ≈ w/RTT. Let **W** = the value of `w` right when a loss event occurs. Rate ranges from **W/(2·RTT)** to **W/RTT**, growing linearly in between → average:
$$\text{average throughput} = 0.75 \cdot \frac{W}{RTT}$$
- A related, more useful formula relates throughput to **loss rate L**:
$$\text{average throughput} = 1.22 \cdot \frac{MSS}{RTT\sqrt{L}}$$
- **High-bandwidth-path implications**: e.g., to sustain **10 Gbps** over a 1,500-byte-segment, 100ms-RTT connection, the average `cwnd` would need to be **83,333 segments**! Using the throughput-vs-loss formula, sustaining 10 Gbps requires a segment loss probability of only **2×10⁻¹⁰** (one loss per 5 **billion** segments) — an extremely low tolerance, motivating research into **high-speed TCP variants** better suited to such demanding environments.

### 3.7.1 Fairness
- A congestion-control mechanism is **fair** if, for K TCP connections sharing one bottleneck link of rate R (with no competing UDP traffic), each gets on average **≈ R/K**.
- **Why TCP's AIMD converges toward fairness** (two connections, same MSS/RTT, sharing one link, ignoring slow start): plotting the two connections' throughputs on x/y axes, additive increase moves the joint operating point along a **45° line** (equal increase for both) until loss occurs; multiplicative decrease then **halves both**, moving the point **halfway toward the origin along the vector from the loss point**. This cycle **converges** the two connections' throughputs toward the **equal-bandwidth-share line**, regardless of the starting point. (This idealized result assumes only TCP traffic, identical RTTs, and one connection per host pair — often untrue in practice; e.g., connections with **smaller RTT** can grab freed bandwidth faster and thus enjoy a **larger** share.)
- **Fairness and UDP**: Many multimedia apps run over UDP specifically to **avoid** TCP's rate throttling — pumping data at a constant rate and tolerating loss, rather than backing off. This is inherently **unfair** from TCP's perspective, and it's possible for aggressive UDP sources to **crowd out** TCP traffic — an active area of research (mechanisms to force adaptive congestion control onto UDP-based sources too).
- **Fairness and parallel TCP connections**: even well-behaved TCP apps can grab a disproportionate share by opening **multiple parallel connections** (e.g., a Web browser's multiple connections per page). Example: link R shared fairly among 9 single-connection apps (each getting R/10 after a 10th single-connection app joins) — but if that 10th app instead opens **11 parallel connections**, it gets **>R/2** instead of R/10.

### 3.7.2 Explicit Congestion Notification (ECN): Network-Assisted Congestion Control
- Extensions to IP and TCP (RFC 3168) that let the **network explicitly signal congestion** — a form of **network-assisted** congestion control layered on top of TCP's fundamentally end-to-end design.
- Uses **2 bits** in the IP header's **Type of Service** field. One setting: a router marks a datagram to indicate **it is experiencing congestion** (RFC 3168 doesn't define exactly *when* — that's a vendor/operator configuration choice, though **persistent** congestion is recommended as the trigger). A second setting: the sending host signals that **both** sender and receiver are **ECN-capable**.
- **Flow**: router marks the IP datagram → receiving host's TCP sees the marked datagram → receiver sets the **ECE (ECN-Echo)** bit in its next TCP ACK back to the sender → sender reacts by **halving `cwnd`** (just as with a loss event triggering fast retransmit) and sets the **CWR (Congestion Window Reduced)** bit on its next outgoing segment.
- Other protocols leveraging network-layer ECN: **DCCP** (Datagram Congestion Control Protocol, RFC 4340 — a low-overhead, congestion-controlled UDP-like unreliable service) and **DCTCP** (Data Center TCP) — a TCP variant designed specifically for data-center networks.

---

## Principles in Practice — TCP Splitting (Optimizing Cloud Service Performance)
- Cloud services (search, e-mail, social networks) want the responsiveness of a **local** app, but users are often far from the data center → **large RTT** → **slow-start delay** can significantly hurt response time.
- **Example**: a search response typically needs **3 TCP windows** during slow start → total delay ≈ **4×RTT** (1 RTT for connection setup + 3 RTTs for the 3 windows) plus data-center processing time.
- **TCP splitting**: deploy **front-end servers** geographically close to users; the client's TCP connection terminates at this nearby front end, which **maintains its own persistent, large-`cwnd` TCP connection** back to the data center (back-end).
  - Resulting delay ≈ **4·RTT_FE + RTT_BE + processing time**, where RTT_FE (client↔front-end) is small and RTT_BE (front-end↔data-center) ≈ the original full RTT — so total delay drops roughly from **4·RTT to just RTT** plus processing.
  - Also reduces retransmission delays caused by access-network losses (since the "long" backend leg uses an already-warmed-up, large-window connection).
  - Used extensively by **Google and Akamai** via their CDN servers deployed in access networks.

---

## 3.8 Summary

- **UDP**: no-frills — multiplexing/demultiplexing + a light checksum, nothing more.
- **Reliable data transfer** (§3.4) can be layered atop an unreliable channel via a careful combination of **acknowledgments, timers, retransmissions, and sequence numbers**; this general theory applies at the link, network, transport, AND application layers, not just transport.
- **TCP** (§3.5) is a complex protocol combining connection management, reliable data transfer (a GBN/SR hybrid), flow control, and RTT estimation — all hidden behind a simple socket interface for the application.
- **Congestion control** (§3.6–3.7) is essential to prevent network gridlock. TCP's approach is **end-to-end AIMD**: additive increase (probing for bandwidth) + multiplicative decrease (backing off on loss) → the characteristic **saw-tooth**, which also drives connections toward roughly **fair** bandwidth sharing.
- **Beyond TCP/UDP** — newer/alternative transport protocols developed for cases neither fits perfectly: **DCCP** (low-overhead, message-oriented, congestion-controlled, UDP-like unreliable service), **QUIC** (Google's reliable, fast-setup, TCP-friendly protocol built atop UDP — ~half of Chrome→Google traffic used QUIC as of early 2015), **DCTCP** (TCP variant tuned for data centers, using ECN), **SCTP** (reliable, message-oriented, supports **multi-streaming** — independent streams multiplexed over one connection, with per-stream reliability isolation — plus multi-homing over multiple network paths), and **TFRC** (TCP-Friendly Rate Control — an equation-based congestion-control scheme, not a full transport protocol, aiming for a **smoother** sending rate than TCP's saw-tooth, well suited to IP telephony/streaming; pluggable into DCCP).
- This concludes the transport layer. Next: the **Network Layer** (Chapters 4 and 5) — data plane (Ch. 4: forwarding, IP addressing, routers) and control plane (Ch. 5: routing protocols, SDN).

---

## Key Formulas Quick Reference
- Sender utilization (stop-and-wait): $U_{sender} = \dfrac{L/R}{RTT + L/R}$
- RTT estimation: $EstimatedRTT = (1-\alpha)EstimatedRTT + \alpha \cdot SampleRTT$ (α = 0.125)
- RTT deviation: $DevRTT = (1-\beta)DevRTT + \beta |SampleRTT - EstimatedRTT|$ (β = 0.25)
- Timeout interval: $TimeoutInterval = EstimatedRTT + 4 \cdot DevRTT$
- Flow control: $rwnd = RcvBuffer - [LastByteRcvd - LastByteRead]$; sender keeps $LastByteSent - LastByteAcked \le rwnd$
- Congestion control constraint: $LastByteSent - LastByteAcked \le \min\{cwnd, rwnd\}$
- Sending rate ≈ $cwnd / RTT$
- TCP macroscopic throughput: average $= 0.75 \cdot \dfrac{W}{RTT}$, or in terms of loss rate: average $= 1.22 \cdot \dfrac{MSS}{RTT\sqrt{L}}$
- Well-known port range: **0–1023**; UDP socket ID = (dest IP, dest port) 2-tuple; TCP socket ID = (src IP, src port, dest IP, dest port) 4-tuple
- Typical values: **MSS ≈ 1460 bytes** (Ethernet/PPP MTU 1500B); TCP header = **20 bytes**; UDP header = **8 bytes**; TIME_WAIT ≈ 30s–2min
