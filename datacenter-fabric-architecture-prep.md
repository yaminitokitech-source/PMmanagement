# Datacenter Fabric Architecture — Interview Prep Reference

**Target role context:** TPM, Google AI & Infrastructure / Global Network Delivery. You are not being hired to design the fabric — you are being hired to *deliver* it. The technical fluency below exists so you can sit in an engineering review as a participant rather than a passenger, and so you can translate fabric properties into schedule risk, bill-of-materials decisions, and executive-level impact statements.

**How to use this document:** Parts 1–8 build sequentially. Each part ends with a **Say it out loud** block — a 60–90 second spoken answer. The final section maps every concept to a TPM framing.

---

## Table of contents

1. [Why fabrics exist — north-south, east-west, and the monolith](#part-1)
2. [Clos topology and spine-leaf](#part-2)
3. [Oversubscription and non-blocking design](#part-3)
4. [Hashing from first principles](#part-4)
5. [BGP inside the switch](#part-5)
6. [ECMP under the hood](#part-6)
7. [Limitations of ECMP](#part-7)
8. [TPM translation layer](#part-8)

---

<a name="part-1"></a>

# Part 1 — Why Fabrics Exist

## 1.1 What "datacenter fabric architecture" means

A **fabric** is a network design in which many small, identical switches are wired together so that collectively they behave like one enormous switch with uniform, predictable bandwidth between any two attached machines.

The word "fabric" is literal — it describes a woven mesh where every thread crosses every other thread, rather than a hierarchy where traffic funnels up and down a trunk.

The defining promise of a fabric: **any server can talk to any other server at predictable bandwidth and latency, regardless of where either one physically sits.**

That promise is what everything else in this document exists to deliver.

## 1.2 The old model — the three-tier tree

Before fabrics, enterprise datacenters used a three-tier hierarchy:

```
                    ┌──────────────┐
                    │  Core switch │   Big, expensive chassis
                    └──────┬───────┘
              ┌────────────┴────────────┐
       ┌──────┴──────┐           ┌──────┴──────┐
       │ Aggregation │           │ Aggregation │
       └──────┬──────┘           └──────┬──────┘
        ┌─────┴─────┐             ┌─────┴─────┐
     ┌──┴──┐     ┌──┴──┐       ┌──┴──┐     ┌──┴──┐
     │Access│    │Access│      │Access│    │Access│
     └──┬──┘     └──┬──┘       └──┬──┘     └──┬──┘
        │           │              │           │
     servers     servers        servers     servers
```

Three properties of this design matter:

- **Links got fatter as you climbed.** The core was the widest point because all traffic was assumed to pass through it on its way out of the building.
- **You scaled up, not out.** More capacity meant a bigger core chassis — a forklift upgrade and a procurement event.
- **Redundant links were disabled.** Spanning Tree Protocol blocked every link not in the computed tree (see §6.6). You paid for redundancy and got zero extra bandwidth from it.

This design was correct *for the traffic pattern it was built for*. That pattern then changed completely.

## 1.3 North-south vs east-west traffic

These two terms come from the orientation of the diagram above — up and down the page versus across it.

### North-south traffic

**Definition:** traffic that crosses the boundary of the datacenter. Something outside is talking to something inside, or vice versa.

**Examples:**

| Example | What's happening |
|---|---|
| You load `google.com` in a browser | Your laptop → the internet → a Google front-end server |
| A phone app fetches your photo library | Mobile device → API gateway inside the DC |
| A user uploads a video to YouTube | External client → ingest service |
| A customer downloads a report from a SaaS product | Internal service → external user |
| Traffic leaving one DC bound for another DC | Crosses the DC boundary, so still north-south from this building's perspective |

**Mental test:** if one end of the conversation is outside the building, it's north-south.

### East-west traffic

**Definition:** traffic that stays inside the datacenter. Server to server, rack to rack, within the same building.

**Examples:**

| Example | What's happening |
|---|---|
| The front-end service calls the authentication service | Two machines in the same DC, possibly different rows |
| A database write is replicated to two additional copies | One write becomes three internal transfers |
| A disk fails and the cluster rebuilds the lost data | Surviving replicas copy data to a new machine — traffic no user triggered |
| A MapReduce/Spark shuffle | Every mapper sends a slice to every reducer — literal all-to-all |
| GPUs exchanging gradients during a training step | Thousands of machines exchanging simultaneously |
| A scheduler live-migrates a VM to a different host | Memory state copied machine to machine |

**Mental test:** if both ends are inside the building, it's east-west.

### The proportion

By the mid-2010s, intra-datacenter traffic commonly ran to roughly three-quarters or more of all datacenter traffic. Facebook reported its internal traffic doubling at intervals shorter than a year — growing far faster than the external traffic it served.

**The single most important fact in this document:** the tree was built for north-south, and traffic became overwhelmingly east-west.

## 1.4 What changed — the monolith and its opposite

### The monolith

A **monolithic application** is one program running on one machine (or a few identical copies of it). All its internal components — authentication, business logic, data access — live in the same process. When component A calls component B, that is a **function call**: a few nanoseconds across the memory bus, generating **zero network packets**, and it cannot fail.

In a monolith, the only traffic on the wire is the user talking to the machine. North-south by construction.

### The opposite

The opposite is **distributed architecture** — the family that includes service-oriented architecture, microservices, and scale-out data tiers. "Microservices" undersells it, because the network shift began well before that word existed. The more precise framing:

> **The industry moved from scale-up to scale-out.**

Decompose one monolith into fifty services on fifty machines, and every former function call becomes a network call. The work didn't change. The **location** of the work changed, and every internal boundary that used to be free became a packet.

### The five forces that drove it

**1. Scale-up ran out of road.** Growth used to mean buying a bigger machine. That stopped working on two fronts: the cost curve for large systems is superlinear — twice the machine costs far more than twice the money — and around 2005 Dennard scaling ended, so single-thread CPU performance plateaued. Google's founding insight was that a thousand cheap unreliable machines beat one expensive reliable one, if the software handles failure. Accept that and everything is distributed by construction.

**2. The data outgrew the box.** A dataset too large for one machine gets **sharded** across many. To survive disk failure it gets **replicated**, typically three ways. Replication is pure east-west traffic with no external counterpart at all. Worse, when a disk dies the cluster **re-replicates** from surviving copies — a burst of internal traffic no user triggered and no user sees.

**3. Distributed data processing.** MapReduce (2004), then Hadoop and Spark. The **shuffle** phase between map and reduce is a literal all-to-all exchange — every mapper sends a slice to every reducer. The first workload architecturally incapable of being north-south.

**4. Virtualization, then containers and schedulers.** This one matters most for fabric design and is the least appreciated. Once VMs could migrate and Borg/Kubernetes could place a workload on any machine with free capacity, you lost the ability to predict *where* anything runs. You can no longer engineer bandwidth around known traffic locality, because there is no locality — the scheduler will cheerfully put the two chattiest services in different rows. **Uniform any-to-any bandwidth stopped being a luxury and became a prerequisite for the scheduler to be free to do its job.**

**5. Organizational decomposition.** Amazon's 2002 internal mandate that all teams expose functionality only through service interfaces is the canonical example. Teams wanted to deploy independently, which meant hard service boundaries, which meant network boundaries. Conway's law with a wire attached.

### The fan-out picture

```
                        ┌──────────────┐
                        │ User request │        1 north-south request
                        └──────┬───────┘
                               ▼
                     ┌───────────────────┐
                     │  Front-end service │
                     └─────────┬─────────┘
          ┌──────────┬─────────┼─────────┬──────────┐
          ▼          ▼         ▼         ▼          ▼
      ┌──────┐  ┌────────┐ ┌───────┐ ┌───────┐ ┌─────────┐
      │ Auth │  │Profile │ │Ranking│ │  Ads  │ │Inventory│   ← east-west
      └───┬──┘  └────┬───┘ └───┬───┘ └───┬───┘ └────┬────┘
          ▼          ▼         ▼         ▼          ▼
     ╔══════════════════════════════════════════════════╗
     ║   Sharded data tier — every write replicated 3×   ║   ← more east-west
     ╚══════════════════════════════════════════════════╝

     One request in.  Dozens to hundreds of internal calls out.
```

**This is the ratio that broke the tree.** One unit of north-south generates one to three orders of magnitude more east-west.

## 1.5 RPC — the mechanism that turns decomposition into traffic

**RPC = Remote Procedure Call.**

The idea is in the name: a procedure (function) call that executes on a *remote* machine but is written to look like an ordinary local one. Your code calls `getUserProfile(id)` and receives a return value. Hidden inside a generated stub, the request was serialized into bytes, sent over the network, executed on another server, and the response sent back.

**RPC is the direct causal link between distributed architecture and east-west traffic.** Every service boundary that a team drew on a whiteboard becomes an RPC, and every RPC is packets crossing the datacenter floor.

### Why the abstraction is both the point and the problem

| | Local function call | Remote procedure call |
|---|---|---|
| Latency | Nanoseconds | Microseconds to milliseconds |
| Can fail? | No | Yes — timeout, drop, connection reset |
| Retries | Meaningless | Necessary, and generate *more* traffic |
| Network cost | Zero | Two packets minimum, often many |
| Looks like | `getUser(id)` | `getUser(id)` |

The last row is the whole issue. The code looks identical; the failure modes are completely different. Making remote calls *look* local is what made microservices practical to write — and it is also why a network problem shows up to an application team as "the app is slow" with no indication that the network is involved.

## 1.6 gRPC from scratch

**gRPC** is Google's open-source RPC framework, released in 2015. Its internal ancestor at Google is **Stubby**. It is the dominant way modern services talk to each other, and therefore it is what most east-west traffic actually *is* on the wire.

### The three problems any RPC framework must solve

1. **Contract** — how do both sides agree on what the function is called, what arguments it takes, and what it returns?
2. **Serialization** — how do you turn a structured object into bytes and back?
3. **Transport** — how do the bytes get there reliably?

gRPC answers each one specifically.

### Problem 1: the contract — Protocol Buffers as an IDL

You write an **interface definition** in a `.proto` file. This is the single source of truth for both client and server:

```protobuf
syntax = "proto3";

package profile;

service ProfileService {
  rpc GetUserProfile (GetUserProfileRequest) returns (UserProfile);
}

message GetUserProfileRequest {
  int64 user_id = 1;
}

message UserProfile {
  int64  user_id      = 1;
  string display_name = 2;
  string avatar_url   = 3;
  bool   is_verified  = 4;
}
```

A compiler (`protoc`) reads that file and **generates code** in whatever languages you need — Go, Java, Python, C++, and many more. It produces:

- a **client stub** with a real `GetUserProfile()` method you call like any function
- a **server skeleton** with a method you implement

Neither side writes serialization code. Neither side can drift from the contract without regenerating. Language choice becomes irrelevant — a Go service and a Java service interoperate because both were generated from the same `.proto`.

### Problem 2: serialization — Protocol Buffers on the wire

Note the numbers in the message definition: `user_id = 1`, `display_name = 2`. Those are **field tags**, and they are what travels on the wire — **not the field names**.

Compare the same payload:

```
JSON  (~90 bytes):
{"user_id":80421,"display_name":"nchauhan","avatar_url":"/a/8f.png","is_verified":true}

Protobuf (~30 bytes):
[tag 1, varint 80421][tag 2, len 8, "nchauhan"][tag 3, len 9, "/a/8f.png"][tag 4, 1]
```

Three consequences:

- **Smaller on the wire** — often 3–10× smaller than the equivalent JSON, which matters enormously when you're doing hundreds of calls per user request.
- **Faster to parse** — binary fields with known types, no string tokenizing.
- **Forward and backward compatible** — add a new field with a new tag number and old clients simply skip what they don't recognize. This is what lets services deploy independently, which was the organizational goal in the first place.

### Problem 3: transport — HTTP/2

gRPC runs over **HTTP/2**, and this choice has direct consequences for the fabric.

| HTTP/2 feature | What it does | Network consequence |
|---|---|---|
| **Multiplexing** | Many concurrent requests share one TCP connection as independent streams | One long-lived connection carries a lot of traffic |
| **Header compression (HPACK)** | Repeated headers sent once | Less overhead per call |
| **Binary framing** | Structured frames, not text parsing | Efficient, and enables streaming |
| **Bidirectional streaming** | Either side can send a sequence of messages | Supports long-lived flows |

gRPC supports four call patterns, worth knowing by name: **unary** (one request, one response), **server streaming**, **client streaming**, and **bidirectional streaming**.

### The connection back to the fabric — this is the part that matters

HTTP/2 multiplexing means one **long-lived TCP connection** carries many requests between a pair of services.

One TCP connection = **one 5-tuple** (see §4.3). One 5-tuple = **one hash result** = **one physical path** through the fabric.

So a gRPC connection between two busy services **pins all of its traffic to a single link** no matter how much it carries. This is a milder version of the elephant-flow problem that dominates AI training fabrics (§7.1) — and it is why "we use gRPC" is a networking fact, not just a software fact.

A secondary effect: gRPC propagates **deadlines** and encourages **retries**. A retry storm during an incident multiplies east-west traffic exactly when the fabric is already stressed.

## 1.7 Where AI takes this

Distributed training is the endpoint of the arc.

During a multi-day training run, north-south traffic is roughly **zero** — the dataset is already local, nobody outside is talking to the cluster. **All** of the traffic is accelerators exchanging gradients with each other. It is simultaneously the most bandwidth-hungry and the most latency-sensitive east-west pattern ever built, and — unlike everything before it — it is **synchronous**.

The full arc:

```
Monolith            →  east-west doesn't exist
Services / RPC      →  east-west appears, dominates by volume
Schedulers / K8s    →  east-west becomes unpredictable in placement
Distributed training →  east-west becomes bulk, synchronous, and the only traffic
```

> ### Say it out loud — *"Why did datacenter networks change?"*
>
> "Old datacenter networks were three-tier trees, and that was the right design when traffic was north-south — a user outside the building talking to a server inside it. What broke that was decomposition. When applications were monoliths, a call from one component to another was a function call: nanoseconds, zero packets. Once you split that into services across machines, every one of those calls became an RPC on the wire. So a single user request now fans out into hundreds of internal calls, plus three-way replication on every write, plus shuffle traffic and rebuild traffic. East-west went from nothing to roughly three-quarters of all datacenter traffic. And the scheduler made it worse in a useful way — once Borg or Kubernetes can place a workload anywhere, you can't engineer around traffic locality, because there isn't any. That's what forced the move to a fabric: you need uniform any-to-any bandwidth so placement is free. AI training is the extreme version — during a training run north-south is essentially zero and every byte is accelerators talking to each other."

---

<a name="part-2"></a>

# Part 2 — Clos Topology and Spine-Leaf

## 2.1 The core idea of Clos

**Charles Clos, 1953, Bell Labs.** Originally designed for telephone circuit switching, not computers.

The problem Clos solved: you want a switch with N inputs and N outputs where any input can reach any output. Building that as one physical crossbar requires N² crosspoints — the cost explodes as N grows.

**Clos's insight:** you can build an equivalent-behaving large switch out of **multiple stages of much smaller switches**, wired in a specific pattern, at dramatically lower total cost — and prove mathematically that it's non-blocking under defined conditions.

Translated to datacenters:

> **Instead of one enormous expensive switch, use many small cheap identical switches wired so they collectively behave like one enormous switch.**

This is why merchant silicon mattered. Once off-the-shelf switch chips (Broadcom and others) were good enough, buying a thousand of them beat buying one custom chassis — in cost, in failure blast radius, and in the ability to grow incrementally.

## 2.2 The tiers

### Two-tier Clos (spine-leaf) — the standard unit

```
        ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
        │ Spine 1 │ │ Spine 2 │ │ Spine 3 │ │ Spine 4 │     SPINE TIER
        └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘
             │  ╲    ╱   │  ╲    ╱   │  ╲    ╱   │
             │   ╲  ╱    │   ╲  ╱    │   ╲  ╱    │
             │    ╲╱     │    ╲╱     │    ╲╱     │       full mesh:
             │    ╱╲     │    ╱╲     │    ╱╲     │       every leaf to
             │   ╱  ╲    │   ╱  ╲    │   ╱  ╲    │       every spine
        ┌────┴────┐ ┌────┴────┐ ┌────┴────┐ ┌────┴────┐
        │ Leaf 1  │ │ Leaf 2  │ │ Leaf 3  │ │ Leaf 4  │     LEAF TIER (ToR)
        └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘
             │           │           │           │
         [ rack 1 ]  [ rack 2 ]  [ rack 3 ]  [ rack 4 ]
```

**Leaf tier (also called Top-of-Rack, ToR):**
- The only switch a server ever plugs into
- Physically bolted into the top few rack units of every server rack — that is literally where the name comes from
- A 1RU or 2RU fixed-configuration "pizza box"
- **Downlinks** are short: direct-attach copper (DAC) or active optical cable, a meter or two inside the rack
- **Uplinks** are long: optics running to the spine row, tens to hundreds of metres
- A datacenter has thousands of them — one or two per rack

**Spine tier:**
- Lives in a dedicated network row or network rack, **not** in a compute rack
- **No servers attach to it.** Every port faces leaves
- Its only job is to forward traffic between leaves as fast as possible
- Deliberately "dumb" — policy lives at the leaf

### Three-tier Clos — scaling past one pod

A single spine-leaf pod is bounded by spine port count. To go larger you add a third tier:

```
              ┌──────────────┐  ┌──────────────┐
              │ Superspine 1 │  │ Superspine 2 │        SUPERSPINE
              └──────┬───────┘  └───────┬──────┘        (spine block)
          ┌──────────┴──────┬───────────┴─────────┐
          │                 │                     │
    ╔═════╧═════╗     ╔═════╧═════╗         ╔═════╧═════╗
    ║  POD 1    ║     ║  POD 2    ║         ║  POD 3    ║   aggregation
    ║ spine+leaf║     ║ spine+leaf║         ║ spine+leaf║   blocks
    ╚═══════════╝     ╚═══════════╝         ╚═══════════╝
```

Terminology varies by vendor: the third tier is called **superspine**, **core**, or **spine block**. Each two-tier unit is a **pod** or **aggregation block**.

## 2.3 How east-west traffic actually flows

This is the mechanical answer to "how do the tiers communicate east-west."

**Same rack** (server in rack 1 → another server in rack 1):
```
Server A → Leaf 1 → Server B          1 hop. Never touches the spine.
```

**Different rack, same pod** (rack 1 → rack 3):
```
Server A → Leaf 1 → [ Spine N ] → Leaf 3 → Server B      Always exactly 2 hops.
```
Which spine? Whichever the hash picks (§4, §6). All four are equivalent.

**Different pod** (pod 1 → pod 2):
```
Server A → Leaf → Spine → Superspine → Spine → Leaf → Server B     4 hops.
```

**The critical property:** within a tier boundary, every server-to-server path has the **identical hop count**. There is no "near rack" and "far rack."

## 2.4 The properties of spine-leaf architecture

These five are the answer to "what are the properties of spine-leaf." Know them as a list.

### Property 1 — Uniform path length (predictable latency)

Every leaf-to-leaf conversation crosses exactly the same number of hops. Latency becomes a property of the **fabric**, not of where the scheduler happened to place your workload.

This is what makes workload placement free. It's the network-side answer to force #4 in §1.4.

### Property 2 — Every link carries traffic, all the time (ECMP)

Leaf 1 has four valid, equal-cost paths to Leaf 3. Rather than disabling three of them the way spanning tree did, all four are used simultaneously via ECMP (§6).

This also forces a design choice: you need a control plane that can express "many equal paths," which is why fabrics run **Layer 3 routing down to the ToR** — BGP or an SDN controller — rather than Layer 2 (§5).

### Property 3 — Scale-out, not scale-up

| Need | Action |
|---|---|
| More east-west bandwidth | Add a spine + one uplink per leaf |
| More racks / servers | Add leaves |
| More than one pod's worth | Add a superspine tier |

You buy commodity boxes in quantity. Compare to the tree, where adding capacity meant a bigger core chassis — a procurement event rather than a cabling job.

### Property 4 — Bounded, graceful failure blast radius

Losing 1 spine of 4 costs **25% of fabric capacity**, not connectivity. Losing 1 core chassis of 2 in the old design cost 50%.

This is the arithmetic behind "N+1 spine redundancy" in a build plan, and it's why **spine count is a resilience decision as much as a bandwidth decision.**

### Property 5 — Layer 3 to the rack

Because routing runs down to the ToR, loops aren't catastrophic: IP packets carry a **TTL**, so a looped packet dies after 64 hops instead of circulating forever. Ethernet frames have no TTL, which is the whole reason spanning tree had to exist.

The industry didn't just replace STP with ECMP — it **moved routing down into the rack so that STP was no longer needed.**

## 2.5 Which devices are leaves and which are spines

**In a Clos fabric, a switch is a leaf or a spine because of how it is cabled, not because of what it is.** The same box, same silicon, same SKU can be either.

```
   SAME 64-PORT SWITCH, WIRED TWO WAYS

   ┌───────────────────────────┐      ┌───────────────────────────┐
   │  Uplinks: 32 → spines     │      │  Uplinks: none, or        │
   │                           │      │  → superspine             │
   └─────────────┬─────────────┘      └─────────────┬─────────────┘
                 ▼                                  ▼
   ┌───────────────────────────┐      ┌───────────────────────────┐
   │     WIRED AS A LEAF       │      │     WIRED AS A SPINE      │
   └─────────────┬─────────────┘      └─────────────┬─────────────┘
                 ▼                                  ▼
   ┌───────────────────────────┐      ┌───────────────────────────┐
   │  Downlinks: 32 → servers  │      │  Downlinks: 64 → leaves   │
   └───────────────────────────┘      └───────────────────────────┘
```

### Where real hardware differences do show up

Role is cabling, but SKUs aren't always identical. Three differentiators:

**Buffer depth.** Leaves absorb **incast** — many senders converging on one receiver — so they benefit from deep buffers. This splits the ASIC market: Broadcom **Tomahawk** (shallow buffer, maximum port count, cheap per port) is the classic spine chip; **Trident** and **Jericho** (deeper buffer, richer features, larger tables) appear at the leaf. Current generation Tomahawk 5 is 51.2 Tb/s — 64 ports of 800G on one chip.

**Feature and table size.** The leaf is where policy lives: VXLAN/EVPN encapsulation, ACLs, QoS marking, tenant isolation. It needs bigger forwarding and ACL tables. The spine just does ECMP forwarding.

**Optics reach.** Spine-facing ports need longer-reach, more expensive optics than server-facing ports. **This is a bill-of-materials fact you will actually manage** — spine optics are long-lead items on a different line than the DAC cables inside a rack.

### Vendor examples (for recognition, not memorization)

| Tier | Typical products |
|---|---|
| Leaf / ToR | Arista 7050/7060, Cisco Nexus 9300, Juniper QFX5120 |
| Spine (chassis style) | Arista 7800, Cisco Nexus 9500, Juniper QFX10000 |
| Spine (hyperscale) | The same fixed pizza boxes as the leaves, just many of them |

## 2.6 Google's own naming — Jupiter

Google builds its switches in-house from merchant silicon rather than buying Arista or Cisco, and its terminology differs from industry standard. Worth knowing so you aren't caught flat.

| Industry term | Google term |
|---|---|
| Leaf / ToR | ToR (same) |
| Spine tier within a pod | **Aggregation block** — a pod of switches acting as one logical unit |
| Superspine / core | **Spine block** |

**Original Jupiter (~2015, "Jupiter Rising")** rested on three pillars:
1. **SDN** — a logically centralized control plane programming thousands of switch chips
2. **Clos topology** — small switches collectively acting as one non-blocking switch
3. **Merchant silicon** — off-the-shelf chips rather than proprietary hardware

**The problem that forced evolution:** Clos requires a **uniform-speed spine**. As server links climbed 40G → 100G → 200G → 400G, upgrading meant near-total rewiring rather than incremental improvement.

**Current Jupiter ("Jupiter Evolving," SIGCOMM 2022):**
- **Optical Circuit Switching (OCS)** via custom hardware called **Apollo** — MEMS mirrors physically redirect light between fiber ports. No packet parsing at all. Data-rate and wavelength agnostic, so it survives generational upgrades because it's part of the physical building infrastructure, not the electrical switching layer.
- **The fixed spine layer was largely eliminated**, replaced by a direct mesh dynamically reconfigured in real time to match application communication patterns — "**topology engineering**."
- Control plane: the **Orion** SDN controller coordinates topology reconfiguration plus traffic engineering.
- Enables **"hitless" upgrades** — capacity added or removed with zero application-visible impact. Directly relevant to adding capacity to a live ML cluster without disrupting an in-progress training run.

**Published scale:** 6+ Pb/s aggregate bandwidth (vs 1+ Pb/s in 2015), 30,000+ servers per datacenter, per-server links up to 400 Gb/s.

**The interview-grade point:** Google's current fabric doesn't really have a conventional fixed spine layer. Apollo OCS is not a "switch" in the leaf/spine sense — it's a reconfigurable patch panel that lets the topology itself change to match traffic.

## 2.7 Two networks in an ML cluster — a distinction that matters in this org

| | Scale-up domain | Scale-out fabric |
|---|---|---|
| **Connects** | Accelerators within a pod | Pods to each other, and to storage |
| **Technology** | TPU inter-chip interconnect (ICI); NVLink/NVSwitch on GPU systems | Ethernet (or InfiniBand) |
| **Topology** | Torus, or a dedicated switch fabric — **no leaf/spine at all** | Clos / spine-leaf |
| **Bandwidth** | Highest in the system | High, but an order below scale-up |

When someone in the AI infra org says "the fabric," **ask which of the two they mean.** Asking that question correctly is itself a signal of fluency.

> ### Say it out loud — *"Explain Clos / spine-leaf."*
>
> "Clos comes from telephone switching in the fifties — the idea that you can build one large non-blocking switch out of multiple stages of small switches, far cheaper than a single crossbar. Applied to datacenters: instead of one giant chassis, you use many small identical merchant-silicon switches wired in a full mesh. Leaf switches sit at the top of every rack and are the only thing servers plug into. Spines sit in a network row, have no servers attached, and exist purely to forward between leaves. Every leaf connects to every spine, so any rack to any rack is always exactly two hops — there's no near rack and far rack, which is what lets the scheduler place workloads anywhere. You get four properties out of it: uniform latency, all links active via ECMP instead of spanning tree blocking them, scale-out by adding a spine rather than buying a bigger box, and a graceful blast radius — losing one spine of four costs you twenty-five percent of capacity, not connectivity. One thing worth flagging: leaf and spine are roles defined by cabling, not by hardware. The same switch is either one depending on which way its ports face."

---

<a name="part-3"></a>

# Part 3 — Oversubscription and Non-Blocking Design

## 3.1 Definitions

> **Oversubscription ratio** = total downlink capacity ÷ total uplink capacity on a leaf switch.
>
> It expresses how much bandwidth could arrive from the servers versus how much can actually leave the rack toward the rest of the fabric.

> **Non-blocking (1:1)** = uplink capacity equals downlink capacity. Every downlink bit has a guaranteed uplink bit behind it. No contention is possible in the fabric, ever.

A ratio of **3:1** means three units of potential server traffic compete for one unit of fabric capacity.

## 3.2 Worked example — the one to use with an interviewer

### Example A: a classic leaf

```
                    UP TO SPINES
              8 × 400G  =  3,200 Gb/s  (3.2 Tb/s)
                         ▲
                         │
            ┌────────────┴────────────┐
            │       LEAF SWITCH        │
            └────────────┬────────────┘
                         │
                         ▼
                  DOWN TO SERVERS
             48 × 100G  =  4,800 Gb/s  (4.8 Tb/s)


        OVERSUBSCRIPTION  =  4,800 ÷ 3,200  =  1.5 : 1
```

**Reading it aloud:** "If every server in this rack simultaneously tried to send at full line rate to servers outside the rack, we'd only have two-thirds of the bandwidth needed to carry it. One third would queue, and eventually drop."

**To make this leaf non-blocking:** you need 4,800 Gb/s of uplink. That's 12 × 400G instead of 8 × 400G — 50% more uplink optics, 50% more spine ports consumed, 50% more inter-row fiber.

### Example B: the same switch, two different designs

This version shows the ratio as a **design lever**, which is the more sophisticated framing.

Take one 64-port 800G switch — the same physical box in both cases:

| Design | Ports down (servers) | Ports up (spines) | Down capacity | Up capacity | Ratio |
|---|---|---|---|---|---|
| Non-blocking | 32 | 32 | 25.6 Tb/s | 25.6 Tb/s | **1:1** |
| Cost-optimized | 48 | 16 | 38.4 Tb/s | 12.8 Tb/s | **3:1** |

**Same hardware. Same price for the switch itself.** The difference is entirely in how you allocate the ports, and it changes how many servers a rack holds versus how much bandwidth they can each get off the rack.

### Example C: scaling the cost

A 128-rack cluster, comparing the two designs above:

| | 3:1 | 1:1 |
|---|---|---|
| Uplinks per leaf | 16 | 32 |
| Total uplink optics | 2,048 | 4,096 |
| Total spine-side optics | 2,048 | 4,096 |
| Inter-row fiber runs | 2,048 | 4,096 |
| Spine ports required | 2,048 | 4,096 |

Going non-blocking **doubles** the optics count, the spine port count, the fiber runs, and the install labour — and every one of those optics is a long-lead item. It also roughly doubles the power draw of the optical layer.

**This is a bill-of-materials decision with a lead-time consequence, and it is made at design time.**

## 3.3 Why oversubscription exists — the problem it solves

Oversubscription is not a defect. It is a **deliberate economic optimization** resting on one assumption:

> **Not every server bursts off-rack at the same time.**

For general-purpose workloads this assumption is excellent:
- Web servers are mostly idle between requests
- Traffic is bursty and uncorrelated across machines
- A meaningful share of traffic stays **within** the rack and never touches an uplink at all
- Statistical multiplexing smooths the aggregate

If you build 1:1 for a workload that peaks at 20% off-rack utilization, you have bought five times the optics you need — real money, real power, real install time, for capacity that is permanently idle.

**Oversubscription solves the problem of paying for bandwidth nobody uses.**

Historical ratios for context:
| Era / workload | Typical ratio |
|---|---|
| Classic enterprise three-tier | 20:1 or worse |
| General-purpose cloud | 3:1 to 4:1 |
| High-performance / storage | 2:1 to 1.5:1 |
| ML training fabric | 1:1 |

## 3.4 Why it matters in ML training — where the assumption collapses

Distributed training violates the oversubscription assumption in three ways at once.

### Reason 1: the traffic is simultaneous and all-to-all

A synchronous **all-reduce** means every accelerator in the job sends its gradients and waits for every other one before the next step begins. The traffic pattern is **simultaneous, bulk, and all-to-all** — precisely the scenario oversubscription assumes won't happen.

This is not a burst. It happens on **every single training step**, thousands of times a day, for days.

### Reason 2: it's synchronous, so the slowest path sets the pace

```
   Step N:  all 1,024 accelerators exchange gradients
               │
               ├─ 1,023 finish in 40 ms
               └─ 1 is stuck behind a congested uplink: 95 ms
               │
               ▼
   Step N+1 cannot begin until ALL have finished.
   Effective step time = 95 ms.  1,023 accelerators sat idle for 55 ms.
```

One congested link doesn't degrade the job by a few percent. It **stalls the entire cluster** at the synchronization barrier. This is the mechanism behind "stragglers."

### Reason 3: the idle cost is enormous

The compute is the expensive asset. A large training cluster represents tens of millions of dollars of accelerators. If oversubscription costs you 15% of step time, you have **stranded 15% of that capital investment permanently** — far more than the optics you saved.

**The economics flip.** In a web-serving fleet, saving optics is correct. In a training fabric, saving optics to strand accelerators is the most expensive possible mistake.

### The secondary consequence: incast and lossless transport

Oversubscription plus all-to-all traffic produces **incast** — many senders converging on one receiver simultaneously, overrunning the switch egress buffer, dropping packets, triggering retransmits.

TCP handles drops by backing off, which is fatal for a synchronous collective. That is why training fabrics run **lossless transport**: RoCE with PFC (Priority Flow Control), ECN (Explicit Congestion Notification), and DCQCN — mechanisms that push back on senders *before* the buffer overflows, rather than dropping and recovering.

## 3.5 How to think about oversubscription ratios — the framework

Four questions, in order. This is the structure to use if an interviewer asks how you'd approach the decision.

**1. What fraction of traffic actually leaves the rack?**
If 70% of traffic stays intra-rack, a 3:1 uplink ratio is generous. If it's an all-reduce and 100% leaves the rack every step, 3:1 is a guaranteed bottleneck. *You have to characterize the workload before you can pick a number.*

**2. What does congestion cost this workload?**
Web serving degrades **gracefully** — a request takes 40 ms instead of 20 ms and nobody notices. Synchronous training degrades **catastrophically** — the whole cluster stalls. The tolerance for oversubscription is a function of how the workload fails, not how fast it runs.

**3. What do the extra ports actually cost?**
Not just optics. Spine ports, inter-row fiber, structured cabling, install labour, rack space in the network row, power, and — critically — **lead time**. Optics are long-lead items; doubling the count can move the date.

**4. Can you change it later?** — *This is the decisive one.*

**No.** The ratio is physically cabled in. Changing a fabric from 3:1 to 1:1 after the fact means:
- returning to **every** rack in the cluster
- pulling new fiber from every leaf to the spine row
- consuming spine ports that may not exist, potentially requiring a new spine tier
- scheduling maintenance windows against a live workload

> **The oversubscription ratio is a one-way door, decided at design time, driven by the workload it will carry.** For a TPM, that means it must be locked before the bill of materials is issued, and challenging it late is a schedule event, not a design conversation.

## 3.6 The related term: bisection bandwidth

**Bisection bandwidth** = cut the fabric in half; how much bandwidth crosses the cut?

It's the fabric-wide version of the same idea that oversubscription expresses per-leaf. In a tree it's small, because the design assumed traffic went up and out. In a Clos it's large by construction.

When an engineer asks *"what's the bisection on that build?"* they're asking whether the fabric can survive an all-to-all workload — which, per Part 1, is the question the last twenty years of architecture made unavoidable.

> ### Say it out loud — *"Explain oversubscription and non-blocking."*
>
> "Oversubscription is the ratio of downlink capacity to uplink capacity on a leaf switch. Concretely: a leaf with forty-eight hundred-gig ports facing servers has four-point-eight terabits coming down, and if it has eight four-hundred-gig uplinks that's three-point-two terabits going up — so one-point-five to one. If every server in that rack burst off-rack at once, a third of it would queue. Non-blocking means one-to-one — every downlink bit has a guaranteed uplink bit behind it.
>
> Oversubscription isn't a defect, it's an economic bet: general-purpose workloads are bursty and uncorrelated, and a lot of traffic stays in-rack, so building one-to-one means buying optics that sit idle. Three or four to one was standard in cloud fleets for exactly that reason.
>
> ML training breaks the bet completely. An all-reduce is simultaneous, all-to-all, and it happens on every step. And because it's synchronous, the slowest path sets the pace for the whole job — one congested uplink doesn't cost you a few percent, it stalls a thousand accelerators at the barrier. Since the compute is the expensive asset, stranding fifteen percent of it to save optics is the worst trade available. So training fabrics go one-to-one.
>
> The part I'd flag as a program manager is that this is a one-way door. The ratio is physically cabled in. Changing it later means returning to every rack and pulling new fiber, so it has to be locked before the BOM is issued — and challenging it late is a schedule event, not a design conversation."

---

<a name="part-4"></a>

# Part 4 — Hashing From First Principles

## 4.1 What a hash function is

A **hash function** is a recipe that turns any input into a number. You put something in, you get a number out. That's it.

Two properties make it useful here, and **only** these two:

### Property 1 — Deterministic

The same input always produces the same number. Every time, on every machine, forever.

### Property 2 — Scattering

Inputs that look almost identical produce wildly different outputs. There is **no relationship** between how similar two inputs are and how close their outputs are.

```
   INPUT              →   HASH OUTPUT
   "hello"            →   7
   "hellp"            →   91,442          ← one letter different, unrelated output
   "hellz"            →   3,301
```

**This is the scattering property, and it is the entire basis of ECMP's load distribution.** Because outputs are effectively arbitrary, they spread evenly across any range you map them into — without anything measuring, counting, or balancing.

Determinism gives you **correctness** (order preservation). Scattering gives you **load spreading**. You need both, and they come from the same function.

## 4.2 The 5-tuple — what gets hashed

A switch needs to hash "this conversation." The industry definition of a conversation is the **5-tuple** — five fields pulled from the IP and TCP/UDP headers:

| # | Field | Example value | Why it's in there |
|---|---|---|---|
| 1 | Source IP | `10.20.1.47` | Which machine is sending |
| 2 | Destination IP | `10.20.3.88` | Which machine is receiving |
| 3 | Protocol | `6` (TCP) | TCP vs UDP vs other |
| 4 | Source port | `51844` | Which connection on the sending machine |
| 5 | Destination port | `8080` | Which service on the receiving machine |

**The critical property:** these five values are **fixed for the entire life of that connection.** Packet #1 and packet #4,000 of the same TCP connection carry identical values in all five fields.

That is the mechanical basis of everything that follows.

Note that source port is what distinguishes multiple connections between the *same two machines* — the OS picks a fresh ephemeral port for each. This is why two gRPC connections between the same service pair can take different paths.

## 4.3 Worked arithmetic — a toy hash you can do by hand

Real hash functions use bit arithmetic that's unpleasant to follow by hand, so use a fake one that behaves the same way: **add up all the numbers.**

**Flow A:**

```
  Source IP        10.20.1.47   →   10 + 20 + 1 + 47   =     78
  Destination IP   10.20.3.88   →   10 + 20 + 3 + 88   =    121
  Protocol         6            →                            6
  Source port      51844        →                       51,844
  Destination port 8080         →                        8,080
                                                       ─────────
                                          HASH  =        60,129
```

## 4.4 From a big number to "which of my 4 uplinks" — the remainder

The hash gave 60,129. We need a number between 0 and 3.

The mechanism is the **remainder after division** (modulo).

**The intuition you already have:** the last digit of a phone number. Last digits are 0 through 9 and spread pretty evenly across all phone numbers. That "last digit" *is* a remainder — it's what's left after dividing by 10. We have four uplinks instead of ten slots, so we divide by 4.

```
   60,129  ÷  4   =   15,032   remainder  1

   ────────────────────────────────────────────────
   The remainder is ALWAYS 0, 1, 2, or 3 when dividing by 4.
   You can't have 4 left over — another group of 4 would fit.
   So it is GUARANTEED to be a valid uplink number,
   no matter how enormous the hash was.
   ────────────────────────────────────────────────
```

### The same thing without any division

Picture dealing cards to four players: player 0, 1, 2, 3, back to 0, around again. Deal 60,129 cards. Which player gets the last one?

```
   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
   │   Uplink 0   │ │ ► Uplink 1 ◄ │ │   Uplink 2   │ │   Uplink 3   │
   │              │ │              │ │              │ │              │
   │ 4, 8, 12, 16 │ │ 1, 5, 9, 13  │ │ 2, 6, 10, 14 │ │ 3, 7, 11, 15 │
   │      …       │ │      …       │ │      …       │ │      …       │
   │              │ │ … and 60,129 │ │              │ │              │
   └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
```

**Flow A → uplink 1.**

### And a second flow, to see the scattering

Same two servers, second connection. Everything identical except the source port — the OS picked 51901 instead of 51844.

```
   78 + 121 + 6 + 51,901 + 8,080  =  60,186

   60,186 ÷ 4 = 15,046 remainder 2      →   UPLINK 2
```

```
   Server A ──┐                    ┌── Spine 2 ──┐
  10.20.1.47  │                    │ (remainder 1)│
              ├──► Leaf 1 ─────────┤              ├──► Leaf 3 ──► Server B
              │                    │              │             10.20.3.88
              │                    └── Spine 3 ───┘
              │                      (remainder 2)
              
   Same two servers. One port number different. Two different physical paths.
   Nobody balanced anything — the arithmetic did it.
```

## 4.5 The ordering problem — why not just round-robin?

This is the question to have a crisp answer for, because round-robin *is* better at balancing.

> **The difference is the unit of decision. Round-robin decides per packet. Hashing decides per conversation.**

### Why splitting one conversation breaks it

The four paths are equal in **cost** — same hops, same link speed. They are **not equal in delay at any given moment**, because each spine has its own buffer with other traffic sitting in it.

```
   SENDER sends 1, 2, 3, 4 back-to-back.  Round-robin sprays them.

                 ┌─── packet 1 → Spine 1 ── queue  5 µs ───┐
                 │                                          │
   ┌────────┐    ├─── packet 2 → Spine 2 ── queue 60 µs ───┤    ┌──────────┐
   │ SENDER ├────┤                                          ├───►│ RECEIVER │
   └────────┘    ├─── packet 3 → Spine 3 ── queue  8 µs ───┤    └──────────┘
                 │                                          │
                 └─── packet 4 → Spine 4 ── queue  6 µs ───┘

   ARRIVAL ORDER:  1 , 3 , 4 , ......... , 2

   Packet 2 was never lost. It just sat behind someone else's traffic.
```

### What TCP does next

The receiver can only acknowledge a **contiguous** run of bytes:

| Event | Receiver's response |
|---|---|
| Packet 1 arrives | ACK: "I have through 1, send me 2" |
| Packet 3 arrives | Can't ACK 3 — there's a hole. Repeats: "send me 2" → **duplicate ACK** |
| Packet 4 arrives | **Duplicate ACK** again |
| Packet 5 arrives | **Third duplicate ACK** |

**Three duplicate ACKs is TCP's definition of packet loss.** The sender triggers **fast retransmit**: it resends packet 2 — which was never lost and is still in flight — and, expensively, **halves its congestion window**.

The connection now runs at half throughput recovering from a problem that didn't exist. And round-robin doesn't do this once. It does it **on every burst, forever**, because the cause is structural.

**A second cost: head-of-line blocking.** The receiver has packets 3, 4, 5 sitting in its reassembly buffer but cannot hand a single byte to the application until packet 2 arrives. Real data, delivered, unusable.

**For RDMA over Ethernet it's worse.** Classic RoCE required strict in-order delivery and would discard out-of-order arrivals, forcing go-back-N retransmission of everything after the gap.

### The secondary difference: state

| | Round-robin | Hashing |
|---|---|---|
| State required | A counter per group, updated at line rate | **None** |
| Survives a reboot consistently? | No | Yes |
| Cost in silicon | Real | Essentially free |

Hashing is **stateless**. The switch remembers nothing. It recomputes the answer fresh from the packet's own header every time.

## 4.6 How hashing preserves order

> **Hashing doesn't "handle" ordering. It makes the problem impossible.**

That distinction matters. Round-robin *creates* a reordering problem and then needs machinery to fix it. Hashing is designed so the problem never occurs.

### Why one path means in-order, automatically

A single physical path is a **chain of FIFO queues**:

```
   Leaf egress buffer  →  fiber  →  Spine egress buffer  →  fiber  →  Leaf buffer
        (FIFO)            (in       (FIFO)                   (in      (FIFO)
                         order)                             order)
```

Nowhere along that chain is there any mechanism that could move packet 3 ahead of packet 2. **Reordering requires choice** — two routes with different delays and something deciding between them. Remove the choice and you remove the possibility.

So the hash isn't preserving order through cleverness. It makes **one decision per conversation** instead of one per packet, and everything downstream of that decision is a queue.

### The precise scope of the guarantee

Order is preserved **within a flow**, and only within a flow.

Two separate connections between the same pair of servers can arrive in any relative order. That's fine — they're independent conversations and TCP sequence numbers are per-connection. **The 5-tuple is the hash input precisely because those five fields are the definition of "one conversation."**

### The one-sentence version

> Hashing preserves order by making the path a **property of the conversation** rather than of the moment. Compute it once from fields that never change, and every packet inherits the same answer for the life of the flow.

## 4.7 What real switches do differently

Only the recipe changes. The shape is identical.

**CRC instead of digit-sum.** Adding digits scatters poorly — ports 51844 and 51845 land in adjacent buckets, which isn't very scattered. Real switches use a CRC variant built into the ASIC, which takes inputs differing by one bit and produces unrelated outputs.

**Per-device hash seed.** Without it, every switch in the fabric computes the same hash for the same flow, and traffic funnels onto the same relative path at every tier while other links sit idle. This is called **polarization**. Seeding each device differently breaks the correlation.

**Configurable hash inputs.** Some fabrics hash on fewer or more fields — for tunneled traffic (VXLAN, GRE) the outer header is what the spine sees, so entropy has to be deliberately injected into the outer source port, or every tunneled flow between a rack pair collapses onto one path.

> ### Say it out loud — *"How does ECMP hashing work and how does it preserve order?"*
>
> "A hash function is just a recipe that turns an input into a number, with two properties: it's deterministic — same input, same number, always — and it scatters, so similar inputs give unrelated outputs. The switch hashes the five-tuple: source and destination IP, protocol, and source and destination port. Those five values are fixed for the life of a TCP connection, so every packet of that connection hashes to the same number. Then it takes that number modulo the number of uplinks — with four uplinks the remainder is always zero through three, so it's guaranteed to be a valid port.
>
> Order falls out for free. A single path is just a chain of FIFO queues; nothing along it can reorder anything. Reordering requires a choice between two paths with different delays, and by making one decision per conversation instead of one per packet, you've removed the choice.
>
> That's also the answer to why not round-robin. Round-robin balances perfectly, but it sprays one conversation across four paths with different queue depths, so packets arrive out of order. The receiver sends duplicate ACKs, TCP reads three dupes as loss, triggers a fast retransmit and halves the congestion window — so you'd get perfect link utilization and terrible application throughput. Hashing trades balancing quality for in-order delivery. Load spreading still happens, just statistically across many flows rather than within one."

---

<a name="part-5"></a>

# Part 5 — BGP Inside the Switch

## 5.1 What BGP fundamentally is

BGP is a **path vector** protocol. A router tells its neighbours: *"I can reach prefix X, and here is the list of networks this route passed through to reach me."* The neighbour adds itself to that list and passes it on.

Two jobs come out of one mechanism:
1. You learn **reachability** — who can reach what
2. The accumulated list — the **AS_PATH** — lets you **detect loops** and **compare route lengths**

An **AS** (autonomous system) is an administrative identity with a number. On the internet these are globally assigned. Inside a datacenter you use **private ASNs** and assign them yourself.

## 5.2 Why BGP ended up inside a datacenter

This surprised the industry. BGP was the internet's protocol; datacenters ran OSPF or IS-IS. The switch is documented in **RFC 7938, "Use of BGP for Routing in Large-Scale Data Centers."** The reasoning:

| Concern | Link-state (OSPF/IS-IS) | BGP |
|---|---|---|
| **Change propagation** | Floods LSAs to every node; every node recomputes the entire graph | Propagates only the changed prefix, hop by hop |
| **State held** | Complete topology at every router | Only routes received from neighbours |
| **Behaviour at scale** | Flooding and recomputation churn across thousands of nodes | Incremental, locally contained |
| **Policy control** | Limited per-hop policy | Rich per-hop filtering and tagging |
| **Operational familiarity** | Datacenter-specific tuning | Every engineer already knew it |

The **policy** point matters more than it sounds: you can filter and tag exactly what each switch advertises, which is how you **drain a switch for maintenance** — stop advertising through it, let traffic move off, then work on it. That's an operational capability a TPM cares about directly, because it's what makes a maintenance window possible without an outage.

## 5.3 Session mechanics

BGP does not discover neighbours by broadcast. Every session is **explicitly configured** and runs over a **TCP connection on port 179**.

That's unusual — most routing protocols use their own transport — and it means BGP inherits TCP's reliability and ordering rather than implementing its own.

### Session states

```
   Idle  →  Connect  →  OpenSent  →  OpenConfirm  →  ESTABLISHED
                                                          │
                                        only here does route exchange happen
```

When someone says *"the session is flapping,"* they mean it keeps falling out of Established.

### Message types

| Message | Purpose |
|---|---|
| **OPEN** | Negotiates the session: my ASN, router ID, hold time, capabilities |
| **UPDATE** | The actual work — prefixes advertised (with attributes) and prefixes withdrawn |
| **KEEPALIVE** | Proof of life |
| **NOTIFICATION** | Something is wrong; tear the session down |

### Timers — and why datacenter values differ

The **hold timer** is the failure detector: no KEEPALIVE within the hold time and the session drops.

| Setting | Internet default | Datacenter fabric |
|---|---|---|
| Keepalive interval | 60 s | ~1 s |
| Hold timer | 180 s | ~3 s |
| Typical detection | Up to 3 minutes | Sub-second with BFD |

Three minutes of black-holed traffic is unusable in a fabric. Beyond aggressive timers, fabrics run **BFD (Bidirectional Forwarding Detection)** alongside BGP — a lightweight hello mechanism detecting failure in **milliseconds** and telling BGP to tear the session down immediately.

**BFD support and timer capability are NPI qualification criteria**, not deployment details. A switch generation that can't hold sub-second detection changes the failure blast radius for every job on that fabric.

## 5.4 The AS numbering design

In a fabric, every switch is its own AS, and **every link is an eBGP session** — external, between different ASNs. There is essentially no iBGP.

```
        ┌──────────────────┐        ┌──────────────────┐
        │     Spine 1      │        │     Spine 2      │
        │    AS 65500      │        │    AS 65500      │   ← spines SHARE an ASN
        └───┬─────┬────┬───┘        └──┬─────┬─────┬───┘
            │     │    └───────────────┼──┐  │     │
            │     └───────────┐        │  │  │     │
            │          ┌──────┼────────┘  │  │     │
     ┌──────┴───┐  ┌───┴──────┴─┐  ┌──────┴──┴───┐
     │  Leaf 1  │  │   Leaf 2   │  │   Leaf 3    │
     │ AS 65001 │  │  AS 65002  │  │  AS 65003   │      ← leaves UNIQUE ASNs
     └──────────┘  └────────────┘  └─────────────┘

     One eBGP session per cable. Six sessions here.
```

**ASN ranges:**
- Two-byte private range: **64512–65534** — only about 1,023 numbers. Large fabrics exhaust this.
- Four-byte private range: **4200000000–4294967294** — what hyperscale fabrics use.

## 5.5 Why communication can't be spine → leaf → spine

This is the loop-prevention question, and it has two layers.

### Layer 1 — the topology itself

A two-tier Clos is **bipartite**: spines connect only to leaves, leaves connect only to spines. **Spines are not cabled to each other at all.** So spine-to-spine is only physically conceivable by transiting a leaf.

### Layer 2 — BGP's AS_PATH loop rule enforces it

> **BGP's loop rule: if a router receives a route whose AS_PATH already contains its own ASN, it rejects the route.** It must have originated or already carried that route, so accepting it would close a loop.

Now apply the shared spine ASN:

```
   Suppose Spine 1 receives, via Leaf 2, a route that originally came from Spine 2.

   AS_PATH as seen by Spine 1:   [ 65500 , 65002 , 65003 ]
                                   ▲
                                   └── Spine 1's OWN ASN is 65500

   → Spine 1 REJECTS the route.
   → Spine → Leaf → Spine is structurally impossible.
```

Giving both spines **65500** means any route that has already passed through *any* spine carries 65500 in its AS_PATH, so **no spine will ever accept it.** Traffic can only go **leaf → spine → leaf**, which is exactly the fabric's intended behaviour — enforced by the protocol rather than by policy you have to write and maintain.

### Why leaves get unique ASNs — the mirror image

Leaf 1 **must** be able to accept routes originated by Leaf 3. If all leaves shared an ASN, Leaf 1 would see its own number in the AS_PATH of Leaf 3's route and reject it — and the fabric would carry no traffic at all.

| Tier | ASN assignment | Reason |
|---|---|---|
| Spines | **Shared** (e.g. all 65500) | Blocks spine → leaf → spine transit |
| Leaves | **Unique** (65001, 65002, …) | Allows leaf → spine → leaf to be accepted |

**This is an elegant piece of design and a great thing to be able to explain**: the loop prevention isn't a firewall rule or an ACL. It's a consequence of how you chose to number the switches.

### The other reason it matters: valley-free routing

If a spine *could* transit to another spine via a leaf, that leaf would be carrying transit traffic on behalf of the fabric — consuming its uplink capacity, which was sized for its own rack's servers. The shared-ASN design prevents a leaf from ever becoming an accidental transit path.

## 5.6 How BGP produces the ECMP group

Here is where Part 5 connects to Part 6.

Leaf 1 receives the route to `10.20.3.0/24` from both spines:

```
   via Spine 1:   AS_PATH = 65500  65003
   via Spine 2:   AS_PATH = 65500  65003
                  ─────────────────────
                  IDENTICAL length. IDENTICAL content.
```

BGP's best-path algorithm has nothing to break the tie on. With multipath enabled, **both get installed** — and that is your ECMP group.

### The configuration knob

BGP's **default** is to run best-path selection, pick **one** winner, and discard the rest. Left alone, it installs a single next hop and you are back to a tree.

ECMP requires explicitly enabling multipath: `maximum-paths 4` or equivalent.

### The classic gotcha: `as-path multipath-relax`

Some fabrics give each spine a **unique** ASN instead of a shared one:

```
   via Spine 1:   AS_PATH = 65501  65003
   via Spine 2:   AS_PATH = 65502  65003
                  ─────────────────────
                  Same LENGTH. Different CONTENT.
```

Standard BGP refuses to treat those as equal and picks one. You must configure **`bestpath as-path multipath-relax`**, which tells BGP to compare AS_PATH **length only** and ignore the contents.

**Forgetting this knob is a classic fabric bug:** the cabling is right, ECMP looks configured, and traffic still uses one spine. Worth knowing because it's exactly the kind of defect that shows up as "the fabric is only delivering a quarter of expected throughput" during a build validation.

## 5.7 The one-line summary

> **In a fabric, BGP is not doing internet routing. It is acting as a distributed mechanism for building and maintaining ECMP groups** — each switch advertising what it can reach, ties happening by design because the topology is symmetric, and every tie becoming another member of a load-sharing group.

> ### Say it out loud — *"Explain BGP in a datacenter fabric."*
>
> "BGP is a path-vector protocol — each router advertises what it can reach plus the list of AS numbers the route travelled through. In the datacenter we run it per RFC 7938, which was a deliberate move away from OSPF: link-state protocols flood every change to every node and make everyone recompute the whole topology, which doesn't scale well across thousands of switches with constant churn. BGP propagates only what changed, prefix by prefix, and gives you per-hop policy — which is what lets you drain a switch cleanly for maintenance.
>
> The design is eBGP everywhere — every switch its own AS, every cable a session over TCP port 179. Each leaf gets a unique private ASN, and all the spines share one. That shared spine ASN is doing real work: BGP rejects any route whose AS_PATH already contains its own ASN, so a route that's been through one spine can never be accepted by another spine. Spine-to-leaf-to-spine is structurally impossible — the loop prevention comes out of the numbering scheme, not a firewall rule. Leaves need unique numbers for the opposite reason: leaf one has to be able to accept routes from leaf three.
>
> And the practical output is that BGP is really what builds the ECMP groups. Leaf one learns the same prefix from all four spines with identical AS_PATH length, nothing breaks the tie, so with `maximum-paths` enabled all four get installed as a group. One gotcha: if your spines have unique ASNs instead of a shared one, the AS_PATHs differ in content, and you need `as-path multipath-relax` or BGP silently picks one spine and you lose three-quarters of your fabric."

---

<a name="part-6"></a>

# Part 6 — ECMP Under the Hood

## 6.1 What ECMP is

**ECMP — Equal-Cost Multipath.** The forwarding behaviour that installs **all** tied next hops for a destination and picks among them **per flow**.

Without it, a routing protocol picks one best next hop and three of your four cables sit dark.

## 6.2 Step 1 — where the equal paths come from

Leaf 1 doesn't know the fabric's shape. It learns reachability from BGP (§5) or an SDN controller.

```
   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ Spine 1 │  │ Spine 2 │  │ Spine 3 │  │ Spine 4 │
   │10.20.3.0│  │10.20.3.0│  │10.20.3.0│  │10.20.3.0│   all advertise the
   │   /24   │  │   /24   │  │   /24   │  │   /24   │   SAME prefix
   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘
        └───────┬────┴──────┬─────┴────────────┘
                ▼           ▼
          ┌──────────────────────┐
          │       Leaf 1         │
          │  four equal next hops │
          └──────────────────────┘
```

Every spine is connected to every leaf, so Leaf 3 tells all four spines it can reach `10.20.3.0/24`, and all four spines tell Leaf 1 the same thing. Same prefix, same cost, four different next hops.

## 6.3 Step 2 — the switch must be told to keep all four

Covered in §5.6: `maximum-paths 4`, plus `as-path multipath-relax` if spines have unique ASNs.

## 6.4 Step 3 — what's actually in the forwarding table

The result is **not** four separate routes. It is **one route pointing at a group**:

```
   FORWARDING TABLE (FIB)
   ┌────────────────────┬──────────────────┐
   │ Destination prefix │ Action           │
   ├────────────────────┼──────────────────┤
   │ 10.20.3.0/24       │ → ECMP group 7   │
   └────────────────────┴──────────────────┘

   ECMP GROUP 7
   ┌────────┬──────────────┬──────────────┐
   │ Member │ Egress port  │ Toward       │
   ├────────┼──────────────┼──────────────┤
   │   0    │  Et1/49      │  Spine 1     │
   │   1    │  Et1/50      │  Spine 2     │
   │   2    │  Et1/51      │  Spine 3     │
   │   3    │  Et1/52      │  Spine 4     │
   └────────┴──────────────┴──────────────┘
```

The indirection matters: many prefixes can point at the same group, so when a spine fails you update **one group** rather than thousands of routes. This is why failover is fast.

## 6.5 Step 4 — the packet walk

Four operations, all in hardware, all in nanoseconds:

```
   ┌──────────────────────────────────────────────────────────────┐
   │ 1. LOOKUP destination IP                                     │
   │    10.20.3.88 matches 10.20.3.0/24                           │
   │    Result: "ECMP group 7"  ← a group, not a port             │
   ├──────────────────────────────────────────────────────────────┤
   │ 2. READ the 5-tuple from the packet header                   │
   │    src IP, dst IP, protocol, src port, dst port              │
   ├──────────────────────────────────────────────────────────────┤
   │ 3. HASH those five fields, take remainder over group size    │
   │    CRC(5-tuple) mod 4  =  1                                  │
   ├──────────────────────────────────────────────────────────────┤
   │ 4. FORWARD out member 1  →  Et1/50  →  Spine 2               │
   └──────────────────────────────────────────────────────────────┘

   The switch keeps NO record that this happened.
   The next packet of the same flow repeats all four steps from
   scratch and arrives at the same answer — because the inputs
   are identical.
```

## 6.6 Step 5 — failure behaviour, and the contrast with spanning tree

### What happens when a spine dies

Leaf 1 stops receiving BGP keepalives from Spine 2, or BFD detects the link down. BGP withdraws that path. **The group shrinks from four members to three.**

Notice what did **not** happen:
- No topology recalculation
- No fabric-wide convergence event
- No traffic black-holed while the network re-thinks itself

Flows on the other three spines are untouched. Capacity is now 75% of nominal.

### The spanning tree comparison

The tree wasn't just a different topology — it operated at a different layer with an incompatible loop-prevention model.

**Why STP existed:** Ethernet frames have **no TTL field**, so a frame caught in a loop circulates forever and broadcast traffic multiplies until the network melts down. STP prevents this by computing a loop-free tree and **administratively disabling** every link not in it.

```
        ┌────────────────┐            ┌────────────────┐
        │ Aggregation A  │            │ Aggregation B  │
        │  ACTIVE PATH   │            │ BLOCKED BY STP │
        └───────▲────────┘            └───────╳────────┘
                │                             ╎
                │                             ╎ (disabled)
                └──────────┬──────────────────┘
                   ┌───────┴────────┐
                   │ Access switch  │
                   │ two uplinks,   │
                   │ ONE usable     │
                   └────────────────┘
```

| | Spanning tree | ECMP |
|---|---|---|
| **Links in use** | 1 active, N-1 hot spares | All N simultaneously |
| **Bandwidth from redundancy** | Zero | Full |
| **Failover** | Recompute tree + reconverge; classic STP tens of seconds, RSTP a few seconds; traffic black-holed meanwhile | Remove one member from a group; sub-second, existing flows untouched |
| **Adding capacity** | Buy a bigger core chassis (procurement event) | Add another group member (cabling job) |
| **Layer** | Layer 2, no TTL, loops are fatal | Layer 3 to the ToR, TTL bounds any loop |

## 6.7 Resilient hashing — protecting order when the group changes

There is one case where hashing **does** reorder, and real fabrics engineer around it.

### The problem with plain modulo

The hash is reduced modulo the **number of group members**. What happens when that number changes?

```
   Flow A hash = 60,129

   With 4 members:   60,129 mod 4  =  1    →  uplink 1
   With 3 members:   60,129 mod 3  =  0    →  uplink 0    ← IT MOVED

   But uplink 1 was perfectly healthy.
```

Plain modulo reshuffles **every flow** when the divisor changes, not just the flows on the broken link. Mid-flight packets end up spread across two paths with different queue depths — exactly the reordering storm you were avoiding, fabric-wide, at the worst possible moment.

### The fix: a bucket table

Switches insert a lookup table between the hash and the uplink — typically 128 or more **buckets**, each pointing at a member. The hash selects a bucket; the bucket names the uplink.

When a member dies, **only the buckets pointing at the dead member are reassigned.** Every other bucket keeps its pointer.

```
   BEFORE — 8 buckets across 4 spines
   ┌────┬────┬────┬────┬────┬────┬────┬────┐
   │ B0 │ B1 │ B2 │ B3 │ B4 │ B5 │ B6 │ B7 │
   │ S1 │ S2 │ S3 │ S4 │ S1 │ S2 │ S3 │ S4 │
   └────┴────┴─╳──┴────┴────┴────┴─╳──┴────┘
                │                   │
         these two point at S3, which is about to fail

   AFTER — spine 3 fails
   ┌────┬────┬────┬────┬────┬────┬────┬────┐
   │ B0 │ B1 │ B2 │ B3 │ B4 │ B5 │ B6 │ B7 │
   │ S1 │ S2 │ S4 │ S4 │ S1 │ S2 │ S1 │ S4 │
   └────┴────┴─▲──┴────┴────┴────┴─▲──┴────┘
               │                   │
          only these two moved.  Six of eight untouched.
```

**The guarantee:** flows whose buckets didn't move never notice the failure happened. Their packets keep taking the same physical path, still in a single FIFO chain, still in order. Only flows genuinely on the broken link are disrupted — and those were going to be disrupted regardless.

**Resilient hashing support is an NPI qualification criterion.** A switch generation without it means every link failure causes a fabric-wide throughput dip rather than a localized one.

> ### Say it out loud — *"How does ECMP work under the hood?"*
>
> "Start with where the choices come from. In a Clos, every leaf reaches every other leaf through every spine, so BGP hands leaf one the same destination prefix from all four spines with identical cost. BGP's default is to pick one winner, so you explicitly enable multipath — then all four get installed, not as four routes but as one route pointing at an ECMP group.
>
> Per packet, the switch does four things in hardware: match the destination prefix, which resolves to a group rather than a port; read the five-tuple from the header; CRC-hash those five fields and take the remainder over the group size; forward out that member. It stores nothing — the next packet recomputes and gets the same answer because the five-tuple is fixed for the life of the connection. That's what preserves ordering, and the scattering property of the hash is what spreads different flows across members without anything measuring load.
>
> Failure is where it really beats the old design. A spine goes down, BFD catches it in milliseconds, BGP withdraws, and the group drops from four members to three. No topology recomputation, no fabric-wide convergence, flows on the other three spines untouched — you're at seventy-five percent capacity, sub-second. Compare spanning tree, which had to recompute the tree and black-hole traffic for seconds. One refinement worth knowing: naive modulo would reshuffle every flow when the group size changes, so switches use a bucket table and remap only the buckets pointing at the dead member."

---

<a name="part-7"></a>

# Part 7 — Limitations of ECMP

This section is where you demonstrate that you know the limits of the textbook answer. Raising these unprompted is a strong signal.

## 7.1 It is blind to volume — the elephant flow problem

**The most important limitation, and the dominant concern in AI fabrics.**

The hash sees **header fields**, not bytes. It has no idea whether a flow is carrying 10 Kb/s or 400 Gb/s.

ECMP balances well when there are **many small flows** — the law of large numbers does the work. Training traffic is the opposite: **a handful of enormous, long-lived flows** doing gradient exchange.

```
   4 uplinks, 4 elephant flows, each wanting 400 Gb/s

   Uplink 0:  ████████████████  Flow A  (400G)
   Uplink 1:  ░░░░░░░░░░░░░░░░  idle
   Uplink 2:  ████████████████  Flow B + Flow C  (800G wanted, 400G available)
   Uplink 3:  ████████████████  Flow D  (400G)
                                    ▲
                         COLLISION. Flows B and C hashed to the same member.
                         Uplink 2 is congested; uplink 1 carries nothing.
```

With four uplinks and four elephant flows, the probability of a collision is substantial. And because these flows run at line rate continuously, the congested link stays congested — the hash has no mechanism to notice and no mechanism to correct.

**In a synchronous training job, that congested link becomes the straggler that sets the pace for the entire cluster** (§3.4).

**Note:** the same effect appears in a milder form with gRPC (§1.6). HTTP/2 multiplexing means one long-lived TCP connection carries all traffic between a service pair, so that entire connection pins to one path.

## 7.2 It assumes symmetry

ECMP is only **correct** if all members are genuinely equivalent. The word "equal-cost" is about routing metric, not about capacity.

Break symmetry and the hash still spreads evenly — but the paths no longer have equal capacity:

```
   Leaf 1 has 4 uplinks at 400G  → 1.6 Tb/s, spread 25% per member
   Leaf 2 has 3 uplinks at 400G  → 1.2 Tb/s, spread 33% per member

   Both leaves send equal traffic toward the same spine.
   Spine sees uneven load it cannot correct, because the imbalance
   was created upstream by the cabling, not by the hash.
```

Sources of asymmetry, all of which a TPM can cause or prevent:
- **Partial spine deployments** — a new spine cabled to half the leaves during an expansion wave
- **Mixed port speeds** across a tier during a generational upgrade
- **Leaves with different uplink counts** — often a result of a rack being built to an older standard
- **A failed link on one leaf** leaving it at 3 uplinks while its peers have 4

> **This is a direct build-plan constraint:** expansion waves have to be designed to keep the fabric uniform, or to run deliberately degraded until the wave completes. "We'll cable half the leaves this week and half next week" is a fabric-performance decision, not just a schedule decision.

## 7.3 Group size changes disturb flows

Covered mechanically in §6.7. The limitation in summary: with naive modulo, any change to the number of members reshuffles **every** flow, not just affected ones. Resilient hashing mitigates it but requires hardware support.

## 7.4 Polarization

If every switch uses the same hash function with the same seed, a flow that hashed to "member 2" at the leaf will hash to "member 2" again at the spine. Traffic funnels onto correlated paths at every tier while other links sit idle.

Mitigated by **per-device hash seeding**. Worth knowing because it's a subtle defect: the fabric looks correctly configured and simply underperforms.

## 7.5 Poor entropy in tunneled or uniform traffic

The hash needs **variation** in the five fields to scatter. Traffic that lacks it defeats the mechanism:

- **Tunneled traffic** (VXLAN, GRE, IPsec): the spine sees only the outer header. If the outer 5-tuple is identical for all inner flows between a rack pair, they all collapse onto one path. Fixed by deliberately deriving the outer source port from a hash of the inner headers — "entropy injection."
- **Single-connection bulk transfers**: one TCP connection is one flow, full stop. It can never use more than one path's worth of bandwidth no matter how many uplinks exist.

## 7.6 The mitigations — in increasing order of sophistication

| Technique | What it does | Trade-off |
|---|---|---|
| **Flowlet switching** | Exploits natural gaps in a TCP flow's packet train. If a gap exceeds the maximum path latency difference, you can safely re-hash to a different link without risking reordering. | Depends on the traffic having gaps; bulk training flows have few |
| **Adaptive / congestion-aware routing** | The switch observes queue depth per member and steers new flows away from congested paths instead of hashing blind | Requires hardware support; reacts rather than prevents |
| **Packet spraying + reorder-tolerant transport** | Spray per packet for near-perfect balance, and push reassembly into the NIC so the endpoint doesn't mistake reordering for loss | Requires specific NICs and transport stacks **end to end** — a BOM decision, not a config |
| **Topology engineering** | Don't balance traffic over a fixed topology — **reconfigure the physical topology** to match the communication pattern. Google's Apollo OCS in Jupiter Evolving | Requires optical circuit switching infrastructure |

### The framing that elevates this answer

The industry has come back around to round-robin. **Packet spraying** is the better balancing strategy, and modern AI fabrics do it. What changed is that the **endpoint** got fixed rather than the network being constrained to protect it.

If the receiving NIC can reassemble out-of-order packets in hardware, reordering stops being a signal of loss and becomes a normal condition. That's the design in NVIDIA's adaptive routing on InfiniBand and Spectrum-X, and it's central to **Ultra Ethernet**.

So the honest summary:

> - **Hashing** is what you do when the transport can't tolerate reordering. It trades balancing quality for in-order delivery. General-purpose fabrics.
> - **Spraying** is what you do when you've built a transport that can. Near-perfect balance, which is exactly what elephant-flow training traffic needs, at the price of requiring specific NICs and transport stacks end to end.

**This reframes "which load-balancing scheme" from a networking trivia question into a fabric-and-endpoint co-design decision** — which is the level a TPM owning an ML capacity build actually operates at. It determines the NIC bill of materials, not just a switch config.

> ### Say it out loud — *"What are the limitations of ECMP?"*
>
> "The big one is that the hash is blind to volume — it sees header fields, not bytes. That's fine when you have thousands of small flows, because the law of large numbers does the balancing for you. It fails badly with a handful of elephant flows, which is exactly what gradient exchange looks like. Two four-hundred-gig flows hash to the same uplink, that link saturates while another sits empty, and in a synchronous training job that congested link becomes the straggler setting the pace for the whole cluster.
>
> Second, it assumes the members are genuinely equivalent. 'Equal cost' is a routing metric, not a capacity guarantee. If one leaf has four uplinks and its neighbour has three — because of a failed link or a partial expansion wave — the hash still spreads evenly but the paths don't have equal capacity. That's a build-plan constraint: cabling half a spine's leaves this week and half next week is a fabric-performance decision, not just a schedule one.
>
> Third, changing group size reshuffles flows unless the hardware supports resilient hashing with a bucket table. And fourth, tunneled traffic can have poor entropy — if the outer header is identical for every inner flow, everything collapses onto one path.
>
> The interesting part is where the industry landed. Per-packet spraying is actually the better balancer; it was ruled out because TCP reads reordering as loss. Once the NIC can reassemble out of order in hardware, spraying becomes viable again — that's Ultra Ethernet and NVIDIA's adaptive routing. So it's really a fabric-and-endpoint co-design question, and it lands in the NIC bill of materials, not just a switch config."

---

<a name="part-8"></a>

# Part 8 — TPM Translation Layer

The technical material above is table stakes. **This section is the differentiator.** For each concept, the question is: what does this turn into on a program plan?

## 8.1 Concept → program constraint

| Fabric concept | What it becomes for a TPM |
|---|---|
| **Oversubscription ratio** | A **one-way door** decided at design time. Determines the BOM and therefore the long-lead optics order. Decide it late and you've blown a lead time, not just a design review. |
| **Non-blocking (1:1)** | Roughly **doubles** spine ports, inter-row fiber, optics count, install labour, and optical-layer power versus 3:1. A budget and schedule line, not an architecture preference. |
| **Adding a spine** | Touches **every leaf** in the fabric. A capacity expansion is an N-leaf coordinated change, not a single maintenance window. |
| **Fabric symmetry** | Expansion waves must preserve uniformity or accept known degradation. "Half the leaves this week" is a performance decision. |
| **Hitless upgrade capability (OCS)** | The difference between expanding a cluster mid-quarter and negotiating a window with an ML team whose training run is eleven days in. |
| **Resilient hashing support** | An **NPI exit criterion**. Without it, every link failure is a fabric-wide throughput dip rather than a localized one. |
| **BFD / sub-second failure detection** | An **NPI qualification criterion**. Determines failure blast radius for every job on the fabric. |
| **ASN and IP allocation** | A **design artifact that must be correct before rack turn-up.** Unique per leaf, shared per spine tier. Correcting it later means touching live config. Belongs in the build data model — and it is exactly what breaks when "data fidelity across systems" is poor. |
| **`as-path multipath-relax`** | A config defect that presents as "the fabric delivers a quarter of expected throughput" during build validation. A validation test case, not a trivia item. |
| **Optics reach class** | Spine-facing optics are a different, longer-lead BOM line than in-rack DAC. Mixing them up is a materials blocker. |
| **NIC capability (reorder tolerance)** | If the fabric plan assumes packet spraying, the NIC BOM is part of the fabric decision. Endpoint and fabric must be specified together. |

## 8.2 The triage skill this enables

The reason to hold all of this is that several very different problems present **identically** from the ML team's side — "the training job is slow":

| Symptom presented | Possible cause | Remediation | Lead time |
|---|---|---|---|
| Training job slow | Fabric genuinely oversubscribed | Capacity build | Months — long-lead optics |
| Training job slow | Elephant flows colliding on one uplink | Adaptive routing / hash tuning | Days — config |
| Training job slow | Fabric asymmetry from a partial wave | Complete the wave, or rebalance | Weeks |
| Training job slow | `multipath-relax` missing — traffic on one spine | Config fix | Hours |
| Training job slow | Bad optic causing link flap and repeated rehash | Component swap from spares | Hours, **if spares are staged** |

**Being able to sit in that triage and ask which one you're looking at — rather than relaying the complaint — is the difference between the TPM being in the loop and being a message bus.**

## 8.3 The executive translation

The role explicitly calls for status that leads with **machine/cluster impact**, not task status. The fabric concepts above are what let you write that sentence correctly.

| Weak (task status) | Strong (impact) |
|---|---|
| "Spine cabling is 60% complete." | "Pod 3 is running at 3 uplinks per leaf instead of 4 until the wave completes on the 14th. Effective bisection is 75% of target, which caps the pod at ~1,500 accelerators of synchronous training rather than 2,048." |
| "Optics order is delayed." | "The 800G LR4 order slipped four weeks. That pushes pod 4 fabric completion past the ML org's capacity date; the mitigation is either accepting 2:1 on pod 4 at launch or re-sequencing pod 5 ahead of it." |
| "NPI gate 3 is at risk." | "Resilient hashing hasn't passed validation on the new ToR. If it ships without it, a single link failure becomes a fabric-wide throughput event rather than a rack-local one — I'm recommending we hold the gate." |

## 8.4 Glossary

| Term | Definition |
|---|---|
| **All-reduce** | Collective operation where every participant contributes a value and every participant receives the combined result. The dominant traffic pattern in synchronous distributed training. |
| **AS / ASN** | Autonomous System / its number. An administrative routing identity. |
| **AS_PATH** | BGP attribute listing the ASNs a route has traversed. Used for loop detection and length comparison. |
| **BFD** | Bidirectional Forwarding Detection. Lightweight sub-second link failure detection that triggers BGP teardown. |
| **Bisection bandwidth** | Bandwidth crossing a cut that divides the fabric in half. The fabric-wide measure of all-to-all capacity. |
| **Blast radius** | Scope of impact when something fails. |
| **Clos** | Multi-stage switching topology (Charles Clos, 1953) that builds a large non-blocking switch from small ones. |
| **DAC** | Direct-Attach Copper. Short cable used inside a rack for server-to-ToR links. |
| **DCQCN** | Congestion control algorithm for RoCE, using ECN marks. |
| **East-west** | Traffic between machines inside the datacenter. |
| **ECMP** | Equal-Cost Multipath. Installing all tied next hops and hashing flows across them. |
| **ECN** | Explicit Congestion Notification. Marks packets to signal congestion before drops occur. |
| **Elephant flow** | A single, very large, long-lived flow. Defeats hash-based balancing. |
| **eBGP / iBGP** | External BGP (between different ASNs) / internal BGP (within one ASN). Fabrics use eBGP everywhere. |
| **FIB** | Forwarding Information Base. The hardware forwarding table. |
| **Flowlet** | A burst of packets within a flow, separated from the next burst by a gap large enough to allow safe re-hashing. |
| **GGC** | Google Global Cache — edge caching appliances placed inside ISP networks. |
| **gRPC** | Google's open-source RPC framework. Protocol Buffers over HTTP/2. |
| **HPACK** | HTTP/2 header compression. |
| **ICI** | Inter-Chip Interconnect. TPU pod's scale-up network. |
| **Incast** | Many senders converging on one receiver simultaneously, overrunning buffers. |
| **Jupiter** | Google's datacenter fabric architecture. |
| **Leaf / ToR** | The switch at the top of every server rack; the only switch servers plug into. |
| **Merchant silicon** | Off-the-shelf switching ASICs (Broadcom Tomahawk/Trident/Jericho) rather than proprietary chips. |
| **Multipath-relax** | BGP config allowing ECMP across paths with equal AS_PATH length but different content. |
| **Non-blocking** | 1:1 oversubscription. Uplink capacity equals downlink capacity. |
| **North-south** | Traffic crossing the datacenter boundary. |
| **OCS / Apollo** | Optical Circuit Switching; Google's MEMS-mirror implementation in Jupiter Evolving. |
| **Orion** | Google's SDN controller coordinating topology and traffic engineering. |
| **Oversubscription ratio** | Downlink capacity ÷ uplink capacity on a leaf. |
| **PFC** | Priority Flow Control. Pauses a sender to prevent buffer overflow — the basis of lossless Ethernet. |
| **Polarization** | Correlated hash decisions across tiers causing uneven link use. |
| **Protocol Buffers** | Google's binary serialization format and IDL, used by gRPC. |
| **RDMA / RoCE** | Remote Direct Memory Access / RDMA over Converged Ethernet. |
| **Resilient hashing** | Bucket-table indirection so only flows on a failed member are remapped. |
| **RPC** | Remote Procedure Call. A function call that executes on another machine. |
| **Scale-up vs scale-out** | Bigger single machine vs more machines. Also, within ML: the intra-pod accelerator network vs the inter-pod Ethernet fabric. |
| **Shuffle** | The all-to-all exchange between map and reduce phases. |
| **Spine** | Aggregation switch with no servers attached; forwards between leaves. |
| **STP** | Spanning Tree Protocol. Layer 2 loop prevention by disabling redundant links. |
| **Straggler** | The slowest participant in a synchronous collective, which sets the pace for all. |
| **Superspine / spine block** | Third tier connecting multiple pods. |
| **5-tuple** | Source IP, destination IP, protocol, source port, destination port. The definition of a flow. |

---

## Sources and further reading

- RFC 7938 — *Use of BGP for Routing in Large-Scale Data Centers*
- *Jupiter Rising: A Decade of Clos Topologies and Centralized Control in Google's Datacenter Network* (SIGCOMM 2015)
- *Jupiter Evolving: Transforming Google's Datacenter Network via Optical Circuit Switches and Software-Defined Networking* (SIGCOMM 2022)
- Google Cloud blog — *The evolution of Google's Jupiter data center network*
- *B4: Experience with a Globally-Deployed Software Defined WAN* (SIGCOMM 2013)
- Companion notes in this project: `jupiter-fabric-primer.md`, `ml-capacity-build-primer.md`, `npi-process-primer.md`
