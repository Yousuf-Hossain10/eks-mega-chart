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

- [ ] **Automatic deploy to a safe dev/test environment** (what-is-this.md,
      worked example Step 4: "the pipeline automatically installs Priya's
      service into a development environment — a safe practice server that
      isn't handling real customers yet"). `deploy.yml` has one job, one
      target: it deploys straight into the `production` namespace. There is
      no separate dev or staging environment anywhere in the pipeline.

- [ ] **Manual approval gate before production** (what-is-this.md, worked
      example Step 5: "a specific person has to review it and approve it —
      a manual gate the pipeline enforces"; also the comparison table row
      "Path to production"). `deploy.yml` has no `environment:` block, no
      required reviewers, nothing that stops a push from going straight to
      production. See NOTES.md for the related finding that the only thing
      preventing this today is the (now-fixed) wrong branch trigger — not
      an actual gate.

- [ ] **"Nobody can skip the scan or skip the approval"** (what-is-this.md,
      60-second spoken pitch, closing line). This claim depends on all
      three items above being true simultaneously. Don't tick this one
      until the scan and the gate both exist and have been exercised for
      real, not just merged.
