# ADR-0002: CPU requests with no CPU limit; memory request equals memory limit

## Status

Accepted, with explicit conditions for reversal (see bottom). Applies to
`values.yaml` and `values-prod.yaml` as shipped in this chart.

## Decision

Every container gets a CPU `request` and no CPU `limit`. Every container
gets a memory `request` that equals its memory `limit`.

This is a deliberate asymmetry between the two resource types, not an
oversight. It follows from one fact about the Linux kernel: **CPU is
compressible, memory is not.**

## Context: why the two resources need different treatment

### CPU limits don't protect anything — they just add latency

A Kubernetes CPU limit is enforced by the kernel's CFS (Completely Fair
Scheduler) bandwidth controller via cgroups. A limit of `500m` becomes a
quota: in each scheduling period (100ms by default), the container's
cgroup gets 50ms of CPU time. Once a container's threads have collectively
burned through that 50ms *within the 100ms window*, every thread in that
cgroup is paused — throttled — until the next period starts. This happens
**even if the rest of the node is sitting idle.** The kernel does not ask
"is there spare capacity on this box"; it asks "has this cgroup exceeded
its quota for this period," full stop.

This produces a specific, well-documented failure mode: a multi-threaded
or bursty process can look almost idle on an average-CPU-utilization
dashboard (say, 15% average over a minute) while still being throttled
dozens of times a second, because its *bursts* — a request handler doing
a flurry of work across several threads — exhaust the 100ms quota in a
few milliseconds, then wait out the rest of the period doing nothing.
The result is p99/p999 latency spikes that are invisible in average CPU
graphs and only show up as request latency or, in Kubernetes, as the
`container_cpu_cfs_throttled_periods_total` metric. Several companies
(Indeed, Buffer, and others) have published exactly this postmortem:
"our CPU utilization graphs looked fine, our latency didn't." A limit
doesn't protect the node from the container; it protects nothing and
costs latency.

### What a CPU request actually guarantees

A CPU request isn't a reservation of dedicated cores. It sets the
cgroup's CPU *shares* (`cpu.shares` under cgroup v1, `cpu.weight` under
v2) — a relative weight, not an absolute floor. That weight only matters
**once the node's CPU is fully contended.** If the node has idle
capacity, every container can use as much of it as it wants, request or
no request, limit or no limit. Only when the sum of everyone's demand
exceeds the node's actual CPU does the kernel start dividing time
proportionally to each cgroup's shares — at that point, a container with
a 200m request and one with a 400m request get CPU time in roughly a 1:2
ratio, not a hard 200m ceiling and floor.

So: **no limit means a container can burst freely into idle capacity when
it's available, and gets a fair proportional share (never zero) when it
isn't.** That is exactly the behavior wanted for most request-serving
workloads, which are bursty by nature and mostly not simultaneously
peaking.

### Memory has no equivalent safety valve

There is no memory throttle. A cgroup's memory limit
(`memory.limit_in_bytes` / `memory.max`) is a hard ceiling; the kernel
cannot "slow down" a process's memory allocation the way it can pause its
CPU time. The only enforcement mechanism is the OOM killer: the instant a
cgroup would exceed its limit, something in that cgroup gets killed,
immediately, with no graceful degradation. Exceeding a CPU limit costs
latency. Exceeding a memory limit costs the pod.

That asymmetry — safe-to-fail-slow (CPU) versus unsafe-to-fail-fast
(memory) — is the entire reason this ADR treats the two resources
oppositely.

### Why memory `request` must equal `limit`, specifically

This isn't just "set a limit," it's "make request and limit the same
number," and that's a separate decision from "have a limit at all."

The scheduler bin-packs nodes using **requests**, not limits. If memory
`request` is set lower than `limit` (e.g., request 128Mi, limit 256Mi),
the scheduler can legally pack a node full of pods whose *requests* fit
but whose *limits* — their real worst-case usage — do not. If enough of
those pods simultaneously grow toward their limit, the node itself runs
out of memory before any single container hits its own limit cleanly.
That's node-level memory pressure: the kubelet starts evicting pods by
policy, or in the worst case the kernel OOM killer fires against
*whatever process on the node* looks like the best candidate — which
might not even be the container that caused the problem. Setting
`request == limit` for memory means the scheduler's bin-packing decision
already accounts for each container's real ceiling, so a node can never
be scheduled into a state where combined potential usage exceeds its
actual memory. Any single container that leaks or spikes gets a clean,
isolated OOMKill of *that* container — not a noisy, unpredictable
node-wide event.

### A note on QoS class, because it will come up

Kubernetes' `Guaranteed` QoS class requires *every* resource (CPU and
memory) to have request == limit. This chart's containers do not qualify
for `Guaranteed` — they're `Burstable`, because CPU has a request but no
limit. That's a real, known trade-off of this decision: under severe
node-level memory pressure, the kubelet's eviction order considers
`Guaranteed` pods last. This chart's pods rank below a hypothetical fully
`Guaranteed` pod on that axis. In practice this is mitigated by the
memory `request == limit` choice above — eviction scoring is driven
heavily by how far a pod's actual usage exceeds its *request*, and a pod
that can't exceed its memory request without being OOMKilled first is
already close to the best-case position available outside `Guaranteed`.
It's a real cost of this decision, not a hidden one.

## Consequences

**Positive**

- No artificial CPU throttling latency on workloads that are bursty but
  not sustained — the common case for request-serving HTTP services.
- Containers can absorb short traffic spikes using a node's idle capacity
  without needing HPA to react first.
- No node can be scheduled into a state where combined memory limits
  exceed real capacity — memory pressure incidents are isolated to the
  single container that actually misbehaves.
- Simpler mental model for developers: one memory number to reason about
  ("how much does this need"), not two.

**Negative**

- A CPU-greedy or misbehaving container (bug, unexpected load, compromised
  dependency) can consume all of a node's idle CPU with no ceiling, and
  nothing in this chart alone stops it — see the counter-argument below.
- This chart forfeits `Guaranteed` QoS class; see the note above.
- Capacity planning has to reason about *requests* as the real ceiling for
  scheduling purposes, and separately monitor actual usage/throttling —
  it's not "read the limits column and you know the worst case," because
  there effectively is no CPU worst case per container.

## The other side of this argument, argued straight

The case above is not the only reasonable position, and depending on the
organization, "always set both limits" is the more defensible default.
Arguing it honestly:

**Noisy-neighbor CPU starvation is real, and "shares kick in under
contention" undersells the damage.** The proportional-share mechanism
only activates once the node is *already* saturated — by definition,
after the damage has started. A container with a bug (an infinite retry
loop, a runaway job, a compromised dependency doing something it
shouldn't) can burn 100% of a node's idle CPU for as long as it takes for
something else to notice and react — an autoscaler scaling out, an
on-call engineer restarting the pod, a liveness probe finally timing out.
During that window, every other pod on the node is degraded, and
"degraded" for a latency-sensitive service can mean breached SLAs before
any alert fires. A hard CPU limit turns an incident that affects an
entire node into an incident contained to one container. That containment
value is real and this ADR's argument doesn't erase it — it just says the
cost (throttling latency, always, for everyone) isn't worth paying to buy
that protection by default.

**Cost and capacity predictability suffer without limits.** Without a CPU
ceiling, "how much CPU could this node's workloads consume at worst" has
no answer — it's bounded only by the node's total core count, not by
anything declared per-container. That makes capacity planning a matter of
watching actual usage trends and hoping they hold, rather than being able
to compute a hard number from the manifests. For an organization doing
serious capacity forecasting — buying reserved instances, sizing node
pools months in advance, negotiating budget against projected infra
spend — "we don't actually know the ceiling" is a genuinely uncomfortable
answer to give a finance team.

**Chargeback in multi-tenant clusters breaks down without limits.** If
several teams or cost centers share a cluster and are billed or budgeted
against what they *request*, a team whose pods routinely burst well
beyond their declared requests is, in effect, consuming (and potentially
degrading) capacity that was implicitly promised to someone else, for
free. Limits are sometimes the actual chargeback enforcement mechanism:
"you requested this much, you are also capped at this much" is a much
easier policy to bill against and defend to another team than "you
requested this much, but you're welcome to whatever's idle, subject to
change without notice." This ADR's justification — CPU is compressible,
so a limit only costs latency — is true from the kernel's point of view
and irrelevant from a cost-allocation point of view.

**Compliance and audit requirements sometimes just want the field set.**
Some regulated environments — and this project is explicitly framed as a
fintech-relevant portfolio piece, so this isn't hypothetical — have
policies (sometimes enforced by tools like OPA/Gatekeeper or Kyverno
admission policies) that require every container to declare a resource
ceiling as an auditable control, independent of the underlying kernel
mechanics. An auditor's checklist item is usually "does every workload
have a CPU limit," not "does the team understand CFS bandwidth
throttling." Explaining that a limit "only throttles, it doesn't actually
cap the failure blast radius the way you think" is correct and will not
move a compliance requirement that exists to produce a specific,
inspectable artifact. In that context, arguing kernel semantics against a
policy checkbox is a losing and slightly beside-the-point argument.

## When I'd reverse this decision

- **A genuine multi-tenant chargeback requirement** — multiple teams or
  cost centers sharing this platform, with a real business need to cap
  (not just estimate) what each is allowed to consume. I'd add CPU limits
  and accept the throttling-latency cost as the price of enforceable
  fairness.
- **A documented compliance mandate** requiring hard resource ceilings as
  an auditable control, regardless of the throttling nuance. In a
  regulated environment, a compliance requirement beats a performance
  argument — that's not a close call.
- **Repeated, evidenced noisy-neighbor incidents** where a CPU-greedy
  workload measurably degrades co-located latency-sensitive services
  faster than existing guardrails (HPA, PodDisruptionBudgets, priority
  classes, node-level alerting) can react. At that point the fix would
  most likely be scoped narrowly — CPU limits on the specific class of
  workload proven to cause the problem (e.g., batch/background jobs on
  shared node pools), not a blanket policy change for every service.
- **Chronic node-level CPU/memory pressure caused by requests that
  consistently underestimate real usage.** This is a signal the requests
  themselves are miscalibrated first — the fix is usually "fix the
  requests," not "add limits" — but if usage is inherently too bursty for
  any static request to model well, limits (accepting the latency cost)
  become the practical mitigation.

## Alternatives considered

- **Guaranteed QoS everywhere** (CPU and memory both request == limit):
  rejected as the default because it pays the CPU throttling cost on
  every workload, all the time, for a containment benefit that's only
  relevant to a subset of failure modes (see counter-argument above). Not
  ruled out permanently — see reversal conditions.
- **No requests or limits at all (BestEffort):** rejected outright. This
  removes the scheduler's ability to bin-pack sensibly and puts these pods
  first in line for eviction under any node pressure, for zero benefit
  over `Burstable`.
