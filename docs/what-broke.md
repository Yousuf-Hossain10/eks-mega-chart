# What Broke (and How I Found Out)

Interview story bank. Every entry here is something that looked correct —
sitting in the repo, passing a quick glance, or just never having been
questioned — and turned out not to be. Full detail and exact commands are
in `NOTES.md`; this is the compressed version to say out loud.

---

**The wrong-branch "gate."** `deploy.yml` deploying straight to production with zero
approval looked like it hadn't mattered yet purely by luck — and it hadn't, but not
for a safety reason: it triggered on `push: branches: [main]` while this repo's
actual default branch is `master`, so every real push had silently done nothing.
Found by finally reading the file directly (an earlier scan's exclusion glob,
`./.git*`, had accidentally also matched and hidden `./.github`); fixed for real
in Day 4 by rebuilding it as a gated, reusable `workflow_call` workflow instead of
just correcting the branch name.

**Four bugs in a workflow that looked like a normal deploy pipeline.** Static AWS
keys instead of OIDC, the wrong branch above, `./eks-mega-chart` as the chart path
when `Chart.yaml` was at the repo root, and a missing `\` line continuation that
would have silently split `--timeout 5m0s` into its own invalid shell command. None
subtle — the tell was that the file had clearly never been triggered even once, so
nothing had ever forced these bugs to surface.

**`templates/pdb.yaml`'s nil pointer.** The chart looked like it should render —
every other template had sane defaults — but `.Values.pdb.enabled` was referenced
with no `pdb:` key anywhere in `values.yaml` at all, so `helm lint` failed
immediately on a nil pointer, before producing a single line of YAML. Found by
just running `helm lint` for the first time; fixed by adding the missing key with
`enabled: false` as the real default.

**`envFrom` referencing resources that were never created.** `helm lint`/`helm
template` both passed clean with `configData: {}`, which looked like proof the
chart was fine — but `configmap.yaml` correctly skips rendering when empty while
`deployment.yaml`'s `envFrom` named it unconditionally, a mismatch no syntax
checker can see. Found by deliberately emptying the values and grepping the
rendered output by hand; on a real cluster this is `CreateContainerConfigError`,
invisible until runtime. Fixed by making `envFrom` conditional on the same values
its sibling templates already check.

**A `.gitignore` that ignored nothing.** `.gitignore.txt` looked like a working
ignore file sitting in the repo — it wasn't: the `.txt` extension meant git never
read it as one, and it was also completely empty. Found via a `git ls-files` audit;
fixed by renaming it and actually populating it.

**A read-only-root-filesystem mount that could never fire.** The `/tmp` `emptyDir`
mount code already existed in `deployment.yaml`, which looked like the feature was
half-built and just needed a flag flip — but it was gated behind
`.Values.volumeMounts`, an unrelated list that defaults to empty in every
environment, so flipping `readOnlyRootFilesystem: true` alone would have shipped a
container with no writable `/tmp` at all. Found while wiring the flag up for real
and reading the existing template closely instead of trusting that it looked done.

**The schema that broke its own chart on day one.** Adding `values.schema.json` to
enforce real git-SHA image tags looked like a pure, additive safety improvement —
it immediately broke `helm lint .` with zero overrides, because `values.yaml`'s own
default was `image.tag: "latest"`. There is no such thing as a valid default git
SHA; fixed by removing the default entirely and requiring every render, dev
included, to pass a real tag.

**The bound JSON Schema can't express.** `values.schema.json` looked like it could
enforce any values constraint — it can't compare two sibling fields to each other
at all, so `minReplicas: 5, maxReplicas: 2` passes the schema cleanly, each number
individually valid. Found by reasoning through what draft-07 JSON Schema actually
supports before assuming it could; fixed with a template-side `{{ fail }}` guard in
`hpa.yaml`, a completely different enforcement mechanism running at a different
stage of the pipeline.

**A lint warning that had been ignored the entire session.** `helm lint` had been
printing "document starts with an illegal indent" since the very first run, and it
kept looking like harmless noise — it wasn't: four templates had `# comments`
sitting on the same line as a `{{- if }}`/`{{- with }}` action, which leak into
rendered output as literal text. In one case a comment about annotations landed
three lines away, attached to an unrelated label, because of how whitespace
trimming interacted with the next line. Found by finally grepping rendered output
for `#` instead of eyeballing it.

**A password that was never a password.** `values-prod.yaml` had
`DB_PASSWORD: "{{ .Values.secretData.DB_PASSWORD }}"`, which looked like a Helm
template reference — values files aren't templated by Helm at all, so it was the
literal 44-character string, base64-encoded verbatim into the real Secret object.
The "obvious" fix, `secretData: {}`, looked like it would clear the fake dev
values — it doesn't; Helm merges maps recursively, so an empty overlay changes
nothing and the fake dev password would have silently shipped under a `prod`
label. Only `secretData: null` actually unsets an inherited map, confirmed by
rendering both ways and counting the resulting `Secret` objects.

**Values files that quietly contradicted a decision once it was written down.**
Nothing failed `helm lint` or schema validation with a CPU limit set and a memory
request lower than its limit — it looked like a reasonable, unexamined default.
Writing ADR-0002 (CPU: request only, no limit; memory: request == limit) is what
actually caught it: the values files had shipped a different, undocumented policy
the whole time, and only checking them *against a written decision* surfaced the
mismatch, not any tool.

**"Add a CPU limit to prevent starvation."** That comment, sitting in `values.yaml`,
sounded reasonable and is a common belief — it's backwards. A CPU *limit* caps what
a container can use regardless of contention; the thing that actually protects
against starvation under contention is the *request* (cgroup shares), which exists
independent of any limit. Worth naming explicitly in the ADR rather than just
deleting the comment, since it's the exact wrong-but-plausible reasoning someone
else would reach for again.

**A comment claiming a working feature was disabled.** `targetMemoryUtilizationPercentage`
had a comment reading "Uncomment and enable for memory-based scaling" while the
line itself was already active and wired end-to-end in `hpa.yaml`. Found by
rendering the HPA with autoscaling on and checking both metrics actually appeared —
they did. The feature was fine; the comment was just lying about it, which is worse
than no comment, since it points the next reader at "fixing" something that isn't
broken.

**Helm's own "latest" isn't what it used to mean.** Pinning nothing for the
`helm-unittest` plugin install looked safe — `helm/helm`'s own "latest" release tag
is now a v4 release, and Helm v4 plugin manifests use a schema (`platformHooks`)
that a v3 binary can't even parse. Found by trying to install the plugin locally
before writing the CI job, not by trusting `azure/setup-helm@v4`'s unpinned
default; fixed by pinning both Helm and the plugin to specific v3-compatible
versions.

**A CI helper that broke every job it was meant to support.** The `kubeconform`
install step downloaded its tarball straight into the checkout directory and never
deleted it — looked like a normal binary install, and passed every local test,
because locally the download always went to an isolated `/tmp`. On the first real
CI run it broke all three `validate` matrix legs at once: Helm loads every file
under the chart root as chart content and hard-rejects anything over 5MB.
Reproduced the exact failure locally by dropping a dummy 6MB file in the repo root;
fixed by isolating the download to `/tmp` and adding `.helmignore` as a second line
of defense.

**Stock nginx couldn't run under the chart's own security posture.** Using plain
`nginx:1.25.4` as the kind-smoke-test's stand-in image looked like the obvious
choice — any small public image should do. It structurally cannot start under this
chart's hardened `securityContext`: it needs root plus `CAP_NET_BIND_SERVICE` to
bind port 80, and `capabilities.drop: [ALL]` strips that regardless of user. The
first real CI run just showed a bare `context deadline exceeded`; reasoned through
why rather than guessing, then switched to `nginxinc/nginx-unprivileged`, a variant
built for exactly this case, with zero loosening of the chart's own settings.

**No pinned image tag is ever actually clean.** Scanning the chart's real
pinned demo image for the Day 5 security-scan job was expected to be a quick,
clean pass — it returned 152 real HIGH/CRITICAL findings. Tried four more
specifically-pinned tags looking for a clean one; every single one had real,
patchable findings, because new CVEs get disclosed continuously against
already-frozen builds regardless of how recently the tag was cut. Only a
floating tag came back clean, and floating tags are exactly what this
project's own tag-immutability rule exists to reject. Resolved with two
separate mechanisms: `--ignore-unfixed` for CVEs with no patch anywhere, and a
dated, reviewed `.trivyignore` for CVEs that have a fix just not yet in this
build.

**Branch protection looked like the fix; it wasn't.** Asked to link a red scan to a
blocked deploy "the same way we closed the approval gate," branch protection on
this repo's `master` looked like the obvious, parallel answer. It isn't: required
status checks only gate merges *into this repo* — they don't stop a caller
overriding which chart ref they pull, and more fundamentally they never touch a
service's own application image, only this repo's chart template. The real fix was
a `needs:` link inside `deploy.yml` itself, scanning the actual image about to be
deployed.

**The "contradiction" that turned out to be two different jobs.** Said in one
message that `ci.yml` and `deploy.yml` have zero coupling, then two messages later
said enforcement lives in `deploy.yml`'s own `needs:` link — challenged directly on
whether those statements contradicted. They didn't, but only because they were
about two different scan jobs I hadn't distinguished clearly enough out loud:
`ci.yml`'s own scans stayed uncoupled the whole time, unchanged; `scan-image` is a
third, separate job living inside `deploy.yml`, same-file `needs:`, never a
cross-workflow reference (which isn't even possible). Verified with `grep` and the
actual job list before answering, not from memory.

**An approval gate that looked configured but wasn't enforcing anything.** Set
`environment: prod` in `deploy.yml` and expected a triggered run to pause for
review — the first real trigger ran straight through with no pause at all. The
`prod` GitHub Environment existed with the right name, but its "Required
reviewers" checkbox had been checked in the UI without ever clicking the separate
"Save protection rules" action, so no rule was actually active. Found by triggering
it live and watching nothing happen; fixed by re-saving the rule and re-triggering,
which then correctly paused at "Waiting for approval" with zero steps executed
before approval.

**A branch explicitly labeled "never merge" reached `master` anyway.** A scratch
branch (`test/scan-gate-fail`) carried "never merge to master, temporary" in its
own commit message and file headers — that looked like enough. It wasn't: GitHub's
routine post-push "Create a pull request" prompt got clicked through, and PR #2
landed both scratch files directly in the real tree. Caught only because the next
`git push` came back rejected as diverged, and reading exactly what `origin/master`
had gained (`git show --stat` on the merge commit) before reacting, instead of
force-pushing past the rejection. Fixed with a plain subtractive commit removing
the two files — no rebase, no force-push, nothing rewritten. The real lesson:
nothing automated was ever going to catch this (branch protection can't stop a
voluntary PR merge nobody required), so the actual safety net was reading the diff
before acting on a push rejection, not any gate in the pipeline.

**Required status checks turned out to gate everyone except the one person who
mattered.** Pushing the merge-and-cleanup commit straight to `master` printed
`Bypassed rule violations for refs/heads/master: 5 of 5 required status checks are
expected` — a live, first-person hit of a caveat that had only ever been discussed
theoretically, back on Day 4, in the OIDC/approval-gate interview answer: a repo
admin's own push bypasses required status checks by default, no matter how
correctly they're configured. It looked like a blanket rule; the real scope is
"gates everyone except whoever holds admin," and closing that particular gap is a
repo-permissions decision — who's allowed to be an admin — not anything branch
protection settings or workflow YAML can express.
