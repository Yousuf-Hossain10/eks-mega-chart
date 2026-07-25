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
      60-second spoken pitch, closing line). Still false, but the shape of
      the remaining gap changed on Day 5 and is worth recording precisely
      rather than rounding up.

      First checked whether **branch protection on `eks-mega-chart`'s
      `master`** (required status checks) would close it — the obvious,
      repo-settings answer, same category as the Day 4 environment rules.
      Traced it precisely and the honest answer was no: that only gates
      *merging into this repo*. It doesn't stop a caller from overriding
      `chart_ref` to a ref that never went through CI, and more
      fundamentally it never touches the **service's own application
      image** — `ci.yml`'s scans check this repo's chart template and a
      fixed demo image, never the actual thing `deploy.yml` ships via
      `inputs.image_tag`. A fully protected `master` here would have
      flipped this checkbox without making the underlying claim true.

      **Built the real fix instead:** `deploy.yml` now has a `scan-image`
      job that extracts the caller's actual image (from their own
      `values-<environment>.yaml`) and runs `trivy image` against it;
      `deploy` `needs: [validate-inputs, scan-image]`. This is in-workflow
      enforcement, independent of repo settings, `chart_ref`, or which
      repo is calling — structurally the right mechanism, not the one that
      just turns the box green.

      **Still unchecked because it has never been triggered — not once,**
      unlike the approval gate, which was genuinely run and watched
      pausing before that box was marked resolved. The design is now
      correct; that's real, recorded progress. It is not yet a witnessed
      fact. Also worth naming: `examples/sample-api` still has no real
      image behind it, so even a live trigger today would hit
      `UNAUTHORIZED` at the image-pull step rather than demonstrate a
      genuine clean-scan-then-deploy pass — a real image is needed before
      this can be fully exercised, not just re-run.
