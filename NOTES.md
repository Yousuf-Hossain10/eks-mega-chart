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

## Bug: `.gitignore.txt` was a tracked, empty, non-functional gitignore

**Symptom:** The repo had a file named `.gitignore.txt` — the `.txt`
extension means git never recognized it as an ignore file, and it was also
completely empty (0 bytes, the standard empty-blob hash). Nothing was ever
actually being ignored.

**Cause:** Wrong filename, and never filled in.

**Fix:** `git mv .gitignore.txt .gitignore`, then added real patterns:
packaged Helm artifacts (`*.tgz`, subchart dependency directories),
`CLAUDE.local.md` (per CLAUDE.md, that's explicitly where real account
IDs/ARNs/hostnames are meant to live, so it must never be committed),
local secret/env files, editor/OS noise, and local values overrides.

**What it taught me:** A `.gitignore` with the wrong extension is silently
useless — git won't error, it'll just track everything as if the file
didn't exist. Worth checking that ignore files are actually named correctly,
not just present.

---

## Surprise: the `/tmp` emptyDir for read-only root filesystem was already half-written, but wired to a condition that could never be satisfied

**Symptom:** While implementing `readOnlyRootFilesystem: true`, found that
`deployment.yaml` already had code to mount an `emptyDir` at `/tmp` — but it
was gated behind `{{- if .Values.volumeMounts }}`, and `.Values.volumeMounts`
defaults to `[]` in every environment this chart ships with. So even before
today, if someone had simply flipped `readOnlyRootFilesystem: true` in
values without also editing the template, the tmp mount still would not
have appeared — the container would have started with a read-only root and
no writable `/tmp`, and anything that writes temp files (which is most
runtimes) would have failed at runtime, not at render time.

**Cause:** The `/tmp` mount was written as if it belonged to the *optional
extra volumes* feature (`.Values.volumeMounts`/`.Values.volumes`) instead of
being tied to the thing that actually requires it — `readOnlyRootFilesystem`.

**Fix:** Re-gated both the `volumeMounts` and `volumes` blocks in
`deployment.yaml` on `.Values.securityContext.readOnlyRootFilesystem`
directly (with `.Values.volumeMounts`/`.Values.volumes` still supported
alongside it for genuinely optional extra volumes). Now the tmp mount
appears whenever the read-only root filesystem flag is on, independent of
whether the user configured any other volumes.

**What it taught me:** Half-implemented security features are worse than
no feature at all, because they look done in a code review. `helm
lint`/`helm template` couldn't have caught this either — nothing about it
is a syntax error, it's a wiring mistake between two independent
conditions that happened to both default to "off."

---

## Surprise: adding `values.schema.json` forced removing the default `image.tag`

**What happened:** `values.yaml` shipped with `image.tag: "latest"` as its
default the entire time. The moment `values.schema.json` requires `tag` to
look like a git SHA (rejecting `latest`), the chart's own default value
fails its own schema — `helm lint .` with zero overrides now fails,
correctly, because there is no such thing as a valid default git SHA. A
tag is commit-specific by definition; baking one in as a chart default was
never really coherent, "latest" was just a placeholder that happened to
render.

**Fix:** Changed `image.tag` default to `""` (still fails the schema, on
purpose) and documented that every render — including local dev testing,
not just CI — must now supply `--set image.tag=<sha>` explicitly. This
matches what `deploy.yml` already does (`--set image.tag=${{ github.sha
}}`); it just makes that requirement enforced everywhere instead of only
being true in the one place someone remembered to type it.

**Also worth noting:** the schema originally rejected `latest` via
`"not": {"const": "latest"}`, which is technically correct but produces an
unreadable error (`Must not validate the schema (not)`). Switched to
`"pattern": "^[0-9a-f]{7,40}$"` instead — this enforces the actual rule
("image tags are git SHAs") rather than just its negation, and rejecting
`latest` falls out as a side effect, with a much clearer error message
naming the pattern it failed to match.

**What it taught me:** A schema doesn't just validate values going
forward — it immediately audits every existing default in the chart. The
first thing it found wrong was the chart's own values.yaml.

---

## Note: `autoscaling.maxReplicas >= minReplicas` can't live in the schema

**What happened:** JSON Schema (as Helm's validator implements it) has no
built-in way to compare two sibling values to each other — there's no
"this number must be >= that other number" keyword available without
vendor extensions Helm doesn't support. So `minReplicas: 5, maxReplicas: 2`
passes `values.schema.json` cleanly: both are individually valid integers
`>= 1`. The contradiction between them is invisible to a schema that only
ever looks at one field at a time.

**Fix:** Added a guard at the top of `templates/hpa.yaml` —
`{{- if lt (int .Values.autoscaling.maxReplicas) (int .Values.autoscaling.minReplicas) }}{{ fail ... }}{{- end }}`
— that runs during template rendering, after schema validation has
already passed. `helm fail` produces a real, blocking render error, just
via a completely different mechanism (Go template execution) than schema
validation (structural JSON matching before any template even runs).

**What it taught me:** this is the cleanest real example of the boundary
between what a values schema can and cannot catch — see the writeup below
and `values-invalid-replicas-demo.yaml` for the demonstration.

---
