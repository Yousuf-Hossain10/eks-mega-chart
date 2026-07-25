# Gap Tracker

Checklist of specific claims in [what-is-this.md](what-is-this.md) that
describe the target design rather than what's actually built and verified
today. Each box is one overstatement, tied to where it appears in the
explainer. Tick a box only once the underlying thing is real — implemented,
merged, and actually exercised, not just written.

- [x] **Automatic security scanning on push** (what-is-this.md, worked
      example Step 3: "the pipeline stops right there — nothing broken or
      unsafe reaches a server"; also the comparison table row "Code and
      container security scanning"). **Resolved (Day 5):** `ci.yml` now
      has `trivy-config-scan` (misconfiguration scanning against the
      rendered chart) and `trivy-image-scan` (vulnerability scanning
      against a real pinned image), both gating on HIGH/CRITICAL. Both
      proven capable of actually failing, not just running: a deliberately
      insecure fixture and a deliberately old/vulnerable image each verified
      to produce real findings and a nonzero exit code before either job
      was trusted. See NOTES.md for the full design, including the
      `.trivyignore`/`--ignore-unfixed` policy for CVEs that can't be
      fixed immediately.

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
      gate that way. **Update:** this has since been witnessed live, not
      just implemented — see NOTES.md, "Witnessed on 2026-07-25": a real
      prod-targeted run paused at "Waiting for approval" with zero steps
      executed, then proceeded only after approval and failed downstream
      at the AWS step (no real role configured), confirming the gate
      itself blocks execution, not just config. The first attempt at this
      test showed no pause at all — caught and fixed a real gap (the
      `prod` environment's "Required reviewers" checkbox had never
      actually been saved). Self-review prevention specifically remains
      configured-but-not-runtime-verified — see NOTES.md for why that
      one structurally requires a second reviewer.

- [ ] **"Nobody can skip the scan or skip the approval"** (what-is-this.md,
      60-second spoken pitch, closing line). Still false — for a narrower,
      more precise reason again (Day 3: three unbuilt things; Day 4: one
      unbuilt thing; Day 5: zero unbuilt things, one missing *link*). Both
      halves now exist and both are individually proven to work: the scan
      genuinely fails on real findings (Day 5), the approval gate
      genuinely blocks execution (Day 4, witnessed live). What's still
      missing is **enforcement that ties them together and to the deploy
      path**. Checked directly: `ci.yml` and `deploy.yml` have zero
      coupling — no `needs:`, no shared trigger, nothing. A scan failure
      in `ci.yml` does not currently stop `deploy.yml` (or a service's
      caller workflow) from running. For "nobody can skip the scan" to be
      literally true, `ci.yml`'s scan jobs would need to be configured as
      **required status checks on the calling repo's branch protection**
      (a repo Settings change, same category as the environment protection
      rules from Day 4 — not something expressible in this YAML alone),
      and ideally a deploy should only be able to target a commit that
      actually passed CI. Neither is configured or verified yet. Don't
      tick this until that link exists and has been watched actually block
      something, the same bar as everything else on this list.
