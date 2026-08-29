# Chapter 2 — Application Layer
### (Kurose & Ross, *Computer Networking: A Top-Down Approach*, 7th Ed.)

---

## 2.1 Principles of Network Applications

- Core of network application development = writing programs on **different end systems** that communicate over the network.
- You write software for **end systems only** — never for network-core devices (routers, link-layer switches), since those operate below the application layer (network layer and below).

### 2.1.1 Network Application Architectures
Application architecture (developer-designed) ≠ network architecture (fixed 5-layer Internet architecture). Two dominant paradigms:

**Client-Server Architecture**
- An **always-on server** services requests from many **clients**.
- Server has a **fixed, well-known IP address**; clients don't talk directly to each other (e.g., two browsers never communicate directly).
- Examples: Web, FTP, Telnet, e-mail.
- Popular services often use **data centers** (server farms) rather than a single server host — e.g., Google has 30–50 data centers worldwide.

**P2P (Peer-to-Peer) Architecture**
- Minimal/no reliance on dedicated infrastructure servers; direct communication between intermittently-connected **peers** (user-owned desktops/laptops).
- Examples: BitTorrent (file sharing), Xunlei (peer-assisted download), Skype (telephony/video).
- Some apps are **hybrid** — e.g., IM: servers track user IP addresses, but messages go directly peer-to-peer.
- Key advantage: **self-scalability** — each peer both consumes and adds service capacity (redistributes data). Also **cost-effective** (no massive server infrastructure needed).
- Challenges: **security, performance, reliability** due to decentralization.

### 2.1.2 Processes Communicating
- Strictly, **processes** (not "programs") communicate. On different hosts, processes communicate by exchanging **messages** across the network.
- **Client vs. server process definition**: In a given communication session, the process that **initiates** contact is the **client**; the process that **waits to be contacted** is the **server**. (Even in P2P, where a peer can be both client and server, this labeling applies per-session.)
- **Sockets**: the software interface between a process and the network — "**the process is a house, the socket is its door**." A process sends/receives messages through its socket.
  - Socket = the **API** between application and transport layers.
  - App developer controls the application side of the socket; has only limited control (transport protocol choice + a few parameters like buffer/segment size) over the transport side.
- **Addressing processes**: To send to a process, you need (1) the **host's IP address** and (2) a **port number** identifying the specific socket/process on that host (since a host may run many apps).
  - E.g., Web server → port 80; SMTP mail server → port 25. Full list at www.iana.org.

### 2.1.3 Transport Services Available to Applications
Four dimensions along which a transport-layer protocol's services can be classified:

1. **Reliable data transfer**: guarantees data arrives correctly & completely. Critical for e-mail, file transfer, Web, financial apps. **Loss-tolerant applications** (e.g., conversational audio/video) can accept some loss (a small glitch, not catastrophic).
2. **Throughput**: rate at which sender can deliver bits to receiver. **Bandwidth-sensitive applications** need a guaranteed minimum rate (e.g., VoIP at 32 kbps). **Elastic applications** (e-mail, file transfer, Web) can use whatever throughput is available — "more is always better."
3. **Timing**: e.g., guaranteeing data arrives within some delay bound (e.g., ≤100 ms). Important for interactive real-time apps: Internet telephony, virtual environments, teleconferencing, multiplayer games.
4. **Security**: encryption (confidentiality), data integrity, end-point authentication.

### 2.1.4 Transport Services Provided by the Internet
The Internet offers **two** transport protocols: **TCP** and **UDP**.

**TCP services:**
- **Connection-oriented**: client & server perform a **handshaking procedure** before app messages flow; establishes a **full-duplex** TCP connection; must be torn down when finished.
- **Reliable data transfer**: guarantees delivery of the exact byte stream, in order, no missing/duplicate bytes.
- **Congestion control**: throttles a sender when the network is congested — this benefits overall network welfare, not just the two communicating processes; also gives each connection its fair share of bandwidth.
- **Focus on Security — Securing TCP**: neither TCP nor UDP natively encrypts data. **SSL (Secure Sockets Layer)** is an enhancement of TCP (implemented at the application layer, not a third transport protocol) providing encryption, data integrity, and end-point authentication. App must include SSL library code on both client and server; app passes cleartext to the SSL socket, SSL encrypts and hands off to the TCP socket.

**UDP services:**
- **Connectionless** — no handshaking.
- **Unreliable data transfer** — no guarantee of delivery; messages may arrive out of order.
- **No congestion control** — sender can pump data at any rate (though actual throughput limited by link capacity/congestion).

**Services NOT provided by Internet transport protocols:** neither TCP nor UDP provides **throughput or timing guarantees**. Yet time-sensitive apps (e.g., Internet telephony) still work reasonably well in practice because they're designed to cope with this lack of guarantee (Ch. 9 covers design tricks).

- Typical protocol choices: e-mail, remote terminal access, Web, file transfer → **TCP** (need reliability). Internet telephony (e.g., Skype) → usually **UDP** (tolerates loss, avoids TCP's congestion control/overhead) but often falls back to TCP if UDP is blocked by firewalls.

### 2.1.5 Application-Layer Protocols
An application-layer protocol defines:
- The **types** of messages exchanged (e.g., request/response)
- The **syntax** of message types (fields, delineation)
- The **semantics** of fields (meaning of the info)
- **Rules** for when/how a process sends and responds to messages

- Some protocols are **public** (RFC-specified) — e.g., **HTTP** (RFC 2616). Others are **proprietary** — e.g., Skype.
- Important distinction: a **network application** (e.g., the Web) is NOT the same as its **application-layer protocol** (e.g., HTTP). HTTP is just one piece — the Web also includes HTML standard, browsers, servers, etc. Similarly, e-mail's main protocol is **SMTP** (RFC 5321), but the e-mail application includes mail servers, mail clients, message-format standards, etc.

### 2.1.6 Network Applications Covered in This Book
Chapter focuses on 5 apps in depth: **Web (HTTP)**, **e-mail**, **DNS**, **P2P file distribution**, **video streaming**. (Multimedia covered further in Ch. 9.)

---

## 2.2 The Web and HTTP

The Web (early 1990s) was the first application to bring the Internet to the general public — operates **on demand** (unlike broadcast TV/radio).

### 2.2.1 Overview of HTTP
- **HTTP (HyperText Transfer Protocol)**: RFC 1945, RFC 2616 (and HTTP/2: RFC 7540). Client program + server program exchange **HTTP messages**.
- **Web page/document** = consists of **objects** (HTML file, JPEG, Java applet, video clip, etc.), each addressable by a **URL**.
  - URL has two parts: **hostname** + **path name**. E.g., `http://www.someSchool.edu/someDepartment/picture.gif`
- **Browsers** implement the HTTP client side; **Web servers** (e.g., Apache, Microsoft IIS) implement the server side.
- HTTP uses **TCP** as its transport protocol (not UDP). Client initiates a TCP connection; both sides then use the socket interface.
  - Because TCP provides reliable delivery, HTTP need not worry about lost/reordered data — a benefit of **layered architecture**.
- **HTTP is a stateless protocol**: the server keeps **no state information** about clients between requests. If a client asks for the same object twice, the server just resends it (has no memory of prior requests). This simplifies server design and allows very high-performance servers.

### 2.2.2 Non-Persistent and Persistent Connections
Key design decision: should each request/response pair use a separate TCP connection (**non-persistent**), or should all requests/responses share one connection (**persistent**)? HTTP defaults to **persistent** but can be configured for non-persistent.

**Non-Persistent Connections — example walkthrough** (base HTML + 10 JPEGs, all on same server):
1. Client opens a TCP connection to server on **port 80** (default HTTP port).
2. Client sends HTTP request for the base HTML file via its socket.
3. Server retrieves the object, encapsulates it in an HTTP response, sends it.
4. Server tells TCP to **close** the connection (though TCP waits until it's sure client got the message intact).
5. Client receives response, connection terminates; client parses HTML, finds 10 JPEG references.
6. Steps 1–4 repeat for each of the 10 JPEGs → **11 TCP connections total** for this page.
- Browsers typically open **5–10 parallel TCP connections** by default (configurable) to speed things up; can be set to 1 for fully serial fetching.
- **RTT (Round-Trip Time)**: time for a small packet to travel client→server→client (includes propagation, queuing, processing delays).
- **Back-of-envelope response time** for one object: **TCP three-way handshake** takes **1 RTT** (client SYN, server SYN-ACK — first 2 parts of handshake = 1 RTT), then client piggybacks the HTTP request on the handshake's final ACK; the HTTP request/response round trip takes another **1 RTT**. **Total ≈ 2 RTTs + object transmission time.**

**Non-persistent connection shortcomings:**
1. New connection (with TCP buffers/variables) must be set up **per object** — burdens the server.
2. Each object suffers **2 RTTs** of delay (1 for connection setup + 1 for request/response).

**Persistent Connections (HTTP 1.1 default):**
- Server leaves the TCP connection **open** after sending a response; subsequent requests/responses (even for multiple Web pages on the same server) go over the **same connection**.
- Requests can be sent **back-to-back without waiting for replies** — called **pipelining**.
- Server closes idle connections after a configurable **timeout**.
- **HTTP/2** (RFC 7540) builds on this — allows multiple requests/replies to be **interleaved** on the same connection, plus **prioritization** of requests/replies.

### 2.2.3 HTTP Message Format
Two message types: **request** and **response**, both plain ASCII text.

**HTTP Request Message — example:**
```
GET /somedir/page.html HTTP/1.1
Host: www.someschool.edu
Connection: close
User-agent: Mozilla/5.0
Accept-language: fr
```
- **First line = request line**: has **method**, **URL**, **HTTP version** fields.
  - **Methods**: `GET` (most common — request an object), `POST`, `HEAD`, `PUT`, `DELETE`.
  - `POST`: used when submitting a form; data goes in the **entity body** (empty for GET).
    - Note: forms can also use GET, with input data appended to the URL (e.g., `www.somesite.com/animalsearch?monkeys&bananas`).
  - `HEAD`: like GET but server omits the requested object in its response — useful for debugging.
  - `PUT`: upload an object to a specific path on the server (Web publishing tools).
  - `DELETE`: delete an object on the server.
- **Subsequent lines = header lines**:
  - `Host:` — specifies the target host (needed by Web **proxy caches**).
  - `Connection: close` — tells server not to use persistent connections; close after responding.
  - `User-agent:` — identifies browser type (server can tailor content per browser).
  - `Accept-language:` — content negotiation (preferred language).

**HTTP Response Message — example:**
```
HTTP/1.1 200 OK
Connection: close
Date: Tue, 18 Aug 2015 15:44:04 GMT
Server: Apache/2.2.3 (CentOS)
Last-Modified: Tue, 18 Aug 2015 15:11:03 GMT
Content-Length: 6821
Content-Type: text/html

(data data data data data ...)
```
- **Status line**: protocol version, status code, status phrase.
- **Header lines**: `Connection:`, `Date:` (when response was generated — not when object was created), `Server:` (software), `Last-Modified:` (critical for **caching**), `Content-Length:` (bytes), `Content-Type:` (MIME type — officially determines object type, not the file extension).
- **Entity body**: the actual requested object/data.

**Common status codes:**
| Code | Meaning |
|---|---|
| **200 OK** | Request succeeded |
| **301 Moved Permanently** | Object moved; new URL in `Location:` header; client auto-retrieves it |
| **400 Bad Request** | Generic — server couldn't understand the request |
| **404 Not Found** | Requested document doesn't exist |
| **505 HTTP Version Not Supported** | Server doesn't support requested HTTP version |

- You can manually test HTTP via `telnet servername 80` and typing a `GET`/`HEAD` request.

### 2.2.4 User-Server Interaction: Cookies
Since HTTP is stateless, sites use **cookies** (RFC 6265) to track users. **4 components:**
1. A `Set-cookie:` header line in the HTTP **response**.
2. A `Cookie:` header line in subsequent HTTP **requests**.
3. A **cookie file** on the user's end system, managed by the browser.
4. A **back-end database** at the Web site.

**Example flow**: Susan visits Amazon for the first time → Amazon creates a unique ID (e.g., `1678`), sends `Set-cookie: 1678` in the response → browser appends this to its cookie file → every subsequent request to Amazon includes `Cookie: 1678` → Amazon's server can now track Susan's activity (pages visited, order, times) even without knowing her real name, unless she registers (then her identity gets linked to ID 1678). Used for **shopping carts**, personalized recommendations, "one-click" purchasing, session tracking for Web-based e-mail logins, etc.
- **Privacy controversy**: combined with account info, sites can learn a lot about users and potentially sell this data.

### 2.2.5 Web Caching
- A **Web cache** (a.k.a. **proxy server**) satisfies HTTP requests on behalf of an origin server; stores local copies of recently requested objects.
- Browser configured to send requests to the cache first:
  1. Browser opens TCP connection to the cache, sends HTTP request.
  2. Cache checks local storage — if it has the object, returns it directly.
  3. If not, cache opens a TCP connection to the **origin server**, requests the object, receives it.
  4. Cache stores a local copy AND forwards it to the client.
- A cache is **both server and client simultaneously** (server to the browser, client to the origin server).
- Typically installed/owned by an **ISP** (e.g., university campus network, residential ISP).

**Why caching helps (two reasons):**
1. Substantially **reduces response time** for the client (especially when the cache↔client link is much faster than the client↔origin-server path).
2. **Reduces traffic on the institution's access link** to the Internet — saving on bandwidth-upgrade costs, and reducing overall Internet traffic.

**Worked numerical example:**
- Institutional LAN: 100 Mbps. Access link to Internet: 15 Mbps. Average object = 1 Mbit; average request rate = 15 requests/sec. "Internet delay" (average) = 2 sec.
- Traffic intensity on LAN = (15 req/s × 1 Mbit) / 100 Mbps = **0.15** → negligible LAN delay.
- Traffic intensity on the **access link** = (15 × 1)/15 = **1** → delay → grows **without bound** (per §1.4.2) → total response time balloons to minutes — unacceptable.
- **Solution A**: upgrade access link to 100 Mbps → traffic intensity drops to 0.15 → total response time ≈ 2 sec (the Internet delay) — but costly.
- **Solution B**: install a **Web cache** with a hit rate of **0.4** (typical range: 0.2–0.7):
  - 40% of requests served almost instantly by cache (~10 ms, on fast LAN).
  - 60% still go to origin servers, but access-link traffic intensity drops to 0.6 (small delay, ~tens of ms).
  - Average delay ≈ 0.4×(0.01 s) + 0.6×(2.01 s) ≈ **1.2 seconds** — better than Solution A, and cheaper (caches often run on inexpensive PCs with public-domain software).
- **CDNs (Content Distribution Networks)**: companies (e.g., Akamai, Limelight, Google, Netflix) install many geographically distributed caches to further localize traffic (full detail in §2.6.3).

**Conditional GET** — solves the **staleness** problem (cached copy may be outdated):
- A request is a "conditional GET" if it (1) uses the `GET` method AND (2) includes an `If-Modified-Since:` header.
- Flow:
  1. Cache requests object; server responds with the object + a `Last-Modified:` date; cache stores both.
  2. Later, cache re-requests the object with `If-Modified-Since: <that date>`.
  3. If unmodified, server replies **`304 Not Modified`** with an **empty body** — saves bandwidth; cache then serves its own stored copy to the client.
  4. If modified, server sends the full new object (with `200 OK`).

---

## 2.3 Electronic Mail in the Internet

- **Asynchronous** communication medium (like postal mail — read/send at your convenience). Modern e-mail supports attachments, hyperlinks, HTML formatting, embedded photos.
- **Three major components**: **user agents**, **mail servers**, and **SMTP**.
  - **User agents** (e.g., Outlook, Apple Mail): compose, read, reply, forward, save messages.
  - **Mail servers**: form the core infrastructure; each user has a **mailbox** on a mail server. Sending server's message queue holds outgoing mail; if delivery fails, retries every ~30 min, gives up after several days and notifies the sender.
  - **SMTP**: the principal application-layer protocol, uses **TCP's** reliable data transfer to move mail from sender's mail server → recipient's mail server. Runs on **every** mail server (client-side when sending to others; server-side when receiving).

### 2.3.1 SMTP
- Defined in **RFC 5321**; much older than HTTP (original RFC from 1982).
- **Legacy limitation**: restricts message body to **7-bit ASCII** — binary/multimedia data must be **encoded** to ASCII before sending and decoded after. (HTTP has no such restriction.)
- **Basic scenario** (Alice → Bob):
  1. Alice's user agent sends the message to her mail server → placed in message queue.
  2. SMTP **client** (on Alice's mail server) opens a **TCP connection** to the SMTP **server** (on Bob's mail server).
  3. After SMTP handshaking, client sends the message over the TCP connection.
  4. Bob's SMTP server receives it, places it in Bob's mailbox.
  5. Bob reads it via his user agent.
- SMTP does **not** use intermediate mail servers — even across continents, the TCP connection is **direct** between sending and receiving mail servers. If the receiving server is down, the message stays queued at the **sender's** mail server (never handed off to some intermediate node).
- **Example SMTP transcript** (C = client, S = server; connection to port 25):
```
S: 220 hamburger.edu
C: HELO crepes.fr
S: 250 Hello crepes.fr, pleased to meet you
C: MAIL FROM: <alice@crepes.fr>
S: 250 alice@crepes.fr ... Sender ok
C: RCPT TO: <bob@hamburger.edu>
S: 250 bob@hamburger.edu ... Recipient ok
C: DATA
S: 354 Enter mail, end with "." on a line by itself
C: Do you like ketchup?
C: How about pickles?
C: .
S: 250 Message accepted for delivery
C: QUIT
S: 221 hamburger.edu closing connection
```
- Commands: `HELO`, `MAIL FROM`, `RCPT TO`, `DATA` (ends with a lone `.` on a line, i.e., `CRLF.CRLF`), `QUIT`.
- **SMTP uses persistent connections**: if there are multiple messages for the same receiving server, they can all be sent over the **same TCP connection**.

### 2.3.2 Comparison with HTTP
| | HTTP | SMTP |
|---|---|---|
| Direction | **Pull** protocol — receiver initiates the TCP connection to fetch data at its convenience | **Push** protocol — sender initiates the connection to push the file |
| Data format | No restriction | Message body restricted to **7-bit ASCII** |
| Multi-object handling | Each object gets its **own** HTTP response message | SMTP places **all objects of a message into one message** |
| Similarity | Both can use **persistent connections** | |

### 2.3.3 Mail Message Formats
- **RFC 5322** defines header lines for the message itself (distinct from SMTP handshaking commands, even though some words overlap, e.g. "from"/"to").
- Required: `From:`, `To:`. Optional: `Subject:`, etc.
- Header lines separated from the body by a **blank line (CRLF)**.
```
From: alice@crepes.fr
To: bob@hamburger.edu
Subject: Searching for the meaning of life.

(message body)
```

### 2.3.4 Mail Access Protocols
- Modern users run their user agent on a **local** device (PC/laptop/phone), not on the mail server host — so a **mail access protocol** is needed to pull mail from the server to the local client.
- **Two-step delivery**: Alice's UA → SMTP → Alice's mail server → SMTP → Bob's mail server. (Direct UA-to-recipient-server delivery isn't used because Alice's UA would have no recourse if Bob's server were unreachable; relaying through Alice's own server lets it retry.)
- **SMTP is a push protocol → can't be used by the recipient to *pull* mail** from their own mail server. Hence dedicated **mail access protocols**: **POP3**, **IMAP**, and **HTTP**.

**POP3 (Post Office Protocol v3)** — RFC 1939, simple, limited functionality.
- TCP connection to server, **port 110**. Three phases: **authorization** (username/password, sent in the clear), **transaction** (retrieve/mark-for-deletion), **update** (server deletes marked messages after client issues `quit`).
- Server replies: `+OK` (success, possibly with data) or `-ERR` (something wrong).
- **Authorization example**:
```
telnet mailServer 110
+OK POP3 server ready
user bob
+OK
pass hungry
+OK user successfully logged on
```
- **Transaction commands**: `list`, `retr`, `dele`, `quit`.
- Two modes:
  - **Download-and-delete**: messages removed from server after download — but if Bob reads mail from multiple devices (office PC, home PC, phone), each only gets the messages downloaded at that session — messages get **partitioned**, can't be reread from another device.
  - **Download-and-keep**: messages left on server — Bob can reread from multiple devices.
- POP3 is **stateless across sessions** (keeps state like "deleted" flags only *during* a session) — this simplicity is a design virtue.

**IMAP (Internet Message Access Protocol)** — RFC 3501, richer & more complex than POP3.
- Associates each message with a **folder** (new mail → `INBOX` by default); supports creating folders, moving messages between folders, and **searching** remote folders — all stored **on the server**, so accessible from any device (good for nomadic users).
- **Maintains user state across sessions** (folder structure, message-folder associations persist).
- Can fetch just **parts** of a message (e.g., just the header, or one part of a multipart MIME message) — useful over low-bandwidth links (avoids downloading large attachments unnecessarily).

**Web-based e-mail** (Hotmail pioneered this in the mid-1990s; now Gmail, Yahoo, etc.):
- The user agent is just an ordinary **Web browser**; communicates with the mailbox via **HTTP** (not POP3/IMAP) for both reading and sending.
- The mail server itself, however, still uses **SMTP** to send/receive mail to/from other mail servers.

---

## 2.4 DNS — The Internet's Directory Service

- Hosts can be identified by **hostname** (mnemonic, human-friendly, e.g., `www.google.com`) or by **IP address** (fixed 4-byte hierarchical numeric address routers use, e.g., `121.7.106.83`).
- **DNS (Domain Name System)** = (1) a **distributed database** implemented across a hierarchy of DNS servers, and (2) an **application-layer protocol** that lets hosts query this database.
- DNS servers commonly run **BIND** (Berkeley Internet Name Domain) software. **DNS runs over UDP, port 53**.

### 2.4.1 Services Provided by DNS
- **Primary service**: hostname → IP address translation, used internally by other app-layer protocols (HTTP, SMTP, etc.) — e.g., a browser must resolve `www.someschool.edu` to an IP before it can open a TCP connection to port 80.
  - This resolution adds **delay** to the overall app, mitigated by **caching**.
- **Host aliasing**: a host can have a complex **canonical hostname** (e.g., `relay1.west-coast.enterprise.com`) plus one or more mnemonic **alias** names (e.g., `enterprise.com`). DNS can return the canonical name + IP for a given alias.
- **Mail server aliasing**: lets a company's mail server have a simple, mnemonic alias (via the **MX** record) — and even share the same aliased name as the Web server.
- **Load distribution**: DNS supports **replicated servers** — a single canonical hostname can map to a **set of IP addresses**. DNS servers rotate the order of addresses in each reply (**DNS rotation**), spreading client load across replicas. Used for busy sites (e.g., CNN) and by CDNs like Akamai in more sophisticated ways.
- **Principles in Practice note**: unlike Web/e-mail/file-transfer, DNS is not an app users interact with directly — it's a **core Internet function** implemented via the client-server paradigm at the network's **edge** (consistent with the Internet's general design philosophy of pushing complexity to the edges).

### 2.4.2 Overview of How DNS Works
- App calls DNS client (e.g., via `gethostbyname()` on UNIX); DNS sends a **query** in a **UDP datagram to port 53**; after some delay (ms to seconds), receives a **reply** with the mapping.

**Why not a single centralized DNS server?**
- **Single point of failure** — if it crashes, so does much of the Internet.
- **Traffic volume** — one server can't handle queries from hundreds of millions of hosts.
- **Distant centralized database** — can't be geographically close to all clients everywhere (e.g., queries from Australia to a New York server → large delay).
- **Maintenance** — huge, constantly-updated database, difficult to manage centrally.
→ DNS is therefore **distributed by design**.

**Distributed, Hierarchical Database** — three main classes of DNS servers (+ a fourth type, local servers):
1. **Root DNS servers**: 400+ root servers worldwide, managed by 13 different organizations. Provide IP addresses of **TLD servers**.
2. **Top-Level Domain (TLD) servers**: one per TLD — `com`, `org`, `net`, `edu`, `gov`, and country codes (`uk`, `fr`, `jp`, etc.). E.g., Verisign manages `.com` TLD servers; Educause manages `.edu`. Provide IP addresses of **authoritative** servers.
3. **Authoritative DNS servers**: every organization with public hosts must provide DNS records for them — either self-hosted or paid-for at a provider. Universities/large companies typically run their own primary + secondary (backup) authoritative servers.
4. **Local DNS server** (a.k.a. **default name server**): NOT strictly part of the hierarchy, but central to the architecture. Each ISP has one; provided to hosts (often via DHCP). Acts as a **proxy**, forwarding queries into the hierarchy on the host's behalf. Typically **topologically close** to the host.

**Example DNS resolution chain** (`cse.nyu.edu` wants IP of `gaia.cs.umass.edu`, with `dns.nyu.edu` as local server and `dns.umass.edu` as authoritative):
1. Host → local DNS server (`dns.nyu.edu`) — **recursive** query.
2. Local server → a root server → returns list of `edu` TLD server IPs.
3. Local server → a TLD server → returns IP of `dns.umass.edu` (authoritative).
4. Local server → `dns.umass.edu` → returns the IP of `gaia.cs.umass.edu`.
→ **8 total DNS messages** (4 queries + 4 replies) in this basic example. (With an additional intermediate departmental server layer, this could rise to **10 messages**.)

- **Recursive vs. Iterative queries**: A **recursive** query asks the receiving server to obtain the answer **on the querier's behalf** (the querier gets just the final answer). An **iterative** query gets back either the answer or a **referral** to another server to ask next (querier does the further asking itself). In practice: query from **host → local DNS server is recursive**; the **remaining queries (local server → root/TLD/authoritative) are iterative**.

**DNS Caching**: critical for performance.
- Any DNS server, upon receiving a reply, can **cache** the mapping locally. Subsequent queries for the same hostname (even from other requesting hosts) can be answered directly from cache — bypassing further queries up the hierarchy.
- Cached entries are **discarded after a period** (often ~2 days) since mappings aren't permanent.
- Because of extensive caching, **root servers are bypassed for all but a tiny fraction of queries**.

### 2.4.3 DNS Records and Messages
DNS servers store **resource records (RRs)**, each a four-tuple: **(Name, Value, Type, TTL)**. TTL = time-to-live before the record should be purged from a cache.

| Type | Name | Value |
|---|---|---|
| **A** | hostname | IP address of the hostname (standard hostname→IP mapping) |
| **NS** | domain (e.g., `foo.com`) | hostname of an authoritative DNS server for that domain |
| **CNAME** | alias hostname | canonical (true) hostname |
| **MX** | alias hostname | canonical name of the corresponding **mail server** |

- MX records let a company use the **same alias** for both its mail server and, say, its Web server — querying for the MX record gets the mail server's canonical name; querying for CNAME gets the other server's.
- A server **authoritative** for a hostname holds a **Type A** record for it. A server that is **not** authoritative holds a **Type NS** record (pointing to the domain's authoritative server) plus a Type A record giving that authoritative server's own IP.

**DNS Message Format** — only **two** kinds of messages: query and reply, sharing the **same format**:
- **Header section** (12 bytes): 16-bit query **identifier** (echoed in the reply, to match replies to queries); flags — **query/reply flag** (0=query,1=reply), **authoritative flag** (set if the responding server is authoritative), **recursion-desired flag** (set by client wanting recursion), **recursion-available flag** (set if server supports it); plus four "number-of" count fields for the sections below.
- **Question section**: name being queried + query type (A, MX, etc.).
- **Answer section** (in replies): the resource record(s) for the queried name — can have **multiple RRs** (e.g., for replicated servers).
- **Authority section**: records of other authoritative servers.
- **Additional section**: helper records — e.g., for an MX query, this section supplies the Type A record giving the IP of the mail server's canonical name.

- **`nslookup`** utility: lets you manually send DNS queries to any DNS server (root/TLD/authoritative) and view the reply.

**Inserting records into DNS** (example: registering `networkutopia.com`):
- Register the domain with a **registrar** (a commercial entity accredited by **ICANN** — Internet Corporation for Assigned Names and Numbers) which verifies uniqueness, enters the domain into the DNS database, and charges a fee.
  - Before 1999, Network Solutions had a **monopoly** on `.com`/`.net`/`.org` registration; now many competing registrars exist.
- You supply your primary/secondary authoritative DNS server names & IPs; the registrar inserts NS and A records into the TLD `com` servers, e.g.:
```
(networkutopia.com, dns1.networkutopia.com, NS)
(dns1.networkutopia.com, 212.212.212.1, A)
```
- You must also separately ensure your own authoritative server has the A record for your Web server and MX record for your mail server.
- An **UPDATE** option (RFC 2136, RFC 3007) allows dynamic (rather than only static/config-file-based) DNS database updates.

**Focus on Security — DNS Vulnerabilities:**
- **DDoS bandwidth-flooding** against DNS root servers: an actual large-scale attack occurred on **October 21, 2002**, using a botnet to flood all 13 root server IPs with ICMP ping messages — caused **minimal damage** because (a) many root servers had packet filters blocking ICMP pings, and (b) most local DNS servers **cache TLD server IPs**, bypassing root servers anyway.
- A **more effective** DDoS target would be TLD servers (e.g., all `.com` TLD servers) — harder to filter and not as easily bypassed as root servers, though local caching still mitigates severity somewhat.
- **Man-in-the-middle attack**: attacker intercepts queries and returns bogus replies.
- **DNS poisoning/cache poisoning**: attacker sends bogus replies to a DNS server, tricking it into caching false records (can redirect users to malicious sites). These attacks are **difficult to implement** (require intercepting packets or overwhelming servers).
- Overall: DNS has proven **surprisingly robust** — no attack has yet successfully taken down the DNS service at large.

**Full example — Alice in Australia visits `www.networkutopia.com`:**
1. Her host queries her local DNS server.
2. Local server → root (if TLD not cached) → TLD `com` server → gets NS + A records for `dns1.networkutopia.com`.
3. Local server queries `212.212.212.1` for the Type A record of `www.networkutopia.com`.
4. Gets back the Web server's IP (e.g., `212.212.71.4`), passed to Alice's host.
5. Alice's browser opens a TCP connection to that IP and sends the HTTP request.

---

## 2.5 Peer-to-Peer File Distribution

Unlike client-server apps (Web, e-mail, DNS) which rely heavily on always-on servers, **P2P** apps minimize reliance on infrastructure — peers communicate directly, redistributing data among themselves.

### Scalability of P2P Architectures — Quantitative Model
Setup: server upload rate **u_s**; peer *i*'s upload/download rates **u_i**, **d_i**; file size **F** bits; **N** peers want the file. Assume the Internet **core** has abundant bandwidth (bottlenecks are only in **access links**).

**Client-Server distribution time (D_cs):**
- Server must send the file **N** times → time ≥ **NF/u_s**.
- Slowest peer (min download rate **d_min**) needs at least **F/d_min** seconds.
$$D_{cs} = \max\left\{\frac{NF}{u_s}, \frac{F}{d_{min}}\right\} \quad (2.1)$$
- For large N, **D_cs ≈ NF/u_s** — grows **linearly** with N. (E.g., N growing 1,000× → distribution time grows 1,000×.)

**P2P distribution time (D_P2P):**
- Server needs to inject each bit only **once**: ≥ **F/u_s**.
- Slowest peer still needs ≥ **F/d_min**.
- Total system upload capacity = **u_s + u_1 + ... + u_N**; must deliver NF bits total → ≥ **NF/(u_s + Σu_i)**.
$$D_{P2P} = \max\left\{\frac{F}{u_s}, \frac{F}{d_{min}}, \frac{NF}{u_s + \sum_{i=1}^{N} u_i}\right\} \quad (2.3)$$
- **Key result**: as N grows, the third term's numerator (NF) and denominator (includes ΣU_i, which grows with N) both scale with N — so P2P distribution time does **NOT** grow unboundedly like client-server does. This is **self-scalability**: peers add capacity as they add demand.
- Example (F/u=1 hr, u_s=10u, d_min≥u_s, all peers same rate u): client-server time grows linearly without bound as N increases; P2P time stays **below 1 hour for any N**.

### BitTorrent
The dominant P2P file-distribution protocol (originally by Bram Cohen).
- **Torrent**: the collection of all peers distributing a particular file.
- Peers download/upload equal-size **chunks** (typical chunk size: **256 KB**).
- New peer starts with zero chunks, accumulates them over time while also uploading to others. Can leave once complete (selfishly) or stay and keep seeding (altruistically); can also leave mid-download and rejoin later.
- **Tracker**: an infrastructure node per torrent that tracks which peers are currently participating.
  - New peer (Alice) registers with the tracker → tracker returns a **random subset** (e.g., 50) of participating peers' IPs.
  - Alice attempts TCP connections to all of them → those she connects to become her **neighboring peers** (set fluctuates over time as peers join/leave).
- **Which chunks to request — "rarest first"**: Alice periodically asks neighbors which chunks they have, then requests the chunks that are **rarest** among her neighbors first — spreads rare chunks faster, roughly equalizing chunk availability across the torrent.
- **Trading algorithm ("tit-for-tat")** — who gets served:
  - Alice continuously measures the rate at which each neighbor sends her data; she picks the **top 4** fastest-feeding neighbors and reciprocates by sending them chunks. Recalculated every **10 seconds**. These 4 are called **unchoked**.
  - Every **30 seconds**, she also randomly picks **one additional neighbor** to send chunks to regardless of rate — this peer is **optimistically unchoked**. This lets new peers get a foothold (something to trade) and lets better partnerships be discovered — if the randomly-chosen peer reciprocates well, it may end up in the top-4 list.
  - All other neighbors are **choked** (receive nothing).
  - This incentive scheme is called **tit-for-tat**. It has been shown to be circumventable in some ways, but is credited as critical to BitTorrent's success — without it, most users would likely have been **free-riders**.
- Other (not detailed) mechanisms: pieces/mini-chunks, pipelining, random-first-selection, endgame mode, anti-snubbing.
- **DHT (Distributed Hash Table)**: a simple, distributed database whose records are spread across the peers of a P2P system — used e.g. within BitTorrent; widely researched.

---

## 2.6 Video Streaming and Content Distribution Networks

Streaming prerecorded video dominates residential ISP traffic today (Netflix ~37%, YouTube ~16% of North American residential downstream traffic in 2015).

### 2.6.1 Internet Video
- Video = sequence of images displayed at a constant rate (e.g., 24–30 fps); each image = array of pixels; heavily **compressible**, trading quality for bit rate.
- Compressed rates: ~100 kbps (low quality) to 3+ Mbps (HD streaming); 4K envisions >10 Mbps.
- Most important performance metric: **average end-to-end throughput** — must be ≥ the video's encoded bit rate for continuous playout.
- Sites often prepare **multiple encoded versions** at different bit rates (e.g., 300 kbps / 1 Mbps / 3 Mbps) so users/clients pick based on available bandwidth.

### 2.6.2 HTTP Streaming and DASH
- **HTTP streaming**: video is just an ordinary file on an HTTP server; client does a `GET`, server sends the file as fast as network conditions allow; client buffers received bytes and starts playback once buffer exceeds a threshold.
  - **Shortcoming**: all clients get the **same encoding**, regardless of hugely varying available bandwidth (across clients, and over time for the same client).
- **DASH (Dynamic Adaptive Streaming over HTTP)**: video encoded into **multiple versions** at different bit rates; client requests small (~few-second) **chunks** one at a time via HTTP GET, dynamically picking chunks from a higher- or lower-rate version depending on currently measured available bandwidth.
  - Server hosts a **manifest file** listing each version's URL + bit rate; client fetches this first.
  - Client runs a **rate-determination algorithm**, continuously measuring received bandwidth to decide the next chunk's quality/version.
  - Especially valuable for mobile users whose bandwidth fluctuates as they move between base stations.

### 2.6.3 Content Distribution Networks (CDNs)
- Naive approach — one giant data center streaming to the whole world — has **3 major problems**:
  1. **Long/many-hop paths** to distant clients → any single slow/congested link along the way bottlenecks the whole stream → freezing/rebuffering.
  2. **Wasted bandwidth**: popular videos sent repeatedly over the same links; provider pays its ISP repeatedly for the same bytes.
  3. **Single point of failure**.
- **CDN**: manages servers at multiple **geographically distributed** locations, stores copies of content, and directs each user to the best-performing location.
  - **Private CDN** (owned by the content provider, e.g., Google's CDN for YouTube) vs. **third-party CDN** (serves multiple providers, e.g., Akamai, Limelight, Level-3).

**Two server-placement philosophies:**
| Philosophy | Description | Pros/Cons |
|---|---|---|
| **Enter Deep** (pioneered by Akamai) | Deploy clusters **deep inside access ISPs** worldwide (~1,700 Akamai locations) | Gets very close to users → better delay/throughput; harder to maintain/manage (highly distributed) |
| **Bring Home** (e.g., Limelight) | Build larger clusters at fewer sites, typically at **IXPs** | Lower maintenance overhead; possibly higher delay/lower throughput to end users |

- Many CDNs use a **pull strategy** (like Web caching, §2.2.5) rather than pre-pushing every video to every cluster: if a cluster doesn't have a requested video, it fetches it (from a central repo or another cluster), caches it locally, and streams simultaneously. Least-frequently-requested videos get evicted when storage fills.

**Case Study — Google's Network Infrastructure** (three tiers):
1. **~14 "mega data centers"** (8 North America, 4 Europe, 2 Asia), each ~100,000 servers — serve **dynamic/personalized** content (search results, Gmail).
2. **~50 clusters at IXPs** worldwide, each ~100–500 servers — serve **static** content (e.g., YouTube videos).
3. **Hundreds of "enter-deep" clusters** inside access ISPs, typically tens of servers in a single rack — perform **TCP splitting** and serve static content (e.g., static portions of search-result pages).
- All tiers connected via **Google's own private network**. E.g., for a search query: local ISP → nearby enter-deep cache (returns static content + forwards query over Google's private network) → mega data center (personalized results). For YouTube: video from a bring-home cache; surrounding page from enter-deep cache; ads from data centers.

**CDN Operation — DNS-based redirection** (example: NetCinema uses third-party KingCDN):
1. User visits NetCinema's page.
2. Clicking a video link (`http://video.netcinema.com/6Y7B23V`) triggers a DNS query for `video.netcinema.com`.
3. NetCinema's authoritative DNS server recognizes the "video" string and hands off by returning a **hostname in KingCDN's domain** (e.g., `a1105.kingcdn.com`) instead of an IP.
4. The query then enters **KingCDN's own DNS infrastructure**, which resolves it to the IP of an actual KingCDN content server — **this is where cluster selection happens**.
5. That IP is passed back to the user's LDNS, then to the user's host.
6. Client opens a direct TCP connection to that server and issues an HTTP GET (using DASH if applicable, first fetching the manifest file).

**Cluster selection strategies** (proprietary, vary by CDN):
- **Geographically closest cluster**: uses commercial geo-location databases (e.g., Quova, MaxMind) to map the client's LDNS IP to a location, then picks the nearest cluster "as the bird flies." Works reasonably well but has flaws: geographic proximity ≠ network-path proximity; some users are configured to use **remotely located** LDNS servers; ignores real-time delay/bandwidth variation (always picks the same cluster for a given client).
- **Real-time measurement-based selection**: clusters periodically probe (e.g., via ping or DNS queries) LDNSs worldwide to gauge current delay/loss — more responsive to real conditions, but many LDNSs are configured to **not respond** to such probes.

### 2.6.4 Case Studies: Netflix, YouTube, Kankan

**Netflix** (37% of North American residential downstream traffic, 2015):
- Two major components: **Amazon cloud** + Netflix's **own private CDN**.
- **Amazon cloud handles**: the entire website/backend (registration, billing, browsing, recommendations); **content ingestion** (receiving studio master videos); **content processing** (transcoding into many formats/bit rates for different devices, supporting DASH); **uploading** finished versions to the Netflix CDN.
- Netflix's own CDN: server racks installed in **50+ IXP locations** and hundreds of **ISP locations** (Netflix provides free racks to partner ISPs). Each server: several 10 Gbps Ethernet ports, 100+ TB storage. IXP racks often hold the **entire** video library (with all DASH versions); smaller local ISP racks may hold just the most popular videos.
- Netflix uses **push caching** (not pull) — content is pushed to CDN servers during **off-peak hours**, based on daily-updated popularity data — NOT dynamically on cache misses.
- Client interaction: Amazon-cloud software determines the best CDN server for a request (preferring a local ISP rack if it has the movie, else a nearby IXP), then **directly tells the client which server to use** (no DNS redirect needed, unlike third-party CDN setups) along with a manifest file. Client and CDN server then use a **proprietary DASH variant** with **~4-second chunks**, using HTTP byte-range requests.

**YouTube** (300 hours uploaded/minute, billions of daily views; started April 2005, acquired by Google Nov 2006):
- Uses Google's own **private CDN**, with clusters at hundreds of IXP/ISP locations plus Google's data centers.
- Unlike Netflix, YouTube uses **pull caching** and **DNS redirect** (like the generic CDN description above).
- Cluster-selection: usually picks the cluster with the **lowest RTT** to the client, but sometimes redirects to a more distant cluster for **load balancing**.
- Uses **HTTP streaming** with a small number of fixed bit-rate versions — **NOT** DASH; user must **manually** select quality. Uses HTTP **byte-range requests** to limit transmitted data (avoiding wasted bandwidth from repositioning/early termination).
- Uploaded videos also travel client→server over **HTTP**; Google's data centers do all the transcoding into multiple YouTube formats/bit rates.

**Kankan** (Xunlei, since 2011, tens of millions of monthly users):
- Uses **P2P delivery** instead of (or combined with) pure client-server/CDN delivery, to cut infrastructure/bandwidth costs.
- High-level mechanism similar to BitTorrent: contacts a **tracker** to find peers holding the video, requests chunks in **parallel** from multiple peers — but, unlike BitTorrent's rarest-first, **prioritizes chunks needed for imminent playback** (to support continuous streaming).
- Has migrated to a **hybrid CDN-P2P** model: a few hundred Kankan-owned servers within China seed content, primarily aiding the **start-up phase**. Client requests initial content from CDN servers **in parallel** with requesting from peers; once P2P throughput is sufficient, streaming shifts to **P2P-only**; if P2P traffic becomes insufficient, it falls back to CDN — balancing short start-up delay against minimal infrastructure cost.

---

## 2.7 Socket Programming: Creating Network Applications

- A network application = a **client program** + a **server program**, each creating a process that communicates by reading/writing **sockets**.
- Two categories:
  - **Open/standardized** apps — implement a published protocol (e.g., an RFC), so independently-written clients/servers can **interoperate** (e.g., Chrome talking to Apache).
  - **Proprietary** apps — protocol not published; one developer/team controls both sides; won't interoperate with independently-written implementations.
- First major design decision: **TCP or UDP**? If implementing a standard protocol, must use its designated **well-known port**; proprietary apps must **avoid** well-known ports.
- Examples given in **Python 3** (chosen for concise, clear exposure of core socket concepts — Java/C/C++ equally usable).

### 2.7.1 Socket Programming with UDP
- Sending process must explicitly **attach a destination address** (IP + port) to each packet before sending (UDP is connectionless — no persistent "pipe").
- **Destination address** = destination host's IP + destination socket's **port number** (needed since a host may run many apps/sockets). Source address (sender's IP + port) is attached **automatically by the OS**, not by application code.

**Example app used throughout**: client reads a line from the keyboard, sends it to server; server **uppercases** it and sends it back; client displays the result.

- **`UDPClient.py`** — key lines:
  - `from socket import *` — imports the socket module.
  - `clientSocket = socket(AF_INET, SOCK_DGRAM)` — creates a **UDP** socket (`AF_INET`=IPv4, `SOCK_DGRAM`=UDP). No port number is bound explicitly — OS assigns one.
  - `clientSocket.sendto(message.encode(), (serverName, serverPort))` — attaches destination address and sends.
  - `modifiedMessage, serverAddress = clientSocket.recvfrom(2048)` — receives a reply (buffer size 2048); also gets the sender's address, though the client doesn't need it here.
  - `clientSocket.close()`.
- **`UDPServer.py`** — key differences:
  - `serverSocket.bind(('', serverPort))` — explicitly **binds** the socket to a known port (e.g., 12000) so incoming packets addressed to that port reach this socket.
  - Runs an infinite `while True:` loop: `recvfrom()` to get an incoming packet + client's address, process (`.upper()`), then `sendto()` the reply back to that **client's address** (acts like a return address).

### 2.7.2 Socket Programming with TCP
- TCP is **connection-oriented** — client and server must **handshake** to establish a TCP connection before any data flows.
- Server needs a **special "welcoming" socket** that listens for initial contact from any client (`serverSocket` in the example). When a client connects, the server creates a **new, dedicated socket** for that specific client (`connectionSocket`) — distinct from the welcoming socket, which stays open to accept further clients.
- With TCP, the app just drops bytes into the connection — **no explicit destination address per send** (unlike UDP), since the connection itself already ties the two endpoints together.

- **`TCPClient.py`** — key lines:
  - `clientSocket = socket(AF_INET, SOCK_STREAM)` — `SOCK_STREAM` = TCP socket.
  - `clientSocket.connect((serverName, serverPort))` — initiates the **three-way handshake**, establishing the TCP connection (handshake itself is invisible to the application).
  - `clientSocket.send(sentence.encode())` — sends bytes directly into the established connection (no address needed).
  - `clientSocket.recv(1024)` — receives the reply.
  - `clientSocket.close()` — closes socket & connection; this causes TCP to send a closing message to the server side.
- **`TCPServer.py`** — key lines:
  - `serverSocket.bind(('', serverPort))` then `serverSocket.listen(1)` — sets up the **welcoming socket**, listening for connection requests (parameter = max queued connections).
  - `connectionSocket, addr = serverSocket.accept()` — blocks until a client connects; upon connecting, creates the **new dedicated socket** (`connectionSocket`) for that client, completing the handshake.
  - Reads via `connectionSocket.recv()`, processes, replies via `connectionSocket.send()`, then `connectionSocket.close()` — but `serverSocket` (the welcoming door) **stays open** for the next client.
- TCP guarantees bytes arrive **in order and without loss**, unlike UDP.

---

## 2.8 Summary

- Studied the **client-server** and **P2P** architectures underlying most Internet applications.
- Deep-dived into four major application-layer protocols: **HTTP** (Web), **SMTP/POP3/IMAP** (e-mail), **DNS** (naming), plus **P2P file distribution (BitTorrent)** and **video streaming/CDNs**.
- Learned the **socket API** as the interface between applications and the transport layer, and implemented simple **UDP** and **TCP** client-server programs in Python.
- Reinforced the general definition of "protocol" (message formats, order, and the actions taken on sending/receiving) with concrete, detailed examples.
- This completes the **top layer** of the top-down journey through the protocol stack — next: the **Transport Layer** (Chapter 3), examining how TCP and UDP actually work "under the hood."

---

## Key Formulas Quick Reference
- Non-persistent HTTP, one object: response time ≈ **2×RTT + object transmission time**
- Web-cache average delay example: weighted average = (hit rate × cache-hit delay) + (miss rate × miss delay)
- P2P vs client-server distribution time:
  - $D_{cs} = \max\left\{\dfrac{NF}{u_s}, \dfrac{F}{d_{\min}}\right\}$
  - $D_{P2P} = \max\left\{ \dfrac{F}{u_s},\dfrac{F}{d_{\min}},\dfrac{NF}{u_s + \sum_{i=1}^{N} u_i} \right\}$

- DNS resource record 4-tuple: **(Name, Value, Type, TTL)**
- Well-known ports: **HTTP = 80**, **SMTP = 25**, **POP3 = 110**, **DNS = 53** (UDP), TCP welcoming-socket example port used in text = **12000** (arbitrary app-chosen)
