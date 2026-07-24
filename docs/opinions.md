# Opinions

What is decided for you, and why. If you disagree with one of these,
argue it — some of them are genuinely close calls (see "Open decisions"
at the bottom). Don't silently work around one; change it here, for
everyone, in a reviewed diff.

## Two categories of "decided for you"

Not everything in this chart is decided the same way, and conflating the
two categories is where a security policy quietly rots:

**Configuration** — things that legitimately vary by service. Whether a
service needs an `Ingress`, whether it needs autoscaling, whether a `PDB`
makes sense at `replicaCount: 1`. These get a values.yaml toggle
(`ingress.enabled`, `autoscaling.enabled`, `pdb.enabled`), because a real
"off" state exists for some services and not others. **Opt-out here
requires justification** — write down *why* this service doesn't need it
— but the mechanism for opting out is just... setting the value, because
that's what the field is for.

**Platform invariants** — things that are true of every workload this
platform runs, full stop. There is no version of "this service legitimately
runs as root" that isn't actually "we haven't fixed the image yet." These
don't get a values.yaml toggle at all, because a toggle implies a real
off-state exists, and for these three, it doesn't.

## Platform invariant: no per-service opt-out on container hardening

`securityContext.runAsNonRoot`, `securityContext.readOnlyRootFilesystem`,
and `securityContext.allowPrivilegeEscalation` are pinned in
`values.schema.json` with `const` — `true`, `true`, `false` respectively.
No environment overlay, no per-service values file, can set any of them
to anything else. `helm lint`/`helm template` will reject the attempt.

**Why not a toggle with required justification, like the configuration
category above?** Because a toggle here is a silent escape hatch, not a
documented decision. Nothing forces the "justification" to actually be
written anywhere, reviewed by anyone, or visible to the next engineer who
reads `values.yaml` for that service — it's just a `false` sitting in a
file. Given this whole project exists to stop ten teams from each quietly
inventing their own security posture (see
[what-is-this.md](what-is-this.md)), a per-service flag that flips a
non-negotiable off is exactly the failure mode being designed against.

**If a workload genuinely can't meet one of these** (a legacy image that
writes to arbitrary paths, say): the path is not a values override. It's
either fixing the image, or opening a change against `values.schema.json`
itself — reviewed like any other change to this chart, visible in git
history, and applying to every future workload that reads the diff, not
just the one that needed the exception today. That's still justification.
It just changes the rule for everyone instead of exempting one service
from it quietly.

This is a real design choice, not an obviously-correct one — a team with
enough scale genuinely does end up needing a formal waiver process for
exactly this kind of exception (audited, time-boxed, tied to a ticket).
That doesn't exist here. Building a values.yaml toggle that *looks* like
governance without any actual review process behind it would be worse
than no opt-out at all, so until a real waiver mechanism exists, the
answer is: no opt-out.

## Open decisions

- **Resource limits (ADR-0002).** Whether `resources.limits.cpu` should
  exist at all, and whether `requests.memory` must equal `limits.memory`,
  is intentionally not enforced by `values.schema.json` yet. It's flagged
  there as an open question rather than encoded as a rule, because it's a
  genuine trade-off (CPU throttling behavior vs. predictable bin-packing)
  that hasn't been decided yet for this project. See
  [adr/0002-resource-limits.md](adr/0002-resource-limits.md) once it
  exists.
- **Image tag format.** `values.schema.json` requires `image.tag` to be
  set and rejects `"latest"`. Whether it should also require the tag to
  look like a git SHA specifically (chart-level contract) versus leaving
  that enforcement to CI (deployment policy for this org's own app
  images) is an open question — see the tag-pattern discussion in
  NOTES.md.
