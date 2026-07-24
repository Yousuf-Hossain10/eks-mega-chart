# Golden Path for Amazon EKS

## What this project is

An opinionated platform repository providing **one supported route from source code to a running service on Amazon EKS**.

The Helm chart in `charts/service` is the engine. The product is the scaffold script, the reusable CI/CD workflow, and the documented opinions — the things that let a developer ship a service without needing to know Helm.

This repository is a portfolio project for a cloud engineering role. **I must be able to defend every line of it in a technical interview.** That constraint outranks speed.

## Repository layout

```
charts/service/          The Helm chart (engine)
  values.schema.json     Validates the values contract at install time
  templates/
  tests/                 helm unittest specs
environments/            values.dev.yaml, values.staging.yaml, values.prod.yaml
scaffold/new-service.sh  Generates a new service from template
examples/
  sample-api/            HTTP service, has ingress
  sample-worker/         No ingress, no Service — proves the abstraction
.github/workflows/
  ci.yml                 lint + template + unittest + kind smoke test
  deploy.yml             Reusable (workflow_call), consumed by service repos
docs/
  opinions.md            What is decided for you, and why
  runbook.md             Built from real failures, not theory
  adr/                   Architecture decision records
NOTES.md                 Running log of what broke and how I fixed it
```

---

## Non-negotiable rules

### Correctness
- Every change must keep `helm lint`, `helm template` and `helm unittest` passing for **all** environment values files. Run them before claiming a task is done. Do not assert something works without having executed it.
- Never reference a values key from a template without that key existing in `values.yaml`. This repo has already shipped one nil-pointer bug that way (`.Values.pdb.enabled` with no `pdb` key). Guard optional blocks with defaults.
- If a template references a ConfigMap or Secret, the chart must not render a Deployment pointing at a resource that was conditionally skipped.
- Never put a `#` comment on the same line as a Helm action — it leaks into rendered YAML. Use `{{/* ... */}}` or put it in docs.

### Security
- **No static cloud credentials anywhere.** GitHub Actions authenticates to AWS via OIDC role assumption only. No `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`.
- No real secrets in values files, templates, or Git history. Placeholders must be obviously fake, and the real mechanism must be documented.
- Containers run: `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, `seccompProfile: RuntimeDefault`. Read-only root filesystem requires a writable `emptyDir` at `/tmp`.
- No AWS account IDs, cluster names, role ARNs or hostnames committed to this repo. Those go in `CLAUDE.local.md` (gitignored) or are passed at deploy time.

### Platform opinions
- Image tags are **git SHAs**. Never `latest`. The schema should reject it.
- Every service gets liveness, readiness **and** startup probes. No opt-out.
- Resource strategy: set CPU requests but **no CPU limit**; set memory request equal to memory limit. See `docs/adr/0002-resource-limits.md`. If you disagree, argue it — don't silently change it.
- Dev deploys automatically. Prod requires a manual approval gate.
- Rollback is `helm rollback`, documented and actually tested.
- Everything reaches the cluster through the pipeline. No manual `kubectl apply`.
- Scans gate the pipeline: fail on HIGH/CRITICAL, warn below.

---

## How I want you to work

- **Explain the reasoning** for each change alongside making it. I have to defend this in an interview, so a change I don't understand is worse than no change.
- **When there is a real trade-off, present both sides and let me decide.** Do not silently pick. This applies especially to the resource-limits decision, secret management, and topology spread strategy.
- **If you find a bug I didn't ask about, tell me — don't silently fix it.** I want to know what was wrong and why it mattered. Finding my own bugs is part of the point of this project.
- **Report before fixing** when I ask for an audit. Give me the findings list first.
- Make small, focused commits with messages that explain *what was wrong*, not just what changed.
- Prefer boring, standard solutions over clever ones. This is infrastructure.
- **Verify file existence with `git ls-files`, not directory listings.** Directory/glob-based scans (`find`, `ls`) can silently miss dot-directories like `.github/` — an exclusion pattern meant to skip `.git` can accidentally also match `.github` and hide real, tracked files. `git ls-files` is the source of truth for what's actually in the repo.
- Don't claim something is "production-grade" or "tested" unless it has actually been deployed and exercised. Say what has and hasn't been run.
- When I'm deploying to a real cluster, give me a checklist **I** execute — don't automate it away. The failures are the deliverable.

---

## Commands

```bash
# Lint every environment
helm lint charts/service -f environments/values.dev.yaml
helm lint charts/service -f environments/values.staging.yaml
helm lint charts/service -f environments/values.prod.yaml

# Render every environment (must produce clean YAML, no stray comments)
helm template test charts/service -f environments/values.dev.yaml

# Unit tests
helm unittest charts/service

# Validate rendered manifests against Kubernetes schemas
helm template test charts/service -f environments/values.dev.yaml | kubeconform -strict -summary

# Local smoke test
kind create cluster --name golden-path
helm install smoke charts/service -f environments/values.dev.yaml --wait --timeout 5m
helm uninstall smoke
kind delete cluster --name golden-path

# Scaffold a new service
./scaffold/new-service.sh
```

---

## Definition of done for any task

1. `helm lint`, `helm template` and `helm unittest` pass for all three environments
2. Rendered YAML is clean — no leaked comments, no placeholder values
3. Any new behaviour has a unit test covering it
4. Any decision with a real trade-off has an ADR
5. Anything that broke on the way is written into `NOTES.md`
6. I have read the diff and can explain it

---

## Current known state

Carried over from the original `eks-mega-chart`, verify these are fixed before building on top:

- [ ] `templates/pdb.yaml` referenced `.Values.pdb.enabled` with no `pdb` key in values — chart failed to render
- [ ] CI triggered on branch `main` while the default branch was `master` — the workflow had never run
- [ ] `helm upgrade` command had a missing line continuation before `--timeout`, so it executed as a separate shell command
- [ ] CI paths pointed at `./eks-mega-chart` while `Chart.yaml` was at the repo root
- [ ] `envFrom` referenced ConfigMap/Secret unconditionally while those templates were conditional
- [ ] `readOnlyRootFilesystem` was never set despite being claimed
- [ ] `startupProbe` was supported by the template but never defined in values
- [ ] `affinity: {}` — no multi-AZ spread despite being claimed
- [ ] No explicit `strategy` block — rolling update was the implicit default, not a decision
- [ ] `values-prod.yaml` contained a literal `"{{ .Values.secretData.DB_PASSWORD }}"` string that would be base64-encoded verbatim
- [ ] Static AWS access keys in the workflow instead of OIDC
