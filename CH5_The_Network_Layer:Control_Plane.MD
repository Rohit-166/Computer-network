# Chapter 5 — The Network Layer: Control Plane
### (Kurose & Ross, *Computer Networking: A Top-Down Approach*, 7th Ed.)

---

## Chapter Overview

- Completes the network layer by covering the **control plane**: the network-wide logic that decides how datagrams are routed from source host to destination host, and how network-layer components are configured and managed.
- Roadmap:
  - **5.2** Routing algorithms (least-cost paths in a graph) — the basis for OSPF and BGP.
  - **5.3** **OSPF** — routing *inside* a single ISP's network (intra-AS).
  - **5.4** **BGP** — routing *between* ISPs (inter-AS); the "glue" that holds the Internet together.
  - **5.5** **SDN** control plane (controller, OpenFlow).
  - **5.6** **ICMP**; **5.7** **SNMP** (network management).

---

## 5.1 Introduction

- The **forwarding table** (destination-based forwarding) and **flow table** (generalized forwarding) are the links between the data plane and the control plane. This chapter is about how those tables are **computed, maintained, and installed**.
- Two approaches:
  1. **Per-router control** (Figure 5.1): a routing algorithm runs in **every router**; each router's routing component communicates with the routing components in other routers to compute its own forwarding table. Used on the Internet for decades; **OSPF and BGP** work this way.
  2. **Logically centralized control** (Figure 5.2): a distinct, typically remote **controller** computes and distributes the tables. Each router has a **control agent (CA)** that talks to the controller through a well-defined protocol.
     - The CA has **minimal functionality**: it only communicates with the controller and does what it is told.
     - **Key distinction**: CAs do **not** talk to each other and do **not** take part in computing the forwarding table.
     - "Logically centralized" = accessed as if it were one central service, though it is actually implemented on **several servers** for fault tolerance and scalability.
- Real-world SDN deployments mentioned: **Google B4** (WAN linking its data centers), **Microsoft SWAN**, **China Telecom / China Unicom** (within and between data centers), and AT&T's SDN-style capabilities.

---

## 5.2 Routing Algorithms

- **Goal**: find good paths (routes) from sender to receiver through the network of routers. "Good" usually means **least cost**, but real-world **policy** also matters (e.g., "router x of organization Y must not forward packets originating from organization Z's network").
- Whichever control-plane approach is used, there must always be a well-defined sequence of routers a packet crosses, so routing algorithms are fundamental.

### Graph model
- Network = graph **G = (N, E)**: **nodes** = routers, **edges** = physical links.
- Each edge has a **cost** c(x, y), which may reflect link length, speed, or monetary cost. If (x, y) is not an edge, c(x, y) = ∞.
- Only **undirected** graphs are considered here (c(x, y) = c(y, x)), but the algorithms extend to directed links.
- Node y is a **neighbor** of x if (x, y) is in E.
- A **path** is a sequence of nodes where consecutive pairs are edges; its **cost** is the sum of its edge costs. The **least-cost path** problem: find the path with minimum total cost between source and destination. If all costs are equal, least-cost = **shortest path** (fewest hops).
- Example (Figure 5.3): least-cost path u → w is (u, x, y, w) with cost 3. Between u and z there are 17 possible paths — nobody checks them all, which shows that a human solving this is effectively running a *centralized* algorithm.

### Three ways to classify routing algorithms
| Axis | Option A | Option B |
|---|---|---|
| **Information used** | **Centralized / global** — complete knowledge of connectivity and all link costs; called **link-state (LS)** algorithms | **Decentralized** — each node starts knowing only its directly attached link costs and iteratively exchanges info with neighbors; called **distance-vector (DV)** algorithms |
| **Rate of change** | **Static** — routes change very slowly, often via human intervention | **Dynamic** — routes change with traffic load or topology changes; more responsive but prone to **routing loops** and **route oscillation** |
| **Load sensitivity** | **Load-sensitive** — link costs reflect current congestion (early ARPAnet; ran into difficulties) | **Load-insensitive** — link cost does not reflect current congestion; **today's Internet protocols (RIP, OSPF, BGP) are load-insensitive** |

---

### 5.2.1 The Link-State (LS) Routing Algorithm

- Network topology and all link costs are known. In practice each node **broadcasts link-state packets** (identities and costs of its attached links) to all other nodes (via a **link-state broadcast algorithm**, as in OSPF). Result: every node has an **identical, complete view**, so each can compute the same least-cost paths.
- The algorithm is **Dijkstra's algorithm** (closely related to Prim's). It computes least-cost paths from **one source node u** to all other nodes. After the **k-th iteration**, least-cost paths are known to **k destination nodes**, and these are the k with the smallest costs.

**Notation**
- **D(v)**: cost of the current least-cost path from the source to v (as of this iteration).
- **p(v)**: previous node (a neighbor of v) on the current least-cost path from source to v.
- **N′**: set of nodes whose least-cost path from the source is definitively known.

**Algorithm (source node u)**
1. **Initialization**: N′ = {u}. For every node v: if v is a neighbor of u, D(v) = c(u, v); otherwise D(v) = ∞.
2. **Loop**:
   - Find a node w **not in N′** with the **minimum D(w)**.
   - Add w to N′.
   - For each neighbor v of w that is **not in N′**, update D(v) = min( D(v), D(w) + c(w, v) ). (New cost to v is either the old cost, or the known least cost to w plus the cost from w to v.)
3. Repeat **until N′ = N**.

**Worked example (Figure 5.3 / Table 5.1, source u)** — each row gives the table values at the end of the step:

| Step | N′ | D(v),p(v) | D(w),p(w) | D(x),p(x) | D(y),p(y) | D(z),p(z) |
|---|---|---|---|---|---|---|
| 0 | u | 2,u | 5,u | 1,u | ∞ | ∞ |
| 1 | ux | 2,u | 4,x | | 2,x | ∞ |
| 2 | uxy | 2,u | 3,y | | | 4,y |
| 3 | uxyv | | 3,y | | | 4,y |
| 4 | uxyvw | | | | | 4,y |
| 5 | uxyvwz | | | | | |

- Step-by-step: initially D(w) = 5 (the direct link) even though a cheaper path exists; costs to y and z are ∞ (not neighbors). Iteration 1 picks **x** (cost 1) and improves w to 4 (via x) and y to 2 (via x). Iteration 2 has a **tie** between v and y (both 2), broken arbitrarily — y is added.
- When it terminates, each node's **predecessor** along the least-cost path is known, so full paths can be reconstructed. The **forwarding table** at u stores, for each destination, the **next-hop node** on the least-cost path.

**Complexity**
- Searching for the minimum each iteration costs n, n−1, n−2, … → total n(n+1)/2 → worst case **O(n²)**. Using a **heap** reduces the minimum-finding cost to logarithmic, giving a lower complexity.

**Pathology: oscillations with congestion-sensitive costs** (Figure 5.5)
- If link cost = traffic carried (e.g., delay), costs are asymmetric. Example: nodes z and x each send 1 unit of traffic to w, and y sends e units, all destined to w.
- Initial routing: some nodes use the counterclockwise path. When LS reruns, y sees the clockwise path is cheaper (cost 1 vs 1+e) and switches; x also switches; now costs flip, and then x, y, z **all** see a zero-cost counterclockwise path and switch back; and so on forever.
- Possible fixes:
  - Make link costs independent of traffic — **unacceptable**, since avoiding congested links is a goal of routing.
  - Ensure **not all routers run LS at the same time**. But routers on the Internet can **self-synchronize** even if they start out of phase [Floyd 1994]. To avoid this, each router should **randomize** the time at which it sends link advertisements.

---

### 5.2.2 The Distance-Vector (DV) Routing Algorithm

- DV is **iterative, asynchronous, distributed, and self-terminating** (there is no stop signal; it just stops when no more information is exchanged).
  - **Distributed**: each node receives info from directly attached neighbors, computes, and sends results back to neighbors.
  - **Iterative**: continues until no more info is exchanged.
  - **Asynchronous**: nodes do not operate in lockstep.

**Bellman-Ford equation**
- Let d_x(y) = cost of the least-cost path from x to y. Then:

$$d_x(y) = \min_{v} \big( c(x,v) + d_v(y) \big) \quad (5.1)$$

  where the minimum is over all neighbors v of x. Intuition: after going from x to some neighbor v, take v's least-cost path to y; x picks the neighbor giving the smallest total.
- **Check** with source u and destination z in Figure 5.3: d_v(z)=5, d_x(z)=3, d_w(z)=3; c(u,v)=2, c(u,x)=1, c(u,w)=5 → d_u(z) = min(2+5, 5+3, 1+3) = **4**, matching Dijkstra.
- **Practical importance**: the neighbor v* that achieves the minimum is the **next hop** in x's forwarding table. It also suggests the neighbor-to-neighbor communication form used by DV.

**What each node x maintains**
1. The cost c(x, v) to each directly attached neighbor v.
2. Its own **distance vector** D_x = [D_x(y) : y in N] — x's estimates of its cost to all destinations.
3. The **distance vectors of each neighbor** v: D_v = [D_v(y) : y in N].

**Algorithm (at each node x)**
- **Initialization**: for all destinations y, D_x(y) = c(x, y) (∞ if y is not a neighbor). For each neighbor w, D_w(y) = ? (unknown). Send D_x to each neighbor.
- **Loop (forever)**: wait until a link cost to some neighbor changes **or** a distance vector arrives from some neighbor. Then, for each destination y, recompute D_x(y) = min over neighbors v of ( c(x, v) + D_v(y) ). If D_x(y) changed for any destination, **send the new D_x to all neighbors**.
- Provided nodes keep exchanging vectors asynchronously, each estimate D_x(y) **converges to the true d_x(y)** [Bertsekas 1991].
- DV-like algorithms are used in practice in **RIP and BGP**, ISO IDRP, Novell IPX, and the original ARPAnet.

**Worked example (Figure 5.6, three nodes x, y, z)**
- Link costs: c(x,y)=2, c(x,z)=7, c(y,z)=1.
- Node x's initial vector: D_x = [0, 2, 7]; rows for y and z start at ∞ (nothing received yet).
- After exchanging: D_x(y) = min(2+0, 7+1) = 2; D_x(z) = min(2+1, 7+0) = **3** (changed from 7, via y). So v*(y) = y and v*(z) = y.
- Only nodes whose vectors changed send updates; y's did not change in the next round so it sent nothing. The process continues until no updates are sent — the **quiescent state** — which lasts until a link cost changes.

**Link-cost changes and failures**
- **Good news travels fast** (cost decrease, Figure 5.7a: link y–x drops from 4 to 1): y updates and tells z; z's cost to x drops from 5 to 2 and z tells y; y's costs don't change so it stops. **Two iterations** to quiescence.
- **Bad news travels slowly** (cost increase, Figure 5.7b: link y–x rises from 4 to 60):
  - Before: D_y(x)=4, D_y(z)=1, D_z(y)=1, D_z(x)=5.
  - y recomputes D_y(x) = min(60+0, 1+5) = **6**, believing z can still reach x with cost 5 (z's route actually goes **through y**!). This creates a **routing loop** — packets for x bounce between y and z.
  - y tells z; z recomputes D_z(x) = min(50+0, 1+6) = 7; tells y; y gets 8; z gets 9; … The loop persists for **44 iterations**, until z's cost via y exceeds 50 (its direct link to x has cost 50). This is the **count-to-infinity problem** (it would be far worse if c(y,x) went from 4 to 10,000 and c(z,x) were 9,999).

**Poisoned reverse**
- Idea: if z routes through y to reach x, z **advertises D_z(x) = ∞ to y** (a "white lie"), for as long as z routes via y. Then y never tries to route to x via z.
- In the example, after the change to 60, y keeps routing directly (cost 60), z switches to its direct link (cost 50), then y updates to 51.
- **Limitation**: poisoned reverse does **not** solve count-to-infinity in general — loops involving **three or more nodes** are not detected.

**LS vs. DV comparison**
| Attribute | LS | DV |
|---|---|---|
| Information exchanged | Each node broadcasts costs of its **own directly connected links** to **all** nodes | Each node tells only its **neighbors**, but about its cost estimates to **all** nodes |
| Message complexity | **O(\|N\|·\|E\|)** messages; a link-cost change must be sent to all nodes | Messages between neighbors each iteration; convergence time varies; a cost change propagates only if it changes some least-cost path |
| Speed of convergence | **O(\|N\|²)** algorithm | Can converge slowly; can have **routing loops** while converging; **count-to-infinity** |
| Robustness | A router may advertise a wrong cost for its own links, or drop/corrupt LS broadcasts, but each node computes **only its own table** — route calculations are somewhat separated | A node can advertise wrong least-cost paths to **any/all destinations**, and errors get **diffused** through the network (e.g., a 1997 malfunctioning router at a small ISP gave bad routes to backbone routers and disconnected large parts of the Internet for hours) |
- Neither is an obvious winner; **both are used on the Internet**.

---

## 5.3 Intra-AS Routing in the Internet: OSPF

### Why the "one flat network of identical routers" model is too simple
1. **Scale**: with hundreds of millions of routers, storing routing info for all destinations, broadcasting updates, and iterating a DV algorithm would be prohibitive (it would never converge).
2. **Administrative autonomy**: an ISP wants to run its own routing algorithm and hide its internal organization from the outside.

### Autonomous Systems (ASs)
- Solution: organize routers into **autonomous systems**, each a group of routers under the **same administrative control**. Often an ISP's routers and links form one AS, though some ISPs split into multiple ASs (some tier-1 ISPs use one giant AS; others use tens).
- Each AS is identified by a globally unique **autonomous system number (ASN)** [RFC 1930], assigned by ICANN regional registries (like IP addresses).
- Routers in the same AS all run the **same routing algorithm** and know about each other. That algorithm is the **intra-autonomous system routing protocol**.

### Open Shortest Path First (OSPF)
- OSPF (and its close cousin **IS-IS**) is widely used for intra-AS routing. "**Open**" = the specification is publicly available (contrast with Cisco's EIGRP, proprietary for ~20 years). Current version: OSPFv2, **RFC 2328**.
- **OSPF is a link-state protocol**: it **floods** link-state information and uses **Dijkstra's algorithm**. Each router builds a complete topological map of the AS, then locally runs Dijkstra to get a **shortest-path tree to all subnets**, with itself as root.
- **Link costs are configured by the network administrator** (e.g., all set to 1 for minimum-hop routing, or inversely proportional to capacity to discourage low-bandwidth links). OSPF provides the **mechanism**, not the policy for setting weights.
- *Principles in Practice — setting OSPF weights*: in practice cause and effect can be **reversed**. Operators may choose link weights to achieve **traffic-engineering goals** (e.g., minimizing the maximum link utilization), since weights are their main "knob" for steering flows.
- **Broadcast behavior**: a router broadcasts link-state info to **all** routers in the AS (not just neighbors) whenever a link's state changes (cost or up/down), and also **periodically (at least every 30 minutes)** even without a change — adds robustness.
- OSPF messages are carried **directly over IP** with **upper-layer protocol number 89**. So OSPF must itself implement **reliable message transfer** and link-state broadcast. It uses **HELLO messages** to check that links are operational, and lets a router obtain a neighbor's network-wide link-state database.

**OSPF advances**
- **Security**: exchanges can be **authenticated**, so only trusted routers participate. By default packets are **not** authenticated (and could be forged). Two types: **simple** (same plaintext password on every router — weak) and **MD5** (shared secret key; router computes an MD5 hash of the packet plus the key and includes it; **sequence numbers** protect against replay attacks).
- **Multiple same-cost paths**: traffic may be split across several equal-cost paths.
- **Integrated unicast and multicast support**: **MOSPF** (RFC 1584) extends OSPF for multicast by adding a new link-state advertisement type.
- **Hierarchy within a single AS**: an AS can be divided into **areas**.
  - Each area runs its own OSPF LS algorithm; routers in an area broadcast link state only to routers in that area.
  - One or more **area border routers** in each area route packets outside the area.
  - Exactly **one area is the backbone area**, which routes traffic between the other areas; it contains all area border routers (and possibly other routers).
  - Inter-area routing: packet goes (intra-area) to an area border router → through the backbone to the area border router of the destination area → to the final destination.

---

## 5.4 Routing Among the ISPs: BGP

- To route across multiple ASs (e.g., from a phone in Timbuktu to a server in Silicon Valley), an **inter-AS routing protocol** is needed. Since it requires coordination among ASs, **all ASs run the same one: BGP (Border Gateway Protocol)** [RFC 4271].
- BGP is arguably the **most important Internet protocol** after IP — it glues together thousands of ISPs. It is **decentralized and asynchronous**, in the vein of distance-vector routing.

### 5.4.1 The Role of BGP
- For destinations **inside** the AS, forwarding-table entries come from the intra-AS protocol. For destinations **outside** the AS, **BGP** supplies them.
- In BGP, packets are routed to **CIDRized prefixes** (each representing a subnet or collection of subnets), not to individual addresses. Example: `138.16.68/22` covers **1,024 addresses**. A router's forwarding table has entries of the form **(x, I)**: prefix x and outgoing interface I.
- BGP gives each router a way to:
  1. **Obtain prefix reachability information from neighboring ASs** — each subnet can advertise "I exist and I am here." Without BGP, each subnet would be an isolated, unreachable island.
  2. **Determine the "best" routes to prefixes** — by running a local route-selection procedure based on **policy as well as reachability**.

### 5.4.2 Advertising BGP Route Information
- Example network (Figure 5.8): AS1, AS2, AS3; AS3 contains a subnet with prefix **x**.
- Routers are classified as:
  - **Gateway router**: on the edge of an AS, directly connected to one or more routers in other ASs (e.g., 1c in AS1).
  - **Internal router**: connects only to hosts and routers within its own AS (e.g., 1a, 1b, 1d).
- Conceptually: AS3 tells AS2 "AS3 x"; AS2 tells AS1 "AS2 AS3 x". Actually, it is **routers** (not ASs) that exchange messages.
- Pairs of routers exchange routing info over **semi-permanent TCP connections on port 179**. A TCP connection plus all BGP messages on it = a **BGP connection**.
  - **eBGP** (external BGP): connection **between ASs** — typically one per link directly connecting gateway routers in different ASs (e.g., 1c–2a and 2c–3a).
  - **iBGP** (internal BGP): session **between routers in the same AS**; commonly a mesh of TCP connections between every pair of internal routers. iBGP connections do **not always correspond to physical links**.
- **Propagation of prefix x**: 3a sends eBGP "AS3 x" to 2c → 2c sends iBGP "AS3 x" to all routers in AS2 (including 2a) → 2a sends eBGP "AS2 AS3 x" to 1c → 1c sends iBGP "AS2 AS3 x" to all routers in AS1.
- In Figure 5.10 an extra link (1d–3d) gives AS1 **two paths** to x: "AS2 AS3 x" via 1c, and "AS3 x" via 1d.

### 5.4.3 Determining the Best Routes
- A **route** = a prefix plus its **BGP attributes**. Two important attributes:
  - **AS-PATH**: list of ASs the advertisement has passed through. Each AS prepends its ASN as the prefix is passed along. Also used to **detect and reject looping advertisements** — if a router sees its own AS in the path, it rejects the advertisement.
  - **NEXT-HOP**: the **IP address of the router interface that begins the AS-PATH**. It is the critical link between inter-AS and intra-AS routing. (The NEXT-HOP is an interface of a router outside the AS, but on a subnet directly attached to the AS.)
- In the example, each AS1 router knows two routes written as (NEXT-HOP; AS-PATH; prefix): (left interface of 2a; AS2 AS3; x) and (left interface of 3d; AS3; x). Real BGP routes carry additional attributes.

**Hot-potato routing**
- Choose the route with the **least cost to the NEXT-HOP router** (cost determined by the **intra-AS** protocol, e.g., OSPF). Router 1b consults intra-AS info for the least-cost path to 2a and to 3d and picks the smaller; then finds the interface I on that path and adds **(x, I)** to its forwarding table.
- Both **BGP and the intra-AS protocol** are used to add an outside-AS prefix to a forwarding table.
- Idea: get the packet **out of the AS as cheaply as possible**, ignoring the cost of the rest of the path — a **selfish** algorithm (like passing a burning potato). Two routers in the same AS may choose **different AS paths** to the same prefix (1b via AS2, 1d directly to AS3).

**BGP route-selection algorithm** (applies rules in order until one route remains)
1. **Highest local preference** value (a **policy** attribute set by the AS administrator, or learned from another router in the AS).
2. **Shortest AS-PATH** (if this were the only rule, BGP would be a DV algorithm with AS hops as the metric).
3. **Closest NEXT-HOP router** (hot-potato routing).
4. If still tied, use **BGP identifiers** to choose.
- Applied to router 1b: rule 2 comes before rule 3, so BGP picks the route **bypassing AS2** (shorter AS-PATH) even though pure hot-potato would pick the AS2 route. So BGP is no longer purely selfish.
- BGP routing tables from tier-1 ISPs (see routeviews.org) can have **over half a million routes**.

### 5.4.4 IP-Anycast
- BGP is also used to implement **IP-anycast** [RFC 1546, RFC 7094], commonly used in **DNS**. Need: replicate content on servers in many places, and send each user to the **closest** one.
- Mechanism (CDN example, Figure 5.12): assign the **same IP address to all servers** and advertise it via standard BGP from each. A BGP router receiving multiple advertisements treats them as different paths to the same location, and uses its normal route selection (e.g., fewest AS hops) to pick the "closest." Clients are given the common IP and requests are routed to the closest server.
- In practice **CDNs generally avoid IP-anycast**, because BGP routing changes could send packets of the **same TCP connection** to different server instances.
- IP-anycast **is** extensively used by DNS: each of the **13 root server IP addresses** corresponds to many physical servers (some addresses have **100+**) worldwide; a query to such an address is routed to the nearest one.

### 5.4.5 Routing Policy
- AS policy can **trump** shortest AS-path and hot-potato — local preference is checked first.
- **Example (Figure 5.13)**: ASs A, B, C are backbone providers; W, X, Y are access ISPs. X is **multi-homed** (connected to two providers, B and C). Traffic entering an access ISP must be destined for it, and traffic leaving must originate in it (a **stub network** behavior).
  - **How X avoids carrying transit traffic between B and C**: X advertises to B and C that it has **no paths to any destination except itself**. Even if X knows a path like XCY, it does **not advertise** it to B.
  - **Provider B**: if B learns path AW from A, it advertises BAW to its **customer X**, but should it advertise BAW to C? Probably **not** — B would be carrying transit traffic between A and C for free.
  - **Rule of thumb among commercial ISPs**: traffic crossing an ISP's backbone must have a **source or destination (or both) in a customer network** of that ISP; otherwise it gets a "free ride." Peering agreements are negotiated between ISP pairs and are often confidential.
- **Why different inter-AS and intra-AS protocols?** (*Principles in Practice*)
  - **Policy**: among ASs, policy dominates (who may pass through whom, what transit to carry). Within an AS, one administrative authority, so policy matters far less.
  - **Scale**: critical for inter-AS routing; within an AS it's less of a concern (a too-large AS can be split into two, or into OSPF areas).
  - **Performance**: inter-AS routes are chosen by policy first, so quality is secondary (and there is no notion of cost other than AS hop count). Within an AS, routing can focus on performance.

### 5.4.6 Putting the Pieces Together: Obtaining Internet Presence
Scenario: a small company (e.g., Xanadu Inc.) with a Web server, mail server, and DNS server wants global reachability.
1. **Connectivity**: contract with a local ISP (e.g., DSL or leased line); a company **gateway router** connects to an ISP router.
2. **IP addresses**: the ISP gives an address range (e.g., a /24 = 256 addresses); assign addresses to the Web server, mail server, DNS server, gateway router, etc.
3. **Domain name**: contract with a **registrar** (e.g., xanadu.com); give the registrar your DNS server's IP, and it enters your DNS server's record into the **.com TLD servers**.
4. **DNS records**: in your DNS server, add entries mapping the Web server name (e.g., www.xanadu.com) and mail server to their IPs.
5. **Prefix advertisement via BGP**: every router in the Internet must know your prefix (or an aggregate). Your **local ISP advertises your prefix to its neighboring ISPs using BGP**, which propagate it onward until all Internet routers know it.

---

## 5.5 The SDN Control Plane

### 5.5.1 Four key characteristics of SDN architecture [Kreutz 2015]
1. **Flow-based forwarding**: forwarding by SDN switches can be based on **any number of header fields** at the transport, network, or link layer (OpenFlow 1.0: eleven fields) — unlike traditional routers that use only destination IP. The control plane computes, manages, and installs **flow table entries**.
2. **Separation of data plane and control plane**: data plane = simple, fast switches executing **match-plus-action** rules; control plane = servers and software that determine and manage the flow tables.
3. **Network control functions are external to data-plane switches**: software runs on servers distinct and **remote** from the switches. The control plane has two components: the **SDN controller** (a.k.a. **network operating system**) and a set of **network-control applications**. The controller maintains accurate **network state** (links, switches, hosts), provides it to applications, and gives them the means to monitor, program, and control devices. It is only **logically** centralized — implemented on several servers.
4. **A programmable network**: network-control applications are the "brains" and use the controller's APIs to control the data plane. Examples: a **routing** app (e.g., Dijkstra using controller-maintained state), an **access-control** app (which packets to block), a **load-balancing** app.

- SDN is a significant **unbundling** of network functionality (switches, controllers, applications can come from different vendors) vs. the old monolithic, vertically integrated, single-vendor router. Likened to the **mainframe → PC** evolution (separate hardware, OS, applications), hoped to spur similar innovation.

### 5.5.2 SDN Controller and Network-control Applications
Controller functionality is organized into **three layers** (Figure 5.15), described bottom-up:
1. **Communication layer** (**southbound interface**): protocol between the controller and controlled devices. Lets the controller push commands, and lets devices report local events (link up/down, device joined, heartbeat). **OpenFlow** is the example protocol, implemented in most controllers.
2. **Network-wide state-management layer**: holds up-to-date info on hosts, links, switches and other devices; flow-table counter values; possibly copies of the flow tables. This is the controller's network-wide "state."
3. **Interface to the network-control application layer** (**northbound interface**): API through which applications **read/write network state and flow tables** and **register for notifications** of state-change events. Often **REST** request-response interfaces.
- **Logically centralized but physically distributed**: for fault tolerance, availability, and performance. This raises distributed-systems issues (event ordering, consistency, consensus). Modern controllers (OpenDaylight, ONOS) emphasize this architecture.

### OpenFlow Protocol
- Operates between the controller and an SDN-controlled switch implementing the OpenFlow API. Runs over **TCP**, default port **6653**.
- **Controller → switch messages**:
  - **Configuration**: query and set switch configuration parameters.
  - **Modify-State**: add/delete/modify flow table entries; set port properties.
  - **Read-State**: collect statistics and counters from flow table and ports.
  - **Send-Packet**: have the switch send a specific packet (carried in the message) out a specified port.
- **Switch → controller messages**:
  - **Flow-Removed**: a flow entry was removed (e.g., timeout or modify-state).
  - **Port-status**: a change in port status.
  - **Packet-in**: a packet that matched no flow entry (or was explicitly directed to the controller as an action) is sent to the controller for processing.

### Case study — Google's B4 (software-defined global network)
- B4 is Google's private WAN interconnecting its data centers and server clusters, with a **Google-designed OpenFlow-based SDN control plane**.
- Drives WAN links at **near 70% utilization** over the long run (a 2–3× increase over typical utilization) and splits application flows across **multiple paths** by application priority and demand.
- Well suited to SDN because: (i) Google controls all devices from edge servers to core routers; (ii) the heaviest traffic is large **bulk data copies** that can yield to interactive traffic during congestion; (iii) only a **few dozen data centers**, so centralized control is feasible.
- Architecture: custom switches with an **OpenFlow Agent (OFA)** (like the control agent in Figure 5.2) connect to an **OpenFlow Controller (OFC)** in a **network control server (NCS)** over a separate **out-of-band** network. The OFC also keeps node/link status in a **Network Information Base (NIB)** (based on the **ONIX** controller). **BGP** (between data centers) and **IS-IS** (within a data center) are used. **Paxos** is used for hot replicas of NCS components. A **traffic-engineering application** sits above the NCS servers and provides network-wide bandwidth provisioning.

### 5.5.3 Data and Control Plane Interaction: An Example (Figure 5.16)
Setup: link between switches **s1 and s2** fails; shortest-path routing runs as a **separate application** (not in the switches); switches send link updates to the **controller**, not to each other. Steps:
1. s1 notifies the controller of the link failure using an OpenFlow **port-status** message.
2. The controller passes this to the **link-state manager**, which updates a link-state database.
3. The Dijkstra routing application, which **registered** for link-state changes, is notified.
4. The routing application gets updated link state from the link-state manager (possibly consulting other state components) and computes new least-cost paths.
5. It interacts with the **flow table manager**, which decides which flow tables must change.
6. The flow table manager uses OpenFlow to update entries at the affected switches — **s1** (now reaches s2 via s4), **s2** (now receives from s1 via s4), and **s4** (must forward s1's packets for s2).
   - Note: the book's setup paragraph says the rules at s1, s3, and s4 are affected (s2 unchanged), while step 6 names s1, s2, and s4. The text is inconsistent on this point; the key idea is the same either way: the controller updates **only the switches whose flow rules are affected**.
- Advantage: because the controller controls the flow tables, switching from least-cost routing to a more tailored scheme only requires **changing the control application software**, versus updating software in every router from multiple vendors under per-router control.

### 5.5.4 SDN: Past and Future
- Separation of data and control planes has roots well before the SDN buzz: arguments in **2004** [Feamster, Lakshman, RFC 3746]; a multi-controller framework for **ATM** networks [van der Merwe 1998]; the **Ethane** project [Casado 2007], which pioneered simple flow-based Ethernet switches with match-plus-action tables, a centralized controller managing flow admission and routing, and forwarding unmatched packets to the controller. A **300+ switch** Ethane network was operational in 2007; **Ethane evolved into OpenFlow**.
- **NFV (Network Functions Virtualization)**: a generalization of SDN aiming to replace sophisticated **middleboxes** (dedicated hardware, proprietary software, e.g., media caching) with commodity servers, switching, and storage.
- Research also aims to extend SDN from the **intra-AS** setting to **inter-AS**.

### Case studies — OpenDaylight and ONOS controllers
- Early SDN: one protocol (OpenFlow) and one controller (**NOX**). Now many controllers: proprietary (ONIX, Juniper Contrail, Google's) and open-source. **OpenDaylight** and **ONOS** are open-source, developed with the **Linux Foundation**.
- **OpenDaylight (ODL)**:
  - **Network-Service Applications**: determine forwarding and other services (firewalling, load balancing). Two interfaces: **external** apps use a **REST API over HTTP**; **internal** apps talk through the **Service Abstraction Layer (SAL)**.
  - **Basic Network-Service Functions** at the core (map to state management). The **SAL** is the controller's "nerve center": lets components/apps invoke each other's services and subscribe to events, and gives a uniform interface to underlying protocols including **OpenFlow and SNMP**. **OVSDB** manages data-center switching.
- **ONOS**: three layers like the canonical controller:
  - **Northbound abstractions/protocols**: unique **intent framework** — an app requests a high-level service (e.g., connect host A to host B, or forbid A–B communication) without knowing how it is carried out. State is available **synchronously** (query) or **asynchronously** (listener callbacks).
  - **Distributed core**: network state is kept across a set of servers, each running an identical ONOS copy; more servers = more capacity. Provides replication/coordination and the abstraction of logically centralized services.
  - **Southbound abstractions/protocols**: mask heterogeneity of hosts, links, switches and protocols so the core is device- and protocol-agnostic.

---

## 5.6 ICMP: The Internet Control Message Protocol

- **ICMP** (RFC 792) lets hosts and routers communicate **network-layer information** to each other, most typically **error reporting** (e.g., "Destination network unreachable" originates from a router sending an ICMP message to your host).
- Architecturally it sits **just above IP**: ICMP messages are **carried inside IP datagrams** as payload, like TCP/UDP segments. A datagram with upper-layer protocol number **1** is demultiplexed to ICMP.
- Each ICMP message has a **type** and **code**, and contains the **IP header and first 8 bytes** of the datagram that triggered it, so the sender can identify the offending datagram. ICMP is used for more than errors.
- **ping**: sends an ICMP **type 8, code 0** (echo request); the destination replies with **type 0, code 0** (echo reply). Most TCP/IP stacks support ping directly in the OS (not a process).
- **Source quench**: originally for congestion control (a congested router tells a host to slow down); **seldom used** now, since TCP has its own transport-layer congestion control.
- **Traceroute** is implemented with ICMP:
  - The source sends a series of ordinary IP datagrams, each carrying a **UDP segment to an unlikely port number**, with **TTL = 1, 2, 3, …** and a timer for each.
  - When the n-th datagram reaches the n-th router, the TTL expires; the router discards it and sends an ICMP **type 11, code 0** (TTL expired) message to the source, including the router's **name and IP address**. The source gets the **RTT** from its timer.
  - Stopping condition: eventually one datagram reaches the destination host, which replies with an ICMP **port unreachable (type 3, code 3)** because of the unlikely port; this tells the source to stop.
  - Standard Traceroute sends **three packets per TTL** value (so three results per hop).
  - The client must be able to ask the OS to send UDP datagrams with specific TTLs and be notified of arriving ICMP messages.
- **ICMPv6** (RFC 4443) reorganizes types/codes and adds new ones, including **"Packet Too Big"** and an **"unrecognized IPv6 options"** error code.

---

## 5.7 Network Management and SNMP

- **Definition of network management** [Saydam 1996]: the deployment, integration and coordination of hardware, software and human elements to **monitor, test, poll, configure, analyze, evaluate, and control** the network and element resources to meet real-time, operational performance and QoS requirements at reasonable cost. The chapter covers only the **architecture, protocols, and information base**, not the administrator's decision-making.

### 5.7.1 The Network Management Framework (Figure 5.20)
- **Managing server**: an application, usually with a human in the loop, at a centralized network management station in the **network operations center (NOC)**; controls collection, processing, analysis and display of management information and initiates control actions.
- **Managed device**: network equipment (host, router, switch, middlebox, modem, even a thermometer) containing one or more **managed objects** — actual hardware pieces (e.g., a NIC) and configuration parameters for hardware/software (e.g., OSPF).
- **MIB (Management Information Base)**: collects the information associated with managed objects — values the managing server can read and often set. Examples: counters (IP datagrams discarded due to header errors), descriptive info (DNS server software version), status (device functioning?), protocol-specific info (routing path). Objects are defined in the data description language **SMI (Structure of Management Information)**; related objects form **MIB modules** (nearly **400** RFC-defined modules as of mid-2015, plus many vendor-specific ones).
- **Network management agent**: a process in the managed device that communicates with the managing server and takes local actions under its command (similar to the routing/control agent in Figure 5.2).
- **Network management protocol**: runs between the managing server and managed devices; lets the server query device status and indirectly act via agents, and lets agents report exceptional events (failures, threshold violations). **The protocol itself does not manage the network** — it provides capabilities the administrator uses to manage it.

### 5.7.2 SNMP (Simple Network Management Protocol)
- **SNMPv2** (RFC 3416) is an **application-layer** protocol conveying management messages between a managing server and an agent. Two common usages:
  - **Request-response**: server sends a request to an agent, which acts and replies (query or **set** MIB object values).
  - **Trap messages**: unsolicited messages from an agent to the server to report an exceptional situation (e.g., an interface going up/down).
- **SNMPv2 defines seven PDU types** (Table 5.2):
| PDU | Direction | Purpose |
|---|---|---|
| **GetRequest** | manager → agent | get value of one or more MIB object instances |
| **GetNextRequest** | manager → agent | get value of next MIB object instance in a list or table |
| **GetBulkRequest** | manager → agent | get values in a large block (e.g., a large table), avoiding many separate requests |
| **InformRequest** | manager → manager | tell another managing entity about MIB values remote to it |
| **SetRequest** | manager → agent | set one or more MIB object values (agent replies with "noError" status to confirm) |
| **Response** | agent → manager (or manager → manager) | reply to Get*/Set/Inform requests |
| **SNMPv2-Trap** | agent → manager | inform manager of an exceptional event (asynchronous; **no response required**) |
- The MIB objects requested are listed in the **variable binding** part of the PDU; the agent's Response carries object identifiers and values.
- RFC 3418 well-known trap types: cold/warm start of a device, link up/down, loss of a neighbor, authentication failure.
- **Transport**: SNMP PDUs are typically carried in **UDP** datagrams (RFC 3417: UDP is the "preferred transport mapping"). Because UDP is unreliable, the **request ID** field lets the manager number requests; the agent copies it into the response, so the manager can **detect lost requests/replies**. Retransmission policy is **not mandated** — the manager just needs to act "responsibly" regarding frequency and duration.
- **Evolution**: three versions. **SNMPv3 ≈ SNMPv2 + security and administration capabilities**. Earlier lack of security meant SNMP was mostly used for **monitoring rather than control** (SetRequest rarely used in SNMPv1) — another example of security being added late.

---

## 5.8 Summary (Chapter Recap)

- Two broad control-plane approaches: **per-router control** (routing algorithm in every router) and **SDN** (logically centralized controller computes and distributes forwarding tables).
- Two fundamental least-cost-path algorithms: **link-state** and **distance-vector** — used in both per-router and SDN control.
- They underlie the two main Internet routing protocols: **OSPF** (intra-AS) and **BGP** (inter-AS).
- SDN control plane: **network-control applications**, the **controller**, and the **OpenFlow** protocol between controller and devices.
- Network management tools: **ICMP** and **SNMP**.
- Next: **Chapter 6 — The Link Layer and LANs**.

---

## Key Formulas and Facts Quick Reference
- **Bellman-Ford**: $d_x(y) = \min_{v} \big( c(x,v) + d_v(y) \big)$ (min over neighbors v of x)
- **Dijkstra update**: $D(v) = \min\big( D(v),\ D(w) + c(w,v) \big)$; complexity **O(n²)** (better with a heap)
- **LS** messages: **O(|N|·|E|)**; **DV**: neighbor-to-neighbor, slow convergence, count-to-infinity
- **Poisoned reverse** fixes 2-node loops only, not loops of 3+ nodes
- **OSPF**: link-state, runs directly over IP (protocol **89**), refresh at least every **30 min**, areas + backbone area, MD5 authentication
- **BGP**: runs over TCP **port 179**; **eBGP** (between ASs) and **iBGP** (within an AS); attributes **AS-PATH** and **NEXT-HOP**
- **BGP selection order**: (1) highest **local preference** → (2) shortest **AS-PATH** → (3) closest **NEXT-HOP** (hot potato) → (4) BGP identifier
- **OpenFlow**: over TCP, default port **6653**; key messages Modify-State, Read-State, Send-Packet, Packet-in, Port-status, Flow-Removed
- **ICMP**: IP protocol number **1**; ping = type 8/code 0 → reply type 0/code 0; TTL expired = type 11/code 0; port unreachable = type 3/code 3
- **SNMP**: usually over **UDP**; 7 PDU types; traps are asynchronous
