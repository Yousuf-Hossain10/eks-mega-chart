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

## Bug: `templates/pdb.yaml` nil pointer — chart failed to render entirely

**Symptom:**
```
helm lint .
[ERROR] templates/: template: eks-mega-chart/templates/pdb.yaml:1:14: executing "eks-mega-chart/templates/pdb.yaml" at <.Values.pdb.enabled>: nil pointer evaluating interface {}.enabled
Error: 1 chart(s) linted, 1 chart(s) failed
```
Same error, verbatim, from `helm template test .` and `helm template test .
-f values-prod.yaml` — the second command doesn't even get far enough to
apply the prod overrides, because template evaluation fails before that
matters.

**Cause:** `templates/pdb.yaml` reads `.Values.pdb.enabled`, but
`values.yaml` had no top-level `pdb:` key at all. Referencing a field on a
nil map throws immediately in Helm's template engine — this isn't a
Kubernetes-side failure, the chart never got as far as producing YAML.

**Fix:** Added a `pdb:` block to `values.yaml` with `enabled: false` and
`minAvailable: 1`, matching what the README's configuration table already
(incorrectly) claimed was the default. `enabled: false` is the correct
default because a PDB only means something once `replicaCount > 1`.

**What it taught me:** Every values key a template reads has to exist in
`values.yaml`, even if the feature defaults to off. Referencing a key
that's never defined isn't "off by default" — it's a crash by default.
This was already flagged in CLAUDE.md's "Current known state" checklist
before I even started; confirmed for real here rather than taken on faith.

---

## Bug: four separate defects in `.github/workflows/deploy.yml`

**Symptom:** The workflow existed and was tracked in git the whole time —
an earlier scan of mine missed it because a `find`/`ls` exclusion pattern
accidentally also matched `.github`. Once actually read, it had four
distinct, independent bugs, any one of which would break or compromise a
real deploy:

1. **Static AWS credentials.** `configure-aws-credentials@v4` was given
   `aws-access-key-id` / `aws-secret-access-key` from `secrets.*`. This is
   exactly the anti-pattern CLAUDE.md's non-negotiable security rule
   forbids ("GitHub Actions authenticates to AWS via OIDC role assumption
   only"). Long-lived static keys sitting in GitHub Secrets are a standing
   credential-leak risk with no expiry.
2. **Wrong trigger branch.** `on: push: branches: [main]`, but this repo's
   default branch is `master`. This workflow has never fired for a real
   reason — see the finding at the top of this file. It's also the only
   thing that has been preventing the static-credential and
   no-approval-gate problems from being exercised for real.
3. **Wrong chart path.** Every `helm` command referenced `./eks-mega-chart`,
   but `Chart.yaml` lives at the repo root. `helm lint ./eks-mega-chart`
   and the deploy step would both fail with "no such file" the moment the
   branch trigger ever matched.
4. **Missing line continuation.** The deploy step's `run: |` block ended
   the `--atomic` line with only a trailing `#` comment, no `\`. In a
   YAML literal block passed to bash, each line runs as its own command
   unless explicitly continued — so `--timeout 5m0s` would have executed
   as its own (invalid) shell command, not as a flag on `helm upgrade`.

**Cause:** The workflow reads like it was drafted once, by hand, without
ever being run — every one of these bugs would have surfaced immediately
on a single real trigger, and none of them are subtle.

**Fix:** Changed the trigger to `master`; switched to
`role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}` with
`permissions: id-token: write` for OIDC (no static keys, no ARN committed —
the actual ARN is a repo variable set at deploy time, per CLAUDE.md's rule
against committing role ARNs); changed `./eks-mega-chart` to `.` in both
the lint and deploy steps; added the missing `\` after `--atomic`.

**What it taught me:** A workflow file that's never been triggered
provides zero evidence it works — "it's in the repo" and "it runs" are
different claims. Also confirms why the earlier `git ls-files` correction
mattered: I can't audit, let alone fix, a file I never actually read. This
does **not** add the missing production approval gate noted above — that's
still open and out of scope for this pass.

---

## Bug: `envFrom` in deployment.yaml referenced ConfigMap/Secret unconditionally

**Symptom:** `templates/configmap.yaml` and `templates/secret.yaml` are both
conditional (`{{- if .Values.configData }}` / `{{- if .Values.secretData }}`)
— they render nothing when those maps are empty. But
`templates/deployment.yaml` had an unconditional `envFrom` block naming both
resources by their generated names regardless. Verified directly: with
`configData: {}` and `secretData: {}` at the base of `values.yaml`, `helm
lint`/`helm template` still passed clean (0 ConfigMap/Secret resources
rendered, but `envFrom` still pointed at both). Helm's own render can't
catch this class of bug — it's not a template syntax error, it's two
templates disagreeing about whether a referenced resource exists. On a real
cluster this becomes `CreateContainerConfigError`: the kubelet can't start
the container because `envFrom` names a ConfigMap/Secret that was never
created.

**Cause:** `configmap.yaml`/`secret.yaml` and `deployment.yaml` were written
as if they'd always agree, but only the first two actually check
`configData`/`secretData` before rendering. `deployment.yaml` never made the
same check.

**Fix:** Wrapped each `envFrom` entry in `deployment.yaml` with the same
condition its corresponding resource template already uses
(`{{- if .Values.configData }}` for `configMapRef`, `{{- if
.Values.secretData }}` for `secretRef`), and wrapped the whole `envFrom` key
in `{{- if or .Values.configData .Values.secretData }}` so an empty
`envFrom:` key is never emitted with nothing under it. Confirmed by
re-rendering with both maps empty: `envFrom` no longer appears in the output
at all. Confirmed again with the real `values.yaml`/`values-prod.yaml`
(both non-empty): `envFrom` still renders both refs exactly as before — no
behavior change for the actual environments this chart ships with.

**What it taught me:** `helm lint` and `helm template` only catch template
*syntax* errors — they don't know that a name generated in one template is
supposed to match a resource conditionally created in another. That kind of
cross-template contract has to be checked by hand, or by actually deploying
and watching pod status (`CreateContainerConfigError` only shows up at
runtime, not at render time). Any place a resource name is referenced,
check whether the thing that creates it is unconditional too.

---
