# NOTES

Running log of what broke, why, and what it taught me. Entries are appended
in the order things were found and fixed.

---

## Finding: deploy.yml deployed straight to production with no approval gate

**Symptom:** `.github/workflows/deploy.yml` runs `helm upgrade --install
my-app-prod ./eks-mega-chart --namespace production ... --atomic` as a
single job, triggered directly by a push. There is no separate dev/staging
step, no `environment:` block, and no required reviewer — nothing in the
workflow itself stops a push from going straight to the `production`
namespace.

**Cause:** The workflow was written as one job that lints and deploys in
the same breath, with production as the only target. Nobody wired in a
gate, because the workflow was never designed with a dev-first path.

**What actually saved us:** The only reason this workflow has never fired
against a real cluster is an unrelated bug — it triggers `on: push:
branches: [main]`, and this repository's default branch is `master`. Every
push to `master` has silently done nothing, for the wrong reason. The
moment someone renames the branch, or someone pushes to a branch literally
named `main`, this workflow deploys to production with zero human review.

**What it taught me:** A safety property that only holds by accident (a
typo'd branch name) isn't a safety property — it's a coincidence with an
expiration date. "It's never happened" is not the same as "it can't
happen." This is exactly the gap CLAUDE.md's non-negotiable rule ("Dev
deploys automatically. Prod requires a manual approval gate.") is meant to
close, and it's still open. Fixing the branch trigger (below, among the
four `deploy.yml` defects) does not add the missing approval gate — that's
a separate, larger change (environments, `workflow_call`, required
reviewers) and is explicitly out of scope for this pass. Flagging it here
so it isn't forgotten.

---
