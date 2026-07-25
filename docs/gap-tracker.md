# Gap Tracker

Checklist of specific claims in [what-is-this.md](what-is-this.md) that
describe the target design rather than what's actually built and verified
today. Each box is one overstatement, tied to where it appears in the
explainer. Tick a box only once the underlying thing is real — implemented,
merged, and actually exercised, not just written.

- [ ] **Automatic security scanning on push** (what-is-this.md, worked
      example Step 3: "the pipeline stops right there — nothing broken or
      unsafe reaches a server"; also the comparison table row "Code and
      container security scanning"). `deploy.yml` currently runs `helm lint`
      and nothing else — no code scanner, no container image scan.

- [x] **Automatic deploy to a safe dev/test environment** (what-is-this.md,
      worked example Step 4: "the pipeline automatically installs Priya's
      service into a development environment — a safe practice server that
      isn't handling real customers yet"). **Resolved (Day 4):**
      `deploy.yml` is now a `workflow_call`-only reusable workflow with
      `dev`, `staging`, and `prod` as separate GitHub Environments, each
      with its own scoped AWS role/region/cluster vars. A service's caller
      workflow (see `examples/sample-api/.github/workflows/deploy.yml`)
      deploys to `dev` automatically on every push, no approval — exactly
      the "safe practice server" this claim describes. Verified: `helm
      lint`/`helm template` against `examples/sample-api/values-dev.yaml`
      render correctly against the real chart. Not verified: an actual
      live cross-repo `workflow_call` run (needs a real service repo to
      invoke it).

- [x] **Manual approval gate before production** (what-is-this.md, worked
      example Step 5: "a specific person has to review it and approve it —
      a manual gate the pipeline enforces"; also the comparison table row
      "Path to production"). **Resolved (Day 4):** the `deploy` job in
      `deploy.yml` sets `environment: ${{ inputs.environment }}`
      dynamically, so a caller targeting `prod` runs under whatever
      protection rules that environment has in the calling repo's own
      Settings — required reviewers included. A separate `validate-inputs`
      job (no `environment:` of its own) rejects anything except exactly
      `dev`/`staging`/`prod` before the gated job ever runs, so a typo
      can't silently create an unprotected ad-hoc environment and skip the
      gate that way. See NOTES.md for the full design and, importantly,
      what's *not* verified yet: the actual "waiting for approval" UI
      behavior, a real reviewer approving a real run, and the OIDC trust
      policy accepting the token, have not been exercised live — this is
      implemented and reasoned through, not yet watched happen.

- [ ] **"Nobody can skip the scan or skip the approval"** (what-is-this.md,
      60-second spoken pitch, closing line). Still false, but for a
      narrower reason now than when this was written. The approval half is
      real (see above). The **scan** half is not: neither `ci.yml` nor
      `deploy.yml` runs any code or container security scanner — no Trivy,
      no Snyk, no CodeQL, nothing that checks for known CVEs. `ci.yml`
      validates the chart (lint, template, kubeconform, schema, unittest,
      a kind smoke test); none of that is a security scan. Don't tick this
      one until a real scanner exists in the pipeline and actually gates
      it — the same "implemented and exercised, not just written" bar as
      everything else on this list.
