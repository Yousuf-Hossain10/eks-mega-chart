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

**CLOSED (Day 4):** `deploy.yml` is now a `workflow_call`-only reusable
workflow with no trigger of its own - a push to `master` in this repo
deploys nothing. The `deploy` job's `environment: ${{ inputs.environment
}}` is dynamic per caller, and `prod`'s required-reviewer protection rule
(configured in each calling service repo's own Settings, not in this
YAML) is the actual gate; see the full entry further down for the design
and what's verified vs. not.

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

## Bug: same-line `#` comments leaked into rendered YAML in four templates

**Symptom:** `helm lint .` had been printing this warning since the very
first lint run of this whole session, unaddressed:
```
[WARNING] templates/secret.yaml: document starts with an illegal indent: "  # Conditional to skip if no secrets", which may cause parsing problems
```
Grepping rendered output for `#` showed it wasn't just that one line —
`secret.yaml`, `service.yaml`, `deployment.yaml`, and `hpa.yaml` all had
comments sitting on the same line as a `{{- if }}`/`{{- with }}` action or
right after a value substitution. Since only the `{{ }}` delimiters are
template syntax, everything else on that line — including a trailing `#
comment` — is literal text that gets emitted as part of the rendered
output. In the worst case this produced a comment attached to a
completely unrelated line: with `service.annotations` set, the rendered
output showed `app.kubernetes.io/managed-by: Helm  # Add for custom
annotations (e.g., ALB health)` — a comment about annotations, physically
attached to an unrelated label three lines away from where the comment
was actually written, because of how the `{{- with }}` block's whitespace
trimming interacted with the following line.

**Cause:** Comments written as `{{- if X }}  # explanation` instead of
`{{/* explanation */}}`. The former is two separate things (a template
action, then literal text) that only look like one line in the source.

**Fix:** Removed the comments that were self-evident from the code next
to them. Kept two that carried real information, moved to proper
`{{/* ... */}}` block comments so they can't leak: why `containerPort`
falls back to `80` in `deployment.yaml`, and the example shape of
`autoscaling.behavior` in `hpa.yaml`.

**What it taught me:** the lint warning had been sitting there, ignored,
through every single `helm lint` run this whole session — it's easy to
stop reading a warning once it becomes background noise. Also: `helm
template` output can look clean in a quick skim and still have comment
text landed on the wrong line entirely; grepping the rendered output for
`#` line-by-line is a better check than eyeballing it.

---

## Cleanup: purged nginx scaffold leftovers and two dead values keys

**What was there:** `Chart.yaml`'s `appVersion: "1.25.4"` was nginx's own
version number, left over from whatever scaffold this chart started as -
not this project's version, not tied to any real app. `values.yaml`'s
`image.repository` comment said "Change from nginx to your actual app"
even though the repository value itself had already been changed away
from nginx; the comment was stale. `values.yaml` also had `env: []` and
`envFrom: []` keys with comments implying templates would use them -
grepped `templates/` for `.Values.env` and `.Values.envFrom` and found
zero references. Both keys were pure scaffold noise, never wired to
anything.

**Fix:** Changed `appVersion` to a generic `0.1.0` placeholder with a
comment noting real deploys always set `image.tag` explicitly (enforced
by `values.schema.json`), so `appVersion` is essentially inert - it's
only ever a fallback via `.Values.image.tag | default .Chart.AppVersion`
in `deployment.yaml`, and the schema makes that fallback path
unreachable in practice. Cleaned up the repository comment. Removed
`env`/`envFrom` from both `values.yaml` and `values.schema.json`.

**What it taught me:** "the schema documents the contract" cuts both
ways - once `env`/`envFrom` were confirmed dead in templates, leaving
them in the schema as optional-but-unused properties would have kept
describing a contract the chart doesn't actually have.

---

## Bug: values-prod.yaml's DB_PASSWORD was a literal, unrendered template string

**Symptom:** `values-prod.yaml` had
`DB_PASSWORD: "{{ .Values.secretData.DB_PASSWORD }}"`. This was already
in CLAUDE.md's "Current known state" checklist before this session
started; confirmed and fixed here. Values files aren't templated by
Helm — that string was the literal 44 characters, never evaluated,
base64-encoded verbatim into the rendered `Secret`. There was never a
real password behind it, in this file or anywhere else in the repo.

**Also found while fixing it:** the obvious-looking fix,
`secretData: {}`, does not work. Confirmed by testing directly: an empty
map in an overlay merges with (and therefore doesn't clear) the fake dev
placeholders inherited from the base `values.yaml` — the same Helm
merge-semantics gotcha already documented earlier in this file for the
`envFrom` bug. `secretData: null` is what's actually required to unset
an inherited map entirely. Verified: with `null`, zero `Secret` resources
render for prod; with `{}`, the dev fake `DB_PASSWORD`/`API_KEY` would
have silently carried through into the "prod" render.

**Fix:** Set `secretData: null` in `values-prod.yaml`. A prod install
today renders no `Secret` and no secret-derived env vars — an honest
gap, not a fake one. Wrote `docs/secrets-management.md` laying out the
real options (External Secrets Operator, Sealed Secrets, SOPS, `--set`
from CI) with trade-offs, undecided on purpose.

**What it taught me:** a "fix" that just makes the symptom disappear
(`{}`) can be silently wrong in a way that's worse than the obvious bug —
the templated-string version was at least visibly broken; an empty map
would have looked correct while quietly deploying fake dev credentials
under a `prod` label.

---

## Decision: ADR-0002 written, values.yaml/values-prod.yaml didn't match it

**What happened:** CLAUDE.md referenced `docs/adr/0002-resource-limits.md`
as the place this decision lives, but the file never existed - confirmed
via `git ls-files` and a filesystem search before writing anything.
Wrote it, arguing CPU-request-no-limit / memory-request-equals-limit on
the technical merits (CPU is compressible so a limit only adds CFS
throttling latency without capping anything real; memory has no
throttle, only OOMKill, so request must equal limit or the scheduler can
bin-pack a node past what it can actually hold) - and argued the
opposite position just as hard (noisy-neighbor CPU starvation, cost/
capacity predictability, multi-tenant chargeback, compliance mandates
that just want the field set), plus the specific conditions under which
I'd reverse the call.

Checked `values.yaml` and `values-prod.yaml` against the decision once it
was written down, and neither matched it: both had a `limits.cpu` (200m
dev, 500m prod) and a memory request lower than the memory limit (128Mi
request / 256Mi limit dev; 256Mi / 512Mi prod) - the exact pattern the
ADR argues against. Removed `limits.cpu` from both; set memory request
equal to memory limit in both (kept the existing limit value as the
shared number, rather than the existing request value, to avoid
silently shrinking the pod's real memory ceiling).

**What it taught me:** writing the ADR is what actually caught this -
the values files had been carrying an undocumented, unexamined resource
policy since before this session started, and nothing failed lint or
schema validation because "has a CPU limit" isn't structurally wrong,
it's just a different (defensible, but different) policy than the one
now on record. A schema can validate shape; it can't tell you that your
defaults contradict a decision you haven't written down yet.

---

## Follow-up: named the exact misconception ADR-0002 was meant to kill

**What happened:** The `limits.cpu` comment that used to be in
`values.yaml` - "Add minimal limits for dev to prevent resource
starvation" - had the mechanism backwards, and it's a common enough
belief to be worth naming explicitly rather than just fixing the value
and moving on. A CPU *limit* doesn't protect a container from
starvation; it caps what that same container can use, including when
nothing is contending for anything. The thing that actually protects
against starvation under contention is the *request* (cgroup shares),
which exists independent of whether a limit is also set. Added a
dedicated section to ADR-0002 naming this directly, quoting the original
backwards comment as the concrete example, so the next person who
reaches for "add a limit to protect against starvation" sees exactly why
that's the wrong tool before they add one back.

**Separately, confirmed memory-based autoscaling is not half-configured.**
`targetMemoryUtilizationPercentage: 80` in `values.yaml` had a comment
reading "Uncomment and enable for memory-based scaling" while the line
itself was already active, uncommented, set to a real value. Checked
`templates/hpa.yaml`: it conditionally renders a memory `Resource`
metric whenever `.Values.autoscaling.targetMemoryUtilizationPercentage`
is set, alongside the CPU metric (HPA v2 scales on whichever metric
needs more replicas). Checked the container has a `resources.requests.memory`
set (from the ADR-0002 fix), which is required for a memory utilization
percentage to mean anything at runtime. Rendered the HPA with
`autoscaling.enabled=true` and confirmed both metrics actually appear.
It's fully wired, not half-built - the comment was just stale. Fixed the
comment in both `values.yaml` and `values-prod.yaml` instead of removing
a feature that already works.

**What it taught me:** a stale comment claiming something is disabled
when it's actually active is arguably worse than no comment at all - it
actively points the next reader toward "fixing" something that isn't
broken, or worse, toward believing a working feature doesn't exist.

---

## ci.yml: two things found while building it, both verified before writing YAML

**No `values-staging.yaml` existed.** CI was asked to matrix over dev/
staging/prod, but this repo only ever had `values.yaml` (dev) and
`values-prod.yaml`. Confirmed via `git ls-files` before assuming
anything. Created `values-staging.yaml` as a reasonable midpoint between
the two - it's flagged in its own header comment as not having been
argued out the way ADR-0002 was, since nobody's actually stated
staging's real requirements yet.

**`helm/helm`'s "latest" release is now a v4 tag, and Helm v4 plugin
manifests aren't loadable by a v3 binary.** Tried installing the
`helm-unittest` plugin locally before writing the unittest job, and it
failed immediately: `unknown field "platformHooks"` - that field is part
of a plugin manifest schema Helm v3's plugin loader doesn't recognize at
all. This would have been a real trap: `azure/setup-helm@v4` (the GitHub
Action) with no `version` pinned installs whatever Helm currently calls
"latest," which today means v4, an untested major version for this whole
chart. Pinned `HELM_VERSION: v3.17.3` explicitly in `ci.yml` rather than
finding this out the first time CI actually ran. Also had to pin
`helm-unittest` to `v1.0.3` - `v1.1.x` ships the incompatible
`platformHooks` manifest; `v1.0.3` installs and runs cleanly under Helm
v3, verified locally (`helm unittest .` exits 0 with zero test suites,
confirming the "job exists and goes green with no tests yet" requirement
is actually true, not assumed).

**Confirmed kubeconform earns its place, not just added it on faith.**
Injected a typo'd `podSecurityContext.fsGroupp` (extra "p") via a values
override. `values.schema.json` doesn't catch it - `podSecurityContext` is
typed as a loose `{"type": "object"}` with no `additionalProperties`
restriction, so any key sails through. `helm lint`/`helm template` both
succeed. Only `kubeconform -strict`, validating against the real
Kubernetes OpenAPI schema, catches it: `additional properties 'fsGroupp'
not allowed`. This is the same class of typo that caused the original
`.Values.pdb.enabled` bug this whole project started from - kubeconform
is the layer that would have caught that class of mistake, which is
exactly the gap it's there to close.

**Simulated every job's exact shell logic locally before trusting the
YAML** - the matrix's if/else branching for all three environments, both
negative-test assertions, and the unittest job's plugin install and run
- using the same pinned Helm/kubeconform/plugin versions the workflow
pins. All of it ran clean. `helm lint`/`helm template`/`kubeconform` exit
1 on real failure, confirmed directly (an early exit-code check earlier
in this session had been silently wrong - a stray `echo;` before
`echo "EXIT: $?"` was resetting `$?` to 0 - worth remembering that
"exit code" checks need to read `$?` immediately after the command in
question, nothing in between).

---

## Real helm-unittest specs added; both Day 1 regressions proven to actually catch the bug

**Four suites, ten tests:** `tests/deployment_test.yaml` (probes render
for dev/staging/prod, securityContext pinned fields, ADR-0002 resources
- no CPU limit, memory request==limit - plus the envFrom regression),
`tests/pdb_test.yaml` (the pdb nil-pointer regression, plus a positive
case), `tests/hpa_test.yaml` and `tests/ingress_test.yaml` (each renders
only when its `enabled` flag is set).

**A test written against the literal wording would have proven nothing.**
The instruction was "setting `configData: {}`" - tried that first,
directly. It passed with `envFrom` showing 2 entries, not the expected 1,
because an empty-map `set:` override merges with (doesn't clear) the
three keys `configData` already has by default in `values.yaml` - same
merge-semantics gotcha as the `secretData` fix. Verified this directly
with a throwaway `helm template --set-json` before touching the test
file. `configData: null` is what actually empties it; that's what the
test uses, with a comment explaining why `{}` wouldn't have worked.

**Both regression tests were run against the real Day 1 broken commits,
not synthetic approximations.** For the envFrom bug: swapped in
`templates/deployment.yaml` from `20b2be6` (the commit immediately before
the fix), reran the suite, watched the exact three assertions fail with
`envFrom` showing both `configMapRef` and `secretRef` regardless of
`configData` being null - restored the current file, reran, green. For
the pdb bug: swapped in `values.yaml` from `289a309` (immediately before
`bbbd6d3` added the `pdb` key) - and had to *also* temporarily remove
`values.schema.json`, since the schema (added later, requires `pdb` as
a top-level key) now blocks this exact scenario a second, earlier way
and would have made the test fail for the wrong reason (a schema
rejection, not the template nil pointer). With both removed, the test
failed with the literal historical error - `nil pointer evaluating
interface {}.enabled` at `templates/pdb.yaml:1:14` - confirming this is
provably the same bug, not a look-alike. Restored both files, reran,
green.

**What it taught me:** the schema now provides a second line of defense
against some of the same regressions the unittest suite exists to catch
- which is good defense-in-depth, but it means an honest "does this test
still catch the original bug" check has to account for every later
guardrail that might mask it, not just revert the one file the original
fix touched.

---

## kind smoke test added to ci.yml - and the tag-pattern decision it forced

**Building tests/values-smoke.yaml immediately hit the still-open
tag-pattern question from a few prompts back.** `values.schema.json`'s
`image.tag` was still the strict git-SHA-only pattern (option (a)/(b)
were presented, deliberately left unresolved since it was explicitly
your call). `nginx:1.25.4` - a real, pinned, pullable public tag - failed
it immediately:
```
[ERROR] values.yaml: - image.tag: Does not match pattern '^[0-9a-f]{7,40}$'
```
This is the exact failure mode named when the pattern was first flagged
as too strict. Asked rather than silently picking a resolution, since it
was previously and explicitly reserved as your decision. You chose
option (a) - `anyOf` [git SHA / semver-ish / digest] with a sibling
`not: {const: "latest"}`. Applied it to `values.schema.json`; reran the
full regression sweep (dev/staging/prod lint+template, the two
`tests/invalid-*` fixtures, all four unittest suites) - everything still
passes, `latest` is still rejected, and the smoke overlay now validates.

**`tests/values-smoke.yaml`: the deliberate choice, not a shortcut.**
`values.yaml`'s image (`yourorg/myapp`) is a placeholder with nothing
real to pull. Substituted nginx via a dedicated, checked-in overlay file
(not an inline workflow hack), with probes explicitly repointed from
`/healthz`/`/ready` to `/` - the one path nginx actually serves.
Considered `hashicorp/http-echo` first (it answers every request path
with 200 regardless of what's asked), and rejected it specifically
because that would make the probe-path assertion untestable - the job
would pass whether the configured paths were right or completely wrong,
which is closer to papering over a mismatch than resolving it. nginx
genuinely 404s on the wrong path, so this smoke test can actually fail
for the right reason if the wiring is ever broken.

**What I could not verify locally, and said so rather than claiming
otherwise:** Docker Desktop's daemon isn't reachable in this sandbox
(`docker ps` fails identically from both the Bash tool and PowerShell -
not a sandboxing quirk, the daemon itself isn't running here). Could not
actually run `kind create cluster`, `helm install --wait`, `kubectl
wait`, or the endpoints check against a live cluster. What was verified
instead: the smoke values overlay renders correctly end to end (image,
container port, probe paths, and - critically - that the rendered
Service's `selector` and the Deployment's pod-template labels actually
match, which is the exact thing the endpoints-check step exists to
prove); the workflow YAML parses as valid GitHub Actions syntax; and the
`kind` release/download URL for the pinned version resolves for real.
The actual live-cluster run needs to happen for real in GitHub Actions
before trusting this job - flagging that gap explicitly rather than
reporting this as tested.

---

## First real CI run: two failures, both root-caused from actual logs

**`validate` (all three matrix legs) - kubeconform.tar.gz broke every
chart load.** The install step downloaded `kubeconform.tar.gz` straight
into the checkout directory (the chart root) and never deleted it after
extracting the binary. `helm lint`/`helm template` load every file under
the chart root as chart content, and hard-reject anything over 5MB:
```
[ERROR] templates/: chart file "kubeconform.tar.gz" is larger than the maximum file size 5242880
```
Reproduced locally by dropping a 6MB dummy `kubeconform.tar.gz` into the
repo root and running `helm lint .` - failed identically. Fixed two ways:
the install step now downloads and extracts entirely under `/tmp`, never
touching the checkout directory at all (the actual fix); added
`.helmignore` as a second line of defense in case anything else ever
lands loose files in the chart root. Reran the dummy-file repro after
adding `.helmignore` alone (without the /tmp fix) - confirmed it also
independently prevents the failure.

**`kind-smoke-test` - stock nginx cannot run under this chart's
securityContext, full stop.** `helm install` timed out at 3m with a bare
`context deadline exceeded` - true but uninformative on its own, since I
hadn't added any diagnostics for an install-level failure (only for the
endpoints check). Reasoned through why rather than guessing: stock nginx
needs to bind port 80 (needs `CAP_NET_BIND_SERVICE`, which
`capabilities.drop: [ALL]` strips regardless of UID), is built to run as
root (`runAsNonRoot`/`runAsUser` force it not to), and writes cache/pid
files at startup (`readOnlyRootFilesystem: true` blocks that, and the
chart only mounts a writable `/tmp`). Checked whether the smoke overlay
could just relax the security context for this one test - it can't, by
design: `runAsNonRoot`, `readOnlyRootFilesystem`, and
`allowPrivilegeEscalation` are pinned with `const` in
`values.schema.json` specifically so no overlay can weaken them (the
platform-invariant decision in `docs/opinions.md`). Loosening the schema
to work around it would have undone that decision unilaterally, for a
smoke test's convenience.

Switched the smoke overlay to `nginxinc/nginx-unprivileged:1.25.4`
instead - confirmed the image and tag actually exist on Docker Hub via
its API before relying on it. It listens on 8080 (no privileged port, no
capability needed) and is built for exactly this non-root case; it still
needs writable cache/pid paths, supplied via the chart's existing,
already-verified-working `volumeMounts`/`volumes` passthrough - no
template changes. Every security setting stays exactly as strict as the
base chart requires. Verified locally everything short of the live run:
rendered output shows `containerPort: 8080`, the Service's `targetPort`
correctly following it, `securityContext` completely unchanged from the
chart's own defaults, and the extra cache/run emptyDir mounts landing
alongside `/tmp`.

**Could not verify the nginx-unprivileged startup sequence itself** -
still no Docker in this sandbox. Added a "Dump cluster state (on
failure)" step (`kubectl describe`, current + previous container logs,
events, sorted) that runs before cleanup on any failure, specifically so
if this configuration is still wrong in some detail, the next log says
exactly what, instead of another bare timeout.

---

## deploy.yml converted to a reusable workflow with a real approval gate

**What changed:** `deploy.yml` went from a standalone workflow triggered
`on: push: branches: [master]` (deploying straight to a `production`
namespace, no gate - see the finding at the top of this file) to
`on: workflow_call` only, with no trigger of its own at all. A service
repo calls it explicitly, once per environment, via a ~5-line caller
workflow (`examples/sample-api/.github/workflows/deploy.yml`). Pushing
to `master` in *this* repo now deploys nothing, anywhere - confirmed by
checking the file directly: `on:` contains only `workflow_call`.

**The gate itself:** the `deploy` job sets `environment: ${{
inputs.environment }}` dynamically. GitHub Environments do two things at
once here, which is why one mechanism covers both needs: they apply
whatever protection rules are configured for that environment name in
the *calling* repo's own Settings (required reviewers, for `prod` only -
dev and staging are separate environments but carry no reviewer
requirement, see the design note above), and they scope `vars.*` lookups
(`AWS_DEPLOY_ROLE_ARN`, `AWS_REGION`, `EKS_CLUSTER_NAME`) so each
environment can point at its own AWS role and cluster, least-privilege
per environment.

**A gap closed before it could be exploited:** `environment:` is
resolved before any step in that job runs, so validating `inputs.environment`
*inside* the gated job would be too late - a typo'd value (e.g.
"produciton") would make GitHub silently create and use a brand-new,
unprotected ad-hoc environment rather than failing. Added a separate
`validate-inputs` job with no `environment:` set at all, which the
gated `deploy` job `needs:`, that hard-rejects anything except exactly
`dev`/`staging`/`prod` (and separately re-checks `image_tag` isn't empty
or `latest`, even though `values.schema.json` would also catch that -
cheap, and fails faster with a clearer message).

**What's actually verified, and what isn't.** Verified: the workflow
YAML is syntactically valid; `helm lint`/`helm template` against
`examples/sample-api/values-dev.yaml` (the values file the reusable
workflow's own `helm lint`/`helm upgrade` steps would use) render
correctly against the real chart. **Not verified, and can't be from this
sandbox:** the actual OIDC token exchange, whether a real IAM trust
policy accepts it, whether a GitHub Environment's required-reviewer rule
actually blocks the job the way described, or the two-checkout pattern
(service repo + `Yousuf-Hossain10/eks-mega-chart` chart) actually working
end to end in a real cross-repo `workflow_call`. This needs a real
service repo, a real IAM role, and a real GitHub Environment with a
reviewer added, exercised for real - same category of gap as the kind
smoke test, flagged the same way rather than claimed as tested.

**Witnessed on 2026-07-25:** triggering a prod-targeted run of
`deploy.yml` (via the temporary `_test-prod-gate.yml` caller) showed the
`deploy` job sitting at "Waiting for approval" with zero steps executed
- not a step paused mid-run, the job itself never dispatched - confirming
the GitHub Environment protection rule blocks execution before any code
runs, including the OIDC/AWS step. After approving, execution began
immediately and failed downstream at `Configure AWS credentials (OIDC)`
with `Input required and not supplied: aws-region` - no real role/region
configured on `prod` - proving it was the approval gate, not a missing-
config accident, that had been the only thing standing between trigger
and AWS the whole time. (First attempt at this test showed no pause at
all - the `prod` environment existed but its "Required reviewers"
checkbox had never actually been ticked/saved; fixed, then re-tested
with the result above.)

**"Prevent self-review" boundary:** enabled on `prod` as declared policy
- correctly reflecting the intended rule for whenever a second reviewer
exists - but not runtime-verified, and deliberately not pretended to be.
As a solo maintainer, exercising it requires a second real person to
attempt approving their own triggered run and be blocked; that pass was
skipped rather than faked with a throwaway second account for no real
signal. The approval gate mechanism itself (pause before dispatch,
execution only after approval) is witnessed and proven above; self-review
prevention specifically is configured and reasoned about, not watched
happen.

---

## Day 5: security scanning added to ci.yml, and no static image tag is ever clean

**Two new jobs, both verified locally with a real Trivy binary** (downloaded
and run directly, not assumed): `trivy-config-scan` runs Trivy's
misconfiguration scanner against the chart's rendered output, and
`trivy-image-scan` runs its vulnerability scanner against a real pinned
image. Both follow the same "prove it can fail, then prove it passes
clean" discipline as the schema and kind-smoke-test negative tests:
`tests/misconfig-fixture/deliberately-insecure.yaml` (a standalone,
deliberately-insecure Deployment - `privileged: true`, `runAsUser: 0`,
`hostPath: /`, `hostNetwork: true` - not generated by this chart, since
`values.schema.json`'s const-pinned securityContext makes that
structurally impossible) gets 5 real HIGH findings and exit code 1;
`nginx:1.14.0` gets 146 real HIGH/CRITICAL vulnerability findings and
exit code 1. Confirmed both directly before writing the assertions.

**The gating decision:** fail on HIGH/CRITICAL, warn (report, don't
block) below - this was already CLAUDE.md's stated non-negotiable, not a
new choice, but implementing it forced arguing why: LOW/MEDIUM findings
are near-universal in any real image (dozens, routinely) and gating on
them would make the pipeline permanently red for reasons unrelated to
actual exploitability - the "boy who cried wolf" failure mode, where a
check that's always red stops being read. HIGH/CRITICAL is the pragmatic
cutoff where CVSS starts correlating with "remotely exploitable,
meaningful impact."

**The CVE-with-no-fix question turned out not to be hypothetical.**
Scanned the chart's real pinned smoke-test image
(`nginxinc/nginx-unprivileged:1.25.4`) expecting a clean demo pass - got
152 real HIGH/CRITICAL findings instead. Checked whether this was just a
bad pin: tried `1.25.4-alpine` (17, all fixed-available), `1.27.4-alpine`
(37), `1.29.0-alpine` (38), `1.29.2-alpine` (31, the most recent
available) - every single specifically-pinned tag had real findings, all
with fixes already released upstream that just aren't in that particular
frozen build. Only a *floating* tag (`1-alpine`, continuously rebuilt
under the same name) scanned clean - and floating tags are exactly what
this project's "no mutable tags" rule exists to reject (confirmed:
`1-alpine` doesn't even match `values.schema.json`'s semver pattern).
This is not a search failure - it's the actual, verified shape of the
problem: **any statically pinned image tag accumulates real, patchable
CVEs purely from the passage of time**, because new vulnerabilities get
disclosed continuously against already-published, frozen builds. That
reframes "block forever or explicit ignore file" from a hypothetical
into the literal daily-driver reality of running a pinned-tag policy at
all.

**Resolution, and why:** two mechanisms, not one, because they answer
different questions. `--ignore-unfixed` (a real Trivy flag, its behavior
confirmed directly, not assumed from the `--help` text alone) handles
"there is no fix anywhere" automatically - correct to never block on
that, since there's nothing anyone can do about it today. `.trivyignore`
handles the different, more common case this investigation actually
surfaced: a fix *does* exist, just not yet in this pinned build. Every
entry in `.trivyignore` is a real CVE ID from a real scan on 2026-07-25
against `nginxinc/nginx-unprivileged:1.29.2-alpine` (the chosen pin -
alpine-based specifically for a smaller package surface, confirmed:
152→31 HIGH/CRITICAL just from switching base OS at the same nginx
version), with a documented, dated review-by (2026-08-25) rather than a
silent, permanent allowlist. Verified: with both mechanisms applied,
this real image scans to exactly zero findings and exit code 0 - not
faked, not cherry-picked to look clean, actually zero.

**Deliberately did not touch `tests/values-smoke.yaml`'s pin** even
though it's the same image family. That pin (`nginx-unprivileged:1.25.4`,
Debian-based) is proven working end-to-end against a real kind cluster -
changing it now risks a regression with no way to verify locally (still
no Docker in this sandbox). The image-scan job uses its own, separately
pinned `TRIVY_SCAN_IMAGE`, chosen only to prove the scanning mechanism
against something real, not to match the smoke test.

**What it taught me:** "we scan" without a stated threshold and an
answer for the no-fix case isn't a security control, it's a checkbox -
and the specific numbers here (0 → 152 → 31 → 0, across four real pin
attempts) are a better argument for why a pinned-image renewal process
has to be an ongoing operational habit, not a one-time task, than any
amount of abstract reasoning would have been.

---
