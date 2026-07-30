# Gap Tracker

Checklist of specific claims in [what-is-this.md](what-is-this.md) that
describe the target design rather than what's actually built and verified
today. Each box is one overstatement, tied to where it appears in the
explainer. Tick a box only once the underlying thing is real — implemented,
merged, and actually exercised, not just written.

- [x] **Automatic security scanning on push** (what-is-this.md, worked
      example Step 3: "the pipeline stops right there — nothing broken or
      unsafe reaches a server"; also the comparison table row "Container
      image and manifest security scanning" — renamed from "Code and
      container security scanning" during the final reconciliation pass,
      since "code" scanning specifically doesn't exist; see the new item
      below). **Resolved (Day 5):** `ci.yml` now
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

- [x] **"Nobody can skip the scan or skip the approval"** (what-is-this.md,
      60-second spoken pitch, closing line). **Resolved (Day 5).**

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

      **Built the real fix instead:** `deploy.yml` has a `scan-image` job
      that extracts the caller's actual image (from their own
      `values-<environment>.yaml`) and runs `trivy image` against it;
      `deploy` `needs: [validate-inputs, scan-image]`. In-workflow
      enforcement, independent of repo settings, `chart_ref`, or which
      repo is calling.

      **Then witnessed it working, live, in both directions** — not left
      as a design claim. Two scratch branches (never merged), same
      `environment: dev` to isolate this from the already-witnessed
      approval gate:
      - `test/scan-gate-fail`: pointed at `nginx:1.14.0` (146+ real
        HIGH/CRITICAL CVEs). Job graph, confirmed by screenshot:
        `scan-image` failed (110 real findings on this run), `deploy`
        shown with a distinct skip icon, connected downstream by the
        graph's own arrow, zero steps executed. Reproduced on a re-run
        of the same commit.
      - `test/scan-gate-pass`: pointed at
        `nginxinc/nginx-unprivileged:1.29.2-alpine`, the exact image
        `ci.yml`'s own scan already verified clean. `scan-image`
        produced no failure annotation; `deploy` produced a real
        `Input required and not supplied: aws-region` annotation,
        meaning it genuinely executed and reached the AWS step before
        failing — proving it ran strictly *after* `scan-image`, not
        instead of it.

      See NOTES.md, "Day 5 close-out, for real" for the full record,
      including a real, separate, still-open gap this surfaced:
      `deploy.yml` expects `values-<environment>.yaml`, but this repo's
      own root has never had a properly-named `values-dev.yaml` — worked
      around with scratch files for this test, not yet fixed for real.

      **Deliberately still excluded from this claim's scope:** self-review
      prevention on the approval gate remains configured-but-not-runtime-
      verified — a solo maintainer structurally cannot exercise a
      two-person control. That was never part of "skip the scan or skip
      the approval" and stays flagged on its own in the entry above, not
      smoothed into this one.

---

Three new items added during the final reconciliation pass — not previously
tracked here, caught by checking every claim in what-is-this.md against
`git ls-files` and `NOTES.md` rather than memory:

- [ ] **A service-generator script** (what-is-this.md, worked example
      Step 1: used to say "Priya runs one script
      (`scaffold/new-service.sh`)"). Confirmed via `git ls-files`: no
      `scaffold/` directory exists anywhere in this repo. Step 1 now
      describes the honest current state (copy `examples/sample-api` by
      hand) with the script named as planned, not implied to exist.

- [ ] **Application source-code scanning** (what-is-this.md, worked
      example Step 3 and the closing pitch used to say "checks her code
      for known security problems"). Confirmed: no SAST tool (CodeQL,
      Semgrep, or similar) exists anywhere in `.github/workflows/`. What
      exists — and is real — is deployment-configuration misconfiguration
      scanning and container-image vulnerability scanning, neither of
      which inspects application source code. Text rewritten to say this
      explicitly rather than let "security scanning" imply more than it
      is.

- [ ] **A rollback actually exercised against a real deploy.** Step 6 and
      the closing pitch used to say rollback "has actually been tested."
      Checked `NOTES.md` for any record of running `helm rollback` — none
      exists. The command is real and the chart supports it; it has never
      been run in this project. This is the most significant of the three
      overclaims caught in this pass, and the text now says so plainly
      instead of implying a verification that never happened.
