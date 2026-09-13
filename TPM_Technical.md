What is a GPU, TPU?

[Graphics Processing Unit (GPU)](https://www.intel.com/content/www/us/en/products/docs/processors/what-is-a-gpu.html) is a specialized electronic circuit designed to process and render images, video, and animations at high speeds. Unlike a standard central processing unit (CPU) that handles general tasks one after another, a GPU uses **parallel processing** to run thousands of calculations at the same time.  
**GPU:** The specific microchip that performs the mathematical calculations.

**Graphics Card:** The larger physical circuit board that houses the GPU, along with its own cooling fans, ports, and memory (VRAM).

**Integrated vs. Discrete:** A GPU can be built directly into the main processor or motherboard (integrated), or exist as a separate, powerful add-in board (discrete).

The **Tensor Processing Unit** (**TPU**) is a [neural processing unit](https://en.wikipedia.org/wiki/Neural_processing_unit) (NPU) [application-specific integrated circuit](https://en.wikipedia.org/wiki/Application-specific_integrated_circuit) (ASIC) developed by [Google](https://en.wikipedia.org/wiki/Google) for [neural network](https://en.wikipedia.org/wiki/Artificial_neural_network) [machine learning](https://en.wikipedia.org/wiki/Machine_learning).

**Eighth generation TPU**

On April 22, 2026, Google announced the eighth generation of its Tensor Processing Units, consisting of two specialized chips: the TPU 8t and the TPU 8i. This marks the first time Google has bifurcated its TPU architecture into separate training and inference-optimized designs. Both chips are hosted on Google's custom Arm-based [Axion](https://en.wikipedia.org/wiki/Google_Axion?action=edit&redlink=1) CPUs and utilize 4th-generation liquid cooling.

**TPU 8t**

The TPU 8t ("Training") is optimized for large-scale pre-training of frontier models and embedding-heavy workloads. It delivers 12.6 [FP4](https://en.wikipedia.org/wiki/Floating-point_arithmetic#Reduced-precision_formats) PFLOPs of peak performance and features 216 GB of HBM3e memory with 6,528 GB/s bandwidth. It uses the Virgo Network fabric, allowing it to scale up to 9,600 chips per "superpod," delivering 121 FP4 ExaFLOPs of compute performance.

**TPU 8i**

The TPU 8i ("Inference") is designed for high-speed serving, AI agents, and long-context reasoning. It delivers 10.1 FP4 PFLOPs of peak performance and features 288 GB of HBM3e memory with 8,601 GB/s bandwidth. It includes 384 MB of on-chip SRAM, a three-fold increase over the previous generation. The TPU 8i introduces a "Boardfly" network topology and a Collectives Acceleration Engine (CAE) which reduces synchronization latency by five times

NETWORK CAPACITY PLANNING – (FORECASTING)

*What is Network capacity planning?*

Network capacity planning is the discipline of forecasting future traffic demand and provisioning the network ahead of it — with enough margin to survive a failure state, not just meet average load.

*Why we do it?* — so growth and outages don't turn into performance or availability incidents.

*Core components:*

- *Forecasting demand*: using historical traffic trends plus known future growth (new workloads, migrations, roadmap events) to predict what the network will need to carry.

- *Assessing current supply:* understanding what capacity exists today across all paths, and how much of it is usable versus theoretical (accounting for failure states, not just raw link speed).

- *Finding and closing the gap:* identifying where demand will exceed supply, and provisioning ahead of that point — factoring in vendor lead time, so the fix lands before the shortfall does, not after.

- *Designing for resilience, not just volume*: it's not only "will we have enough bandwidth" but "will we still have enough bandwidth if a circuit fails" — which is why N-1 and peak-overlap analysis are core to it, not optional extras.

**<u>Prerequisites — what you need before you can plan anything</u>**

- **Traffic data:**  
  For each workload/application, get historical utilization on existing links (on-prem WAN, existing cloud connections)  
  You need this at fine enough granularity t**o see bursts**, not just 5-minute averages.

- **Growth signal, split into two kinds**:  
  1. organic trend (the workload growing 2% a month) and  
  2. step-function growth (a new system going live, a migration wave landing).  
  These need separate treatment — organic growth extrapolates, step functions come from the roadmap, not the traffic history.

- **Topology and circuit inventory:** what connects where, current speeds, how many paths, whether you're running N or N+1 today, and RTT/latency baseline between sites.

- **SLA/SLO requirements per workload**: trading and risk pipelines might need sub-millisecond and zero tolerance for loss; batch ETL can absorb latency and even brief degradation. This matters because it determines **which workloads drive your worst-case sizing.**

- **Constraints that aren't traffic-related**: **vendor lead time** for new circuits (this is often the binding constraint — 60-120 days for a new Direct Connect circuit means your planning horizon has to be longer than your data horizon), colo cross-connect and power/space availability, **cost per Mbps by circuit type**, and **compliance boundaries** that restrict routing choices.

# The core loop is: demand → corrected capacity requirement → compare to supply → find the gap → decide remediation → validate against reality

# **<u>How it's actually done, step by step</u>**

# Pull flow data from networking team (on-prem, VPC flow logs and Direct Connect metrics in cloud) **over a window that captures your real peak cycles** — for financial workloads that usually means month-end and quarter-end, not just "last 30 days."

# Break utilization down by workload/team, not just by link, so growth can be modeled per-workload rather than as one blended trend.

# Build the demand forecast: organic trend per workload plus known step-functions from the roadmap

# Run peak overlap analysis across workloads to get the true aggregate peak rather than the sum of individual peaks - Instead of taking each workload's individual max and adding them, you look at the *combined* traffic on the shared link, minute-by-minute (or at whatever granularity your flow data supports), across a real observation window — and find the single highest point of that combined curve.

# Apply the N-1 correction — model what happens to that aggregate peak if your largest circuit fails, and size to survive that state. The question N-1 forces you to ask is: **what happens the moment you lose your single largest circuit? What "N-1" means as a term** N is the total number of paths/circuits you have. N-1 means "one fewer than that" — i.e., the state you're in after losing your single largest one. The N-1 correction is the discipline of sizing not against your full provisioned capacity, but against what survives after that loss **How you actually apply it**

# Take your true aggregate peak from the overlap analysis (7.8 Gbps).

# Identify your largest single circuit or path (5 Gbps in this example).

# Subtract that circuit from your total supply to get your N-1 supply (10 - 5 = 5 Gbps).

# Check: does N-1 supply (5 Gbps) cover the aggregate peak (7.8 Gbps)? Here it doesn't — that's your gap.

# Size your remediation so that the *remaining* circuits alone, not the total, can carry the full peak. In this example you'd need the surviving circuit(s) to cover 7.8 Gbps on their own — meaning you either need a bigger second circuit, a third diverse path, or both.

# Compare the corrected demand number against current provisioned supply to get the actual gap.

# Combine the utilization trend with vendor lead time to find your trigger point — the date/utilization level at which you need to have already started the procurement process.

# Choose the remediation: ***Add bandwidth on the existing circuit**, *This means increasing the speed of a circuit you already have — say upgrading a 5 Gbps Direct Connect port to 10 Gbps, rather than standing up anything new. **What it fixes**: pure volume shortfall. If your N-1 supply is short of your aggregate peak simply because the pipe isn't big enough, this is the most direct fix. **What it doesn't fix**: it does nothing for resilience. You still have the same number of physical paths, so the same single point of failure exists — you've just made it a bigger single point of failure. **Practical mechanics**: usually the fastest lever if the underlying physical medium (fiber, port) supports a higher speed tier — sometimes it's a config/contract change rather than new physical work, which is why it's often the shortest lead-time option. **When you'd pick it**: lead time is tight, and the gap is a volume gap, not a failure-survival gap.* **Add a parallel circuit (and if so, watch LAG hashing behavior so you're not creating an uneven split), **This means standing up a second physical circuit on the same logical path and bonding it with the first so traffic spreads across both — a Link Aggregation Group.*

# ***What it fixes**: adds both capacity and some redundancy — if one link in the LAG fails, the other keeps carrying traffic (at reduced total capacity).*

# ***The catch — hashing behavior**: LAG doesn't split traffic evenly by default. It uses a hash function (typically based on source/destination IP and port, sometimes MAC) to decide which physical link a given flow rides on. A single flow always rides one link, never split across both. So if you have a few very large flows (e.g., one massive backup replication stream) alongside many small ones, the hash can land that one huge flow entirely on Link A, leaving Link A near-saturated while Link B sits mostly idle — even though your "total LAG capacity" on paper looks like both links combined.*

# ***Why this matters for capacity planning specifically**: your usable capacity from a LAG is not simply the sum of the two link speeds — it's constrained by how well your actual flow distribution hashes across the links. Two 5 Gbps links in a LAG do not reliably give you 10 Gbps of usable capacity for a workload dominated by a few large flows; you have to look at flow cardinality and size distribution, not just aggregate throughput, to know what a LAG will really deliver.*

# ***When you'd pick it**: you need both more capacity and some fault tolerance quickly, and your traffic is diverse enough (many small/medium flows) that hashing will actually spread the load — and critically, when the underlying physical path itself isn't the risk you're worried about (see \#3 for when it is).*

# ***Add a second diverse path, or** This means a genuinely separate physical route — different provider, different conduit/fiber path, sometimes a different physical building or point of entry — not just a second circuit riding the same physical corridor as the first.* ***When you'd pick it**: the risk you're protecting against is a physical/correlated failure, not just a volume shortfall — for example, if your entire hybrid architecture currently depends on one carrier's fiber into one colo facility, and that's an unacceptable single point of failure for a workload like trading or risk. This is usually the answer to "how do you eliminate a true SPOF," not "how do we get more bandwidth by Friday." **Shift load via traffic engineering instead of adding capacity at all. ***This means changing *when* or *where* traffic flows, rather than growing the pipe — rerouting non-critical workloads to a different path, scheduling batch/backup jobs outside the peak window, or applying QoS to prioritize critical traffic and throttle the rest during contention. **What it fixes**: it can close a gap without spending anything on new infrastructure, and it can be implemented almost immediately — no procurement, no vendor lead time.

# **What it doesn't fix**: it doesn't grow the actual ceiling. If total demand keeps growing, this buys time, it doesn't solve the underlying trend. It's also more operationally fragile — it depends on people/schedules staying disciplined about what runs when, and it can degrade non-critical workloads' experience as the trade-off.

# **When you'd pick it**: lead time is the binding constraint and nothing else can land in time, or the gap is actually caused by *poor traffic distribution* rather than genuine insufficient capacity — e.g., you discover the backup job and the risk batch job happen to overlap by accident, and simply moving one of them fixes the peak without touching infrastructure at all. Before concluding a cross-region link "needs more bandwidth," you'd verify window scaling is enabled on both endpoints and that the configured window size is actually sized against the real BDP of that path — not just left at an OS default that predates the link's actual speed and distance. That's a cheap, fast diagnostic step that can save you from provisioning (and paying for) bandwidth that a protocol setting was quietly preventing you from using anyway. "BDP isn't a step in the process — it's a check I run at data collection and at remediation selection, because it determines whether low utilization on a cross-region link means low demand or means the link is protocol-limited. Getting that wrong means either under-forecasting demand or spending on bandwidth that doesn't actually fix the bottleneck." This is another concrete, checkable lever at Step 7 (remediation) that's cheaper and faster than adding bandwidth: if a transfer tool is only using a single stream and hitting a window ceiling, switching it to multiplex across multiple streams can unlock throughput that already exists on the link — no procurement, no new circuit, just a configuration or tooling change. It's the same category as window scaling — fix the protocol behavior before assuming you need more physical capacity.

# After provisioning, re-baseline against actual traffic and check your model's accuracy — this is the step people skip, and it's the one that turns "we did math" into "we validated the math."

#  DESIGN PRINCIPLES – 

# **You design for peak, not average.** Average utilization is close to meaningless for capacity planning — it hides the moments that actually break things. And "peak" itself needs the right measurement window; a microburst that saturates a link for 200ms disappears entirely in a 5-minute average. 

# **Sum-of-peaks is a trap.** Naive planning adds every workload's individual peak together and assumes they all happen at once. In practice, workloads peak at different times, so the actual system-wide peak is usually well below that sum — this is the peak overlap analysis that got you to ~35% below naive sizing on the Bridgewater program. 

# **Plan for the failure state, not the steady state — the N-1 principle.** If you have two circuits, you don't size each for 50% of peak; you size so that one circuit alone can absorb 100% of peak, because that's the state you're in the moment the other one fails. Usable design capacity is capped by what's left after losing your single largest path. 

# **Distance and window size cap throughput independent of link speed** — the bandwidth-delay product. A high-speed, high-latency hybrid link can be throughput-starved by TCP window behavior long before the physical link is saturated, which matters a lot for cross-region hybrid connectivity. 

# **You provision off a trigger, not off the ceiling.** Because procurement has lead time, you set a threshold (say, sustained 70% utilization) that, combined with your growth rate, guarantees new capacity lands before you actually hit 100%. 

# **Headroom has to be justified, not just round.** Every percentage point of headroom should map to something specific — failure tolerance, burst tolerance, known growth runway — or it reads as waste when someone asks you to defend the number. **Decompose headroom into named components that each map to a real risk.**

# Instead of one blended number, you build headroom as a sum of specific, individually-justified pieces, each answering "what specific thing does this percentage protect against." Here's how you'd actually construct it:

# **Component 1 — Burst tolerance**

# Look at your flow data at fine granularity (1-minute or sub-minute, not 5-minute averages) and find how far short bursts exceed your measured "peak." If your 5-minute average peak is 8 Gbps, but 1-minute bursts within that window hit 9.2 Gbps, that's a real, measured 15% burst margin — not a guess. This is defensible because you can point to the actual burst data that produced it.

# **Component 2 — Growth runway**

# This should map to your actual forecasted growth rate times your provisioning cycle length — not a flat guess. If organic growth is running at ~3%/quarter and your circuit upgrade lead time plus internal approval process means you can't react faster than roughly two quarters, you need at least ~6% of headroom just to survive the gap between "we notice we need more" and "we actually have more." That's a defensible 6% growth-runway margin, derived from real lead time, not intuition.

# **Component 3 — Failure/resilience margin**

# This is different from N-1 (which you've already applied to guarantee survival) — this is extra margin within the N-1 state to avoid running the surviving circuit at a razor's-edge 100% during an actual outage, which itself causes queuing and packet loss even if you're technically "not breaching capacity." A common, defensible standard here is keeping the N-1 state under ~80% utilization, which gives you a ~20% operational margin within the failure state specifically to preserve performance, not just avoid an outright breach.

# **Component 4 — Known step-function events not yet in the baseline**

# If there's a specific known future event — a migration wave landing next quarter, a new workload onboarding — that adds a defined, sourced amount, not a percentage guess. E.g., "the trading desk's new market data feed is confirmed for Q3, adding an estimated 800 Mbps based on their own sizing." This is headroom with a name and a date attached, not a vague growth cushion.

# **How you present the total**

# Instead of "we added 20% headroom," you say: "of our total headroom, roughly 15 points come from measured burst behavior in the flow data, about 6 points come from growth runway matched to our actual procurement lead time, and the rest is a specific, named event landing next quarter — nothing here is a round-number guess."

# **What is TCP WINDOW SIZE -  **TCP is a reliable protocol. Every byte sent needs to eventually be acknowledged (ACK'd) by the receiver. But TCP doesn't send one byte, wait for the ACK, send the next byte — that would be absurdly slow. Instead, it sends a *window* of data — a batch — before it needs to pause and wait for acknowledgments to catch up. The size of that window is called the **TCP window size**.

# **TCP WINDOW CONSTRAINT.** -Here's the key constraint: the sender can only have one window's worth of unacknowledged data in flight at any moment. Once the window is full, it has to stop and wait for ACKs to come back before sending more — even if the physical link has tons of spare capacity sitting idle.

# **Because of this constraint, distance between networks can become a throughput problem - Round-trip time (RTT) -** The time it takes for a byte to travel to the receiver and for its ACK to travel back is the round-trip time (RTT) **On a short local link**, RTT might be under 1ms — ACKs come back almost instantly, so the window refills constantly and the link stays busy **But on a cross-region hybrid link** — say on-prem in the US to a cloud region in Europe — RTT might be 80-100ms. During that 80-100ms round trip, the sender can't send anything beyond what's already in the window — it's just waiting. This blocks throughput. So the *effective* throughput isn't determined by link speed alone, it's determined by **how much data fits in one window, divided by how long you have to wait for that window to clear.** So in one RTT, at most one window's worth of data gets delivered. That gives you a natural rate:

**BDP ( DATA) = Bandwidth × RTT**  
This tells you the *maximum amount of data that can be "in flight" on the link at any given moment* to fully utilize it**So, BDP = bandwidth \* RTT**

Earlier Standard TCP window size was = 64KB  
So depending on what RTT was max throughput was limited to = 64 KB / 80ms (rtt) = 65,536 bytes × 8 bits / 0.080 sec ≈ **6.5 Mbps.** As networks scaled up in speed and hybrid/long-distance architectures became common, this fixed ceiling became a real, practical bottleneck — not a theoretical one.

**TCP WINDOW SCALING -** is an extension that lets TCP effectively use a much larger window than 16 bits would normally allow — up to roughly **1 GB** .It does this by adding a **scale factor** that both sides negotiate during the initial TCP handshake (the SYN/SYN-ACK exchange). Instead of the 16-bit field being read literally, it gets multiplied by 2^(scale factor).  
**Two things have to be true for it to actually work-  
It has to be enabled and negotiated by both endpoints** —It has to be enabled and negotiated by both endpoints. If either side (say, an old legacy on-prem system) doesn't support or hasn't enabled window scaling, the connection falls back to the old 64 KB ceiling, silently. This is a common real-world trap: one modern cloud-side system and one legacy on-prem system, and the connection quietly negotiates down to the lowest common denominator.  
**It has to actually be sized correctly for your BDP**, not just "turned on." Window scaling being enabled doesn't automatically mean the window is sized right — this is exactly the "2-4 MB even with window scaling enabled but not well-tuned" case from before. The OS or application needs to be configured with a window size that matches your actual bandwidth × RTT, or you're still leaving throughput on the table even with scaling technically active.

**What you actually do about it**

- **TCP window scaling** (RFC 1323) needs to be enabled, and the window size needs to be set based on your actual BDP, not left at OS defaults.

- **Multiple parallel TCP streams** for a single data transfer can work around a single-stream window limit — this is why tools like large-scale replication or backup software often multiplex across many connections rather than one.

The core limitation this addresses

- We established: a single TCP connection's throughput is capped by window size ÷ RTT. Even with window scaling enabled and well-tuned, you're still bound by whatever that *one connection's* window allows. If for some reason you can't get a single stream's window large enough — maybe there's a network device in the path that doesn't handle large windows well, maybe the OS/application has a hard ceiling, maybe there's packet loss that keeps forcing the window back down (more on that below) — a single TCP stream simply cannot exceed a certain throughput on a high-RTT path, no matter what you do to it.

- **The fix:** split the data transfer across **multiple TCP connections running in parallel**, each carrying a portion of the total data. Each individual stream is still capped at roughly 400 Mbps by its own window/RTT limit — but now you have, say, 20 streams running simultaneously, each independently limited to ~400 Mbps, and their combined throughput adds up:

- **Why this is a legitimate engineering solution, not a hack**  
  Each TCP stream operates independently — it has its own window, its own sequence numbers, its own congestion control state. The network link itself doesn't care whether the packets it's carrying belong to one connection or twenty; it just carries bits. So multiplexing isn't tricking the network, it's exploiting the fact that the *bottleneck is per* **connection, and there's no rule that a data transfer has to go over exactly one connection.**

- The trade-off to be aware of **- More streams isn't free** — it adds connection overhead (each TCP handshake, more state to track), and past a certain point you get diminishing returns or can even start contending with yourself for buffer/CPU resources on the endpoints. So in practice it's tuned (a reasonable number of parallel streams, not an arbitrary large number), not just "more is always better."

# Because bandwidth is always quoted in bits per second (Gbps, Mbps), but file sizes, buffer sizes, and window sizes are always quoted in bytes (MB, KB). Any time you're moving between a bandwidth number and a data-size number — exactly what BDP requires you to do — you have to cross that bits/bytes line, and it's an easy place to be off by a factor of 8 if you're not deliberate about it.

# A concrete example

# Say you have a 10 Gbps Direct Connect link between on-prem and a cloud region, with 80ms RTT (realistic for a cross-region hybrid connection).

# BDP = 10 Gbps × 80ms = 10,000,000,000 bits/sec × 0.080 sec = 800,000,000 bits = 100 MB

# That means: to fully utilize this 10 Gbps link, you need 100 MB of data in flight at any given moment, unacknowledged, sitting in the pipe

# \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

# **Google Cloud TPM — Program Management Metrics Reference**

*Prepared for: Technical Program Manager III, Networking, Google Cloud interview prep*

| **Category**                     | **Metric**                                             | **Definition**                                                                                                                                               | **Lifecycle Stage**                                                      | **How It's Tracked**                                                                                                       | **Success / Failure Criteria**                                                                                                                                                        |
|----------------------------------|--------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Reliability & Availability**   | **SLO/SLA compliance & error budget**                  | The allowable margin of unreliability before breaching a Service Level Objective/Agreement; the “budget” of failure a system can tolerate in a given period. | Ongoing — Execution & Operations; reviewed in weekly/monthly ops reviews | Error budgets built on real SLO data, monitored via dashboards and alerting policies                                       | Success: error budget not exhausted, SLO met (e.g., 99.9%+ uptime). Failure: error budget burn rate triggers feature-freeze / reliability-first mode.                                 |
|                                  | **Uptime / Availability %**                            | The percentage of time a system or service is operational and accessible to users over a given period.                                                       | Ongoing — Operations phase                                               | Automated monitoring dashboards, incident logs                                                                             | Success: meets or exceeds committed uptime (99.9%+). Failure: drops below threshold, triggers root-cause review.                                                                      |
|                                  | **MTTR (Mean Time to Repair)**                         | The average time taken to detect, diagnose, and resolve an incident from occurrence until service is restored.                                               | Post-incident — Operations / Incident Response                           | Tracked via incident management tooling, connecting monitoring data to business outcomes                                   | Success: MTTR trending down release-over-release. Failure: MTTR increasing or exceeding SLA-committed resolution time.                                                                |
|                                  | **MTBF (Mean Time Between Failures)**                  | The average operating time between one failure and the next, indicating overall system stability.                                                            | Ongoing — Operations phase                                               | Incident frequency tracked over time via monitoring/logging systems                                                        | Success: interval between failures lengthening. Failure: shortening intervals signal systemic instability.                                                                            |
| **Capacity & Utilization**       | **Resource utilization (compute/network/storage)**     | The proportion of available infrastructure resources (CPU, memory, bandwidth, storage) actively being consumed at a given time.                              | Planning (forecasting) + Execution (real-time tracking)                  | Forecast-based alerting and historical metric dashboards for CPU, disk, memory, and throughput                             | Success: utilization stays within healthy buffer (e.g., 60–80%). Failure: hits capacity ceiling unexpectedly, causing outages.                                                        |
|                                  | **Capacity headroom vs. peak demand**                  | The buffer of unused capacity retained above forecasted peak demand to absorb unexpected traffic spikes.                                                     | Planning — quarterly/seasonal capacity reviews                           | Historical utilization metrics inform whether more headroom is needed before a traffic spike                               | Success: headroom sufficient to absorb forecasted peak + buffer. Failure: reactive scaling after a spike already caused degradation.                                                  |
| **Network Performance**          | **Latency, packet loss, jitter**                       | Latency = time for data to travel between two points; packet loss = % of data packets that fail to arrive; jitter = variability in latency over time.        | Ongoing — Execution / Operations                                         | Network telemetry tools, adaptive thresholds auto-updating for growth-dependent metrics like throughput                    | Success: within defined performance thresholds (e.g., sub-X ms latency). Failure: breach triggers alert and remediation workflow.                                                     |
|                                  | **Alert MTTD (Mean Time to Detect)**                   | The average time between an issue first occurring and it being detected by monitoring systems or personnel.                                                  | Ongoing — Operations phase                                               | Smart alerts with severity levels to improve MTTD and MTTR                                                                 | Success: issues detected before customer-facing impact. Failure: issue surfaces via customer complaint before internal detection.                                                     |
| **Program Delivery (TPM-owned)** | **Milestone/roadmap adherence**                        | The degree to which a program hits its planned deliverable dates and sequence as defined in the roadmap.                                                     | Planning through Closure — tracked at every phase gate                   | Program trackers, roadmap reviews, measurable milestones tracked against clear KPIs                                        | Success: milestones hit on/ahead of schedule. Failure: schedule variance beyond agreed tolerance, requiring re-baseline.                                                              |
|                                  | **Risk register health**                               | A structured log of identified program risks, their likelihood/impact, mitigation owners, and current status.                                                | Planning through Execution — reviewed in governance cadences             | RAID logs, risk registers with aging/status flags                                                                          | Success: risks closed or mitigated before impact date. Failure: risks aging past mitigation deadline, escalating to issues.                                                           |
|                                  | **Dependency resolution velocity**                     | The speed at which cross-team blockers or prerequisite deliverables are resolved so downstream work can proceed.                                             | Execution phase — cross-team coordination                                | Dependency trackers reviewed in cross-functional syncs                                                                     | Success: dependencies resolved within committed SLA between teams. Failure: blocking dependencies stall downstream milestones.                                                        |
|                                  | **Go/No-Go pass rate**                                 | The percentage of launch readiness reviews that pass defined entry/exit criteria on first evaluation.                                                        | Pre-launch — Rollout/Deployment phase                                    | Go/no-go frameworks with defined entry/exit criteria                                                                       | Success: launches pass go-criteria on first review. Failure: repeated no-go decisions signal readiness gaps.                                                                          |
| **Resource Stewardship**         | **Resource allocation efficiency (people + machine)**  | How effectively finite compute and engineering/human resources are matched to the highest-impact initiatives.                                                | Planning — prioritization; reviewed quarterly                            | Quantitative analysis translating data into actionable plans for allocating finite resources to highest-impact initiatives | Success: resources mapped to highest-priority/impact work, measurable efficiency gains. Failure: resources tied up on low-impact work while critical initiatives are under-resourced. |
| **Cost & Efficiency**            | **Cost per unit of capacity / infra spend vs. budget** | The dollar cost associated with a unit of infrastructure capacity, tracked against planned budget allocations.                                               | Planning (budgeting) + Ongoing (actuals tracking)                        | Utilization metrics revealing over-provisioned or idle resources, cost dashboards                                          | Success: spend within budget, idle resource % trending down. Failure: cost overruns or high idle-resource waste flagged in audits.                                                    |

*Note: Categories reflect common metric families used in cloud infrastructure and networking program management (reliability/SRE practice, capacity planning, network operations, and TPM delivery governance).*

What happens when a workflow crosses VPC boundaries (peering, transit gateway, or PrivateLink — know the trade-offs between these three.

### The trade-off summary, one line each 

|                  | **Peering**                  | **Transit Gateway**                         | **PrivateLink**                             |
|------------------|------------------------------|---------------------------------------------|---------------------------------------------|
| **Model**        | Direct network merge         | Hub-and-spoke router                        | Service exposure (no network merge)         |
| **Transitive**   | No                           | Yes                                         | N/A (not routing-based)                     |
| **Blast radius** | Full VPC-to-VPC reachability | Full reachability (scoped via route tables) | Single service only                         |
| **Scaling pain** | Mesh explosion (N²)          | Cost/bandwidth at hub                       | Endpoint sprawl per service                 |
| **Best for**     | Few VPCs, point-to-point     | Many VPCs, hybrid hub                       | Cross-org service sharing, strict isolation |

### VPC Peering **How it works:** Direct network-level connection between exactly two VPCs. Traffic stays off the public internet, routed via private IPs, and you add routes in each VPC's route table pointing at the peering connection.

**Trade-offs:**

- **No transitivity** — if VPC A peers with B, and B peers with C, A cannot reach C through B. You'd need a full mesh: for N VPCs, that's N(N-1)/2 peering connections. This is the single biggest scaling limitation.

- Cheapest and lowest-latency option — it's just routing, no intermediate hop/device.

- No bandwidth bottleneck — scales with the underlying network, not a shared appliance.

- CIDR ranges must not overlap between peered VPCs.

- Good fit: small number of VPCs, or specific point-to-point relationships (e.g., a shared-services VPC peered individually with a couple of others).

Transit Gateway (AWS) **How it works:** A central hub router that many VPCs (and on-prem via Direct Connect/VPN) attach to. Acts like a regional router — one connection per VPC into the hub instead of a mesh.

**Trade-offs:**

- **Solves transitivity** — A, B, and C can all reach each other through the hub with N connections instead of N(N-1)/2. This is the whole reason it exists at scale.

- Centralizes route control — one place to manage routing policy, segmentation (via route tables within TGW itself, so you can still isolate certain VPCs from each other even though they're all attached).

- Adds a hop → slightly higher latency than direct peering, and it's a shared, billed resource — data processing charges per GB, plus per-attachment hourly cost. This adds up fast at scale.

- Bandwidth is TGW-limited per attachment (burstable, but not unlimited) — worth knowing if you're pushing high-throughput data pipelines (e.g., Kafka traffic) across it.

- Good fit: hub-and-spoke topologies, many VPCs, multi-account/multi-region enterprise networks, hybrid connectivity where on-prem needs to reach multiple VPCs.

### PrivateLink (AWS) — GCP equivalent: Private Service Connect

**How it works:** Fundamentally different model — it's not VPC-to-VPC routing at all. It exposes a *specific service* (typically running behind a Network Load Balancer) from a provider VPC to consumer VPCs via an ENI (interface endpoint) that gets an IP inside the consumer's VPC. No route tables, no CIDR exposure, no transitive network reachability.

**Trade-offs:**

- **Smallest blast radius by design** — the consumer only gets access to the one exposed service, not the entire provider VPC's network. This is the key differentiator from peering/TGW, which expose full network reachability (constrained further by security groups/NACLs).

- No CIDR overlap concerns at all, since there's no network merge — just a service endpoint.

- Doesn't scale as "give me general connectivity" — it scales as "give me many *services*." If you need N services exposed, you're managing N endpoints.

- Slight per-hour + per-GB cost per endpoint, but no shared-hub bottleneck like TGW.

- Good fit: SaaS-style service exposure (e.g., a shared internal API, a security/observability tool, a database-as-a-service pattern), especially across accounts/orgs where you explicitly do *not* want the consumer to see your network topology — strong architectural boundary for multi-tenant or third-party scenarios.

How would you connect two VPCs.

For exactly two VPCs, the default answer is **VPC Peering** — it's the simplest, cheapest, lowest-latency option, and peering's biggest weakness (no transitivity) doesn't matter when there are only two VPCs to begin with.  
"I'd start by **asking a clarifying question: do these VPCs need full network-level reachability, or does one just need access to a specific service in the other**?  
If it's general reachability — two VPCs, non-overlapping CIDRs, private communication — I'd use VPC Peering. It's a direct connection, no shared hub, no per-GB hub costs, and no added latency hop. I'd confirm CIDR ranges don't overlap, set up the peering connection, then update route tables on both sides to point relevant subnets at the peering connection, plus security groups/NACLs to actually permit the traffic.

If instead one VPC just needs to consume a specific service from the other — without seeing its broader network — I'd reach for PrivateLink instead, since it exposes only the service, not the VPC.

The only reason I'd reach for Transit Gateway instead of peering for just two VPCs is if I expect this to grow — e.g., I know a third and fourth VPC are coming soon. Building on TGW from the start avoids a rearchitecture later, even though it's overkill for two VPCs today.

What are stateful vs stateless systems?  
**Stateful** -A stateful system keeps track of context/history across a sequence of interactions, and uses that memory to make decisions. Firewall example: a stateful firewall remembers "I let this connection in a moment ago," so when the reply comes back, it recognizes it as part of that same conversation and automatically lets it through — you don't have to write a separate rule for the response.

**Stateless -** A stateless system treats every event/request independently, with no memory of what came before. Each packet, request, or interaction is evaluated fresh, on its own, against a fixed rule set — nothing is "remembered" from the last one. Firewall example: a stateless firewall (NACL) doesn't know that an inbound request just happened. So when the reply tries to go back out, it gets checked again from scratch — and if you didn't write an explicit rule allowing that outbound traffic, it gets blocked, even though it's just the response to something you already allowed in.

Stateful = "I remember the conversation." Allow one direction, the return trip is automatic.  
Stateless = "I have amnesia between every packet." Every direction needs its own explicit rule, every time.

This same stateful/stateless distinction shows up everywhere beyond firewalls, by the way — it's the same underlying idea behind stateful vs. stateless application servers, TCP (which is connection-oriented/stateful) vs. UDP (fire-and-forget/stateless), or REST APIs being designed as stateless by convention.
