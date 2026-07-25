# What Is This, Actually?

> **Status note (updated Day 4):** this document still describes the
> project's target design in places, not only what's built and verified
> today — but less of it than before. Step 4 ("safe test environment") and
> Step 5 (a human-approval gate before production) are now real:
> `.github/workflows/deploy.yml` is a reusable `workflow_call` workflow
> with `dev`/`staging`/`prod` as separate GitHub Environments, `dev`
> deploying automatically and `prod` gated behind that environment's
> required-reviewer protection rule. The comparison table's "Path to
> production" row is accurate now. What's still not real: **Step 3**
> (automatic security scanning) — neither `ci.yml` nor `deploy.yml` runs
> any code or container scanner, so the comparison table's "Code and
> container security scanning" row and the closing pitch line's "nobody
> can skip the scan" are both still aspirational. See
> [gap-tracker.md](gap-tracker.md) for the exact, current list.

## The problem, before any of the solution

Picture a mid-size company. Ten teams, ten small pieces of software (a payments
service, a login service, a search service, and so on). Each team ships their
own piece to production whenever they want.

Nobody told them how. So each team figured it out on their own.

Team A writes a deploy script by hand and runs it from their laptop. Team B
copies a script from a tutorial they found online and never updates it. Team
C has a script that runs as the "root" superuser inside the server, because
that was easier than figuring out permissions properly. Team D has no rollback
plan — if a deploy breaks something, someone has to manually undo it at 2 a.m.
while customers are staring at an error page.

Now the questions that matter start piling up, and nobody can answer them
cleanly:

- If a new engineer joins the payments team, do they inherit Team A's laptop
  script, Team B's stale tutorial, or do they invent a fourth way?
- If there's a security review, can anyone say — with confidence, for every
  team — "our software never runs as root, and it never will"? Or is the
  honest answer "depends which team you ask"?
- If a bad deploy goes out at 5 p.m. on a Friday, how long does it take to
  get back to the last working version? Does everyone even know how?
- Six months from now, can anyone say exactly what version of the code is
  running in production, right now, for each of these ten services?

None of this is about any one team being careless. It's what happens by
default when there is no shared, enforced way to do something as important as
"put code in front of customers." Every team invents its own answer, the
answers disagree, and the disagreements are exactly the kind of thing that
turns into an outage, a breach, or a very bad audit finding.

This project is my answer to that problem, for one specific piece of it:
getting a piece of software from a developer's laptop onto a running server,
safely, the same way, every single time.

## The analogy: a licensed electrician with a standard wiring code

Imagine a city where anyone can rewire their own house. Some homeowners hire
a licensed electrician who follows the fire code exactly. Others do it
themselves with whatever wire is in the garage. Both houses look fine from
the outside. You cannot tell which one is a fire hazard until it isn't fine
anymore.

Now imagine the city passes one rule: every new electrical job must be pulled
through a **standard, licensed process**. There's one accepted wiring pattern.
There's a checklist an inspector runs before the power gets switched on.
There's a required breaker that cuts power automatically if something draws
too much current, before it starts a fire. And critically — every job is
logged, so if a fire *does* start, you know exactly which house, which wire,
and which electrician touched it last.

The homeowner doesn't need to learn electrical code. They tell the
electrician "I want power in this room," and the standard process handles
the rest: the right gauge wire, the right breaker, the inspection, the paper
trail.

That's what this project is. It's the standard wiring code and the licensed
process for shipping software — not for one house (one service), but for
every house a company builds, applied the same way, automatically, every
time.

I'll come back to this analogy throughout: the "house" is a piece of
software, the "wiring" is the technical setup underneath it, the "breaker"
is the safety limits, and the "inspector" is the automated checks that run
before anything goes live.

## First, the plain-word definitions

Before the walkthrough, here are the terms I'll actually use, explained once,
in order.

- **Server**: a computer, usually sitting in a data center, that runs
  software and answers requests over the internet. You already know this
  one — it's the thing a website "lives on."
- **Container**: a way of packaging a piece of software together with
  everything it needs to run (code, libraries, settings) into one sealed
  box. The box runs the same way on any computer, because everything it
  needs travels with it. Think of it like a shipping container: the contents
  don't shift or leak, and any ship or truck can carry it without needing to
  know what's inside.
- **Kubernetes**: a system that takes a pile of containers and a pile of
  servers, and figures out which containers run on which servers, restarts
  ones that crash, and moves them around if a server dies. It's the
  dispatcher and traffic controller for containers. "EKS" (Amazon Elastic
  Kubernetes Service) is Amazon's version of this, running on Amazon's
  servers, so a company doesn't have to run the dispatcher itself.
- **Pod**: the smallest unit Kubernetes actually schedules — usually one
  running container, with a bit of wrapping around it. You mostly don't
  interact with pods directly; Kubernetes creates and destroys them for you.
- **Helm**: a tool that packages up all the instructions Kubernetes needs
  ("run this container, give it this much memory, expose it on this address")
  into one reusable bundle, with fill-in-the-blank settings. Without Helm,
  someone hand-writes a stack of configuration files for every single
  service. With Helm, they fill in a short form.
- **Chart**: the actual Helm bundle — the reusable template with the
  fill-in-the-blanks. This project's chart is the "standard wiring pattern"
  from the analogy: one design, reused for every service.
- **CI/CD (Continuous Integration / Continuous Deployment)**: an automated
  pipeline that runs every time someone changes code — it checks the code,
  packages it, and ships it to a server, without a human running commands by
  hand. This is the "inspector" from the analogy: the checklist that runs
  itself, every time, whether anyone remembers to ask for it or not.
- **Ingress**: the rule that tells the outside internet how to reach a
  service running inside Kubernetes — which web address maps to which
  service. Like the address plate on the outside of the house that tells the
  mail carrier which door to use.
- **Autoscaling**: automatically adding more running copies of a service
  when traffic goes up, and removing them when traffic goes down. Like
  opening more checkout lanes at a store when a line forms, and closing them
  when it's quiet.
- **Rollback**: undoing a deploy and going back to the last known-good
  version, on command, without hand-editing anything. The equivalent of
  "if the new wiring trips the breaker, flip it back to the old wiring,
  instantly, without ripping open the wall."
- **Probe**: a small, repeated check Kubernetes runs against a service to
  ask "are you actually alive and able to do your job?" If the answer is no,
  Kubernetes stops sending it traffic, or restarts it. Like a smoke detector
  that's always listening, not just checked once at move-in.

## The worked example: shipping a new payments service

Say a developer, Priya, needs to ship a brand-new payments service. Here is
what happens with this project, step by step — and then, side by side, what
she would have had to do without it.

### Step 1 — Start from the template

**With this project:** Priya runs one script
(`scaffold/new-service.sh`) and answers a few questions: service name, does
it need a public web address, does it need a database connection. The
script generates a new, ready-to-go folder for her service, already wired
into the standard chart.

**Without it:** Priya either copies another team's setup and hopes it still
applies to her, or starts from a blank folder and writes Kubernetes
configuration by hand — the kind of configuration that takes weeks of
learning to get right and has dozens of ways to get subtly wrong.

### Step 2 — Write the actual payments code

**With this project:** Priya writes her application code exactly like she
would for any project. Nothing about this step changes. That's deliberate —
developers shouldn't need to become infrastructure experts to ship a
service.

**Without it:** Same as above — no difference here either way. But she now
also has infrastructure decisions hanging over her that she isn't trained to
make.

### Step 3 — Push the code

**With this project:** Priya pushes her code to the shared code repository.
That single push automatically triggers the pipeline (the CI/CD "inspector"):
it checks her code for known security problems, packages it into a
container, and checks that container against a security scanner. If
anything critical turns up, the pipeline stops right there — nothing broken
or unsafe reaches a server.

**Without it:** Someone has to remember to run these checks manually, if
they exist at all. Under a deadline, they often get skipped "just this once."
"Just this once" is how unpatched, insecure software ends up running in
production.

### Step 4 — Automatic deploy to a test environment

**With this project:** Once the checks pass, the pipeline automatically
installs Priya's service into a development environment — a safe practice
server that isn't handling real customers yet. This happens with zero manual
commands. The service comes up already configured with:
- A wall around it so it can't be tricked into running as an all-powerful
  system user (it runs as an ordinary, restricted user, and can't even
  rewrite its own files after starting).
- Automatic health checks (the probes) so that if the service is frozen or
  broken, Kubernetes notices and stops sending it customer traffic instead
  of silently failing.
- A defined amount of computer memory and processing power it's allowed to
  use, so one broken service can't accidentally starve every other service
  on the same servers.

**Without it:** Someone has to remember to configure every one of those
things, correctly, from scratch, for every single service. In practice,
across ten teams, some will get it right, some will half-get it right, and
some won't think about it at all until something goes wrong.

### Step 5 — Going to production requires a real decision

**With this project:** Going from the test environment to the real,
customer-facing production environment is not automatic. A specific person
has to review it and approve it — a manual gate the pipeline enforces. Once
approved, the deploy happens the same automated way, with the same checks,
just against the real servers this time.

**Without it:** Whether production deploys need approval depends on whether
anyone set that up. In a lot of home-grown setups, the answer is "whoever
has access can push whenever."

### Step 6 — Something goes wrong

**With this project:** If the new payments service misbehaves after going
live, anyone authorized runs one documented command — a rollback — and the
system reverts to the exact previous version, automatically, in minutes.
This has actually been tested, not just described in a document somewhere.

**Without it:** Rolling back means someone manually figuring out what the
previous version even was, then manually redeploying it by hand, hoping they
remember every setting correctly, under pressure, possibly at 2 a.m.

### The side-by-side, all in one place

| | With this project | Without it |
|---|---|---|
| Starting a new service | One script, a few questions | Copy-paste from another team, or start blank |
| Security settings (non-root, locked-down filesystem) | Built into the standard template, always on | Depends who remembers, and how carefully |
| Code and container security scanning | Automatic, on every push, blocks bad code | Manual, easy to skip under deadline |
| Health checks | Required, no opt-out | Optional, often forgotten |
| Multiple servers for reliability | Standard part of the template | Whatever each team decides, if anything |
| Auto-scaling under load | Config flag away | Reinvented per team, or absent |
| Path to production | Automatic to test, gated approval to prod | However each team wired it, or nobody |
| Rolling back a bad deploy | One documented, tested command | Manual, improvised, stressful |
| Knowing what's running in prod, and who approved it | Logged automatically by the pipeline | Tribal knowledge, if it's known at all |

The point of the contrast isn't that the manual path is impossible. A
careful, experienced engineer can do every one of these things correctly by
hand. The point is that "correctly by hand, every time, across every team,
under every deadline" is not something a company can rely on. This project
turns "please remember to do this safely" into "the system won't let you do
it any other way."

## Why this matters more at a bank or fintech than at a normal company

At most companies, a bad deploy is embarrassing. At a bank or a fintech, a
bad deploy can be a compliance failure, a real financial loss, or both, and
someone outside the company will ask about it.

A few reasons this shows up harder in finance:

**Auditors ask "prove it," not "tell me."** A regulator or an internal audit
team doesn't want to hear "we're pretty sure our services run securely."
They want evidence: a record of every change, who approved it, and what
checks it passed. Because every deploy in this project goes through the
same pipeline, that evidence already exists — it's the pipeline's own
history — instead of needing to be reconstructed after the fact from memory
and screenshots.

**"Move fast and fix it later" is not an acceptable posture with money.**
At a social media company, a bug might mean an ugly page for an hour. At a
bank, a bug in a payments path can mean money sent to the wrong place,
double-charged customers, or a transaction that silently fails and no one
notices for days. The controlled, gated path to production — automatic to a
safe test environment, human-approved before touching real money — exists
specifically to put a person in the loop before the highest-stakes changes
go live.

**A bad deploy has a dollar amount attached.** Downtime at a payments
company doesn't just annoy users, it stops transactions from clearing,
which is a directly measurable loss, sometimes with contractual penalties
attached. The tested rollback path exists to make "how fast can we undo
this" a known number, not a guess made under pressure.

**Consistency is itself a security control.** If every service is built the
same way, a security reviewer can review the *template* once, deeply, and
trust that every service built from it inherited that review. Without a
shared template, a reviewer has to re-review every single service from
scratch, and almost never actually has time to.

## What this does NOT do

Being honest about the edges of this matters as much as explaining what it
does.

- **It does not write or test the application code itself.** This project
  gets a service onto a server safely. It has no opinion on whether the
  payments logic inside that service is correct. That's still entirely on
  the developer and their own tests.
- **It does not manage secrets for you in a real, production-safe way yet.**
  The chart has a place for sensitive values like passwords, but a real
  deployment needs a proper secret-storage system behind it (like AWS
  Secrets Manager). Wiring that up is a known next step, not something this
  project claims to already solve.
- **It does not replace a security review.** Automated scanning catches
  known, previously-catalogued problems. It does not catch a subtle logic
  flaw or a novel exploit a human reviewer might find.
- **It does not run itself, unattended, without human decisions.** Someone
  still has to approve production deploys, decide on capacity, and respond
  when something breaks at 2 a.m. This project narrows what can go wrong and
  speeds up the response — it doesn't remove humans from the loop.
- **It has not been battle-tested at real, sustained production scale.**
  What's been done is a controlled build-out and testing of the pattern
  itself. I have not run this under real customer traffic over months, and I
  won't claim otherwise.
- **It's one opinionated path, not a general-purpose platform.** It
  intentionally does not support every possible way of running software on
  Kubernetes. That's a deliberate trade: fewer choices, in exchange for
  every choice that remains being a safe one.

## The 60-second spoken version

"Companies that let every team deploy their own way end up with inconsistent
security, no reliable rollback, and no real answer to 'what's running in
production and who approved it.' I built a standard, opinionated pipeline for
shipping software to Amazon's Kubernetes service, EKS — think of it like a
city adopting one licensed wiring process instead of letting every homeowner
wire their own house. A developer runs one script to scaffold a new service,
pushes their code, and a pipeline automatically security-scans it, deploys
it to a test environment with locked-down security settings and health
checks built in by default, and only reaches real production after a human
approves it. If something breaks, there's a tested one-command rollback. The
underlying engine is a Helm chart — a reusable template for Kubernetes
configuration — but the actual product is the guardrails: nobody can
accidentally ship something that skips the security settings, skips the
scan, or skips the approval, because the system doesn't give them a path to
do that."

## Gaps I had to assume knowledge to bridge

Places where I assumed some baseline, even after trying to define
everything:

- I assumed the reader knows what "the internet," "a website," and "an
  address/URL" are, per your instructions.
- I assumed the reader has an intuitive sense of what "code" and "pushing
  code to a repository" mean, even though I explain "repository" loosely in
  passing rather than formally defining it.
- I assumed the reader understands "memory" and "processing power" as
  general computer resource concepts, without defining them as computing
  terms.
- I assumed a rough intuition for what "security scanning for known
  problems" means, without explaining what a CVE or vulnerability database
  actually is under the hood.
- I used "servers," "computers," and "the cloud" a little loosely as
  near-interchangeable in places — a careful reader might want those
  distinguished more precisely (a physical/virtual machine vs. "AWS's
  computers you rent" vs. Kubernetes' abstraction over a pool of them).
- I did not explain what "OIDC" or "IAM roles" are (the actual mechanism
  behind the "no static AWS credentials" security rule) — I kept that detail
  out of this document entirely since it's one layer past what a
  non-technical reader needs, but it's worth having a crisper answer ready
  if an interviewer probes on exactly *how* the pipeline authenticates to
  AWS without stored passwords.
