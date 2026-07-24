# Getting real secrets into production

## The bug this doc exists because of

`values-prod.yaml` used to contain:

```yaml
secretData:
  DB_PASSWORD: "{{ .Values.secretData.DB_PASSWORD }}"
  API_KEY: "{{ .Values.secretData.API_KEY }}"
```

Values files are plain YAML. Helm does not template them. That string was
never evaluated as a Helm expression — it was the literal 44-character
string `{{ .Values.secretData.DB_PASSWORD }}`, base64-encoded verbatim
into the rendered `Secret`. Anyone who decoded it would find the template
syntax itself sitting where a password was supposed to be. This chart has
never had a real secret-injection mechanism for production; it had a
placeholder that looked like one.

`values-prod.yaml` now sets `secretData: null` instead — which correctly
unsets the fake dev placeholders on merge (an empty `{}` would not; Helm
merges maps recursively, so `{}` contributes no overrides and the dev
fake values leak through). A prod install today renders no `Secret` and
no secret-derived env vars at all. That's an honest gap, not a fixed one.
Below are the real options for closing it. I'm not picking one — this is
exactly the kind of decision CLAUDE.md says to argue out loud, not decide
silently.

## Option 1: External Secrets Operator (ESO)

A controller running in the cluster that reads from a real secret store
(AWS Secrets Manager, SSM Parameter Store) and writes a native Kubernetes
`Secret` from it. You define an `ExternalSecret` resource pointing at the
AWS secret path; ESO keeps the K8s `Secret` in sync on a refresh interval.

**Pros:** Secret material never touches values files, CI logs, or git,
at any point. Rotation in AWS Secrets Manager propagates to the cluster
automatically on ESO's refresh interval, no redeploy needed. Uses IRSA
(the same OIDC-based AWS auth this project already requires elsewhere) —
no separate credential story to build.

**Cons:** Another controller to install, upgrade, and watch for CVEs —
real operational surface. Namespace/RBAC design needed so `ExternalSecret`
in one namespace can't read another team's AWS secret path. Doesn't fit
cleanly into `secretData` as this chart's abstraction works today — the
chart's own `secretData` map and templates/secret.yaml would need to be
bypassed for prod, with a separate `ExternalSecret` template added
instead. That's a real chart change, not just a values change.

## Option 2: Sealed Secrets (Bitnami)

Encrypt a Secret client-side with `kubeseal`, using a public key published
by a controller running in the target cluster. Commit the resulting
ciphertext (`SealedSecret` CRD) to git. Only the controller's private key,
which never leaves the cluster, can decrypt it.

**Pros:** Ciphertext is safe to commit — genuinely git-native, fits the
"everything goes through the pipeline" rule cleanly, no external service
dependency at deploy time. Simple mental model: encrypt once, commit,
the cluster does the rest.

**Cons:** Ciphertext is bound to one specific cluster's key — re-encrypt
for every cluster (dev/staging/prod all need separate `kubeseal` runs
against separate clusters). Rotating a secret means re-running `kubeseal`
and committing again, not just updating a value in AWS. Doesn't fit
`secretData` as-is either — same structural mismatch as ESO, needs its
own template.

## Option 3: SOPS (encrypt the values file itself)

Encrypt the `secretData` block (or a separate `secrets.prod.yaml`) with
`sops`, backed by AWS KMS or `age`. Commit ciphertext. A CI step (or the
`helm secrets` plugin) decrypts it to plaintext just before `helm
upgrade` runs, so plaintext only ever exists transiently in the CI
runner's memory/filesystem for that one job.

**Pros:** Fits this chart's existing `secretData` shape with almost no
template change — it's still "a values file with a map in it," just
encrypted at rest. KMS-backed, so the same IAM/OIDC role story applies.
Diffable structure in git (SOPS preserves keys, encrypts only values), so
`git log` on the encrypted file still shows *which key* changed, just not
the value.

**Cons:** Plaintext exists somewhere at deploy time, even if briefly — a
compromised CI runner during that window is a real (if narrow) exposure
ESO never has, since ESO's Secret material never passes through CI at
all. Requires a CI step and a plugin/tool dependency (`sops`, or `helm
secrets`) that isn't in `deploy.yml` today. Key management (who can
decrypt) becomes its own access-control surface, separate from IAM.

## Option 4: `--set` at deploy time, sourced from GitHub Actions secrets

Store `DB_PASSWORD`/`API_KEY` as encrypted GitHub Actions repository
secrets. `deploy.yml` passes them at install time:
`--set secretData.DB_PASSWORD=${{ secrets.DB_PASSWORD }}`. Nothing is
ever written to a values file.

**Pros:** Zero new infrastructure — no controller, no KMS key, no new
CLI tool. Reuses a mechanism this repo already has (GitHub Actions
secrets are how `AWS_DEPLOY_ROLE_ARN` etc. already flow in). Simplest to
explain and defend in an interview.

**Cons:** GitHub Actions secrets aren't a real secret *store* — no
rotation policy, no fine-grained access audit trail beyond "who can edit
repo settings," no automatic propagation if the secret changes elsewhere.
`--set` puts the value on the command line inside the CI job; it doesn't
appear in `git`, but it can appear in CI job logs if anything echoes the
command (real risk, avoidable but not automatic). Every consuming
service repeats this wiring by hand in its own workflow — doesn't scale
the way a shared controller (ESO) or a shared encrypted-file convention
(SOPS) does across many services.

## What they have in common

None of these can be validated by `values.schema.json` — a schema
describes shape, not where the bytes actually came from or whether
they're real. Whichever mechanism is chosen, `secretData`'s existing
shape (a flat string map) stays compatible with option 3 and 4 without
template changes; options 1 and 2 mean `templates/secret.yaml` stops
being the thing that creates the prod Secret at all, and a different
template (`ExternalSecret` or `SealedSecret`) takes over that job for
prod while dev keeps using the existing `secretData` path for its fake
placeholders.
