# data-ingestion-cd

Desired deployment state for the `data-ingestion` service. This repo does not
contain application code or build logic — it declares which immutable container
image digest should run in each environment, and delegates the actual deployment
mechanics to the reusable workflows in
[`deployment-workflows`](https://github.com/mustapha-smail-org/deployment-workflows).

## Layout

```
service.yaml                  Provider + health-check configuration
environments/
  dev.yaml                    Auto-updated on every merge to data-ingestion main
  staging.yaml                Auto-updated when a semantic release tag is created
  production.yaml             Updated only via reviewed pull request
config/
  dev.yaml                    App config mounted onto Render (secrets as tokens)
  staging.yaml
  production.yaml
.github/
  CODEOWNERS                  Requires data-platform-team review for production.yaml
  workflows/deploy.yml        Updates state, pushes config, calls deploy-render.yml
```

`service.yaml` and each `environments/*.yaml` conform to the schemas in
`deployment-workflows/contracts/`.

## Required repo configuration

- **Variable** `AUTOMATION_APP_ID` and **secret** `AUTOMATION_APP_PRIVATE_KEY` —
  yes, on *this* repo too, not just `data-ingestion`. `deploy.yml` authenticates as
  the `city-pulse-automation` App before checkout/push so the branch ruleset's App
  bypass entry actually applies; without these, `deploy.yml` falls back to the
  default `GITHUB_TOKEN` (identity `github-actions[bot]`), which is a different
  actor than the bypass list allows, and the push is rejected with `GH013`.
- **Secret** `RENDER_API_KEY` — used by `deploy-render.yml`.
- **Secrets** `RENDER_SERVICE_ID_DEV`, `RENDER_SERVICE_ID_STAGING`,
  `RENDER_SERVICE_ID_PRODUCTION` — the actual Render `srv-...` service IDs (not
  the human-readable service name shown in the Render dashboard URL). These are
  deliberately **not** stored in `environments/*.yaml`, and deliberately **not**
  resolved in a `run:` step either — the `deploy` job's `secrets:` block picks
  the right one via a plain `&&`/`||` expression on `inputs.environment`
  (evaluated before any job runs). Routing a secret through a step/job output
  into another job's `with:`/`secrets:` doesn't work: GitHub Actions silently
  blanks any job output derived from a masked value once it crosses that
  boundary (see `deployment-workflows/docs/WORKFLOW_CONTRACTS.md`, "Secret job
  outputs get blanked, not passed through").
- The `main` branch ruleset must list `city-pulse-automation` in its **bypass
  list** with mode **Always** (`Settings → Rules → Rulesets`), or every
  `deploy.yml` run fails to push the desired-state commit.
- **Optional secrets**, one `<KEY>_DEV`/`<KEY>_STAGING`/`<KEY>_PRODUCTION` triplet
  per real secret this service needs — currently `KAFKA_USERNAME`,
  `KAFKA_PASSWORD`, `SCHEMA_REGISTRY_USERNAME`, `SCHEMA_REGISTRY_PASSWORD` (12
  secrets total, 4 keys × 3 environments). Never committed; each resolved
  per-environment via a plain `&&`/`||` expression (same pattern as
  `RENDER_SERVICE_ID_*` above) inside `deploy.yml`'s `push-config` job. Leave
  any unset if that particular environment/key combination isn't needed. This
  list is unbounded — adding a 5th secret is one more `env:` line in
  `push-config`, nothing else changes (see below).

## External app config (`config/*.yaml`)

`deploy.yml`'s `push-config` job resolves `config/<env>.yaml` and mounts the
result onto Render at `/etc/secrets/application.yaml`, before the `deploy` job
triggers the image deploy. `data-ingestion`'s `Dockerfile` sets
`SPRING_CONFIG_IMPORT=optional:file:/etc/secrets/application.yaml`, so Spring
Boot loads it automatically — the `optional:` prefix means a missing file is
a silent no-op, not a startup failure.

Keys in this file satisfy the same `${VAR}`-style placeholders already used
in `data-ingestion`'s `application.yaml` (e.g. `KAFKA_BOOTSTRAP_SERVERS`) —
Spring resolves placeholders from any property source, file or env var,
identically.

**Secret values stay in this same tracked file as placeholder tokens, not
real values** — e.g. `KAFKA_PASSWORD: "%%SECRET:KAFKA_PASSWORD%%"`. This is
**deliberately not** a `deploy-render.yml` capability — see
`deployment-workflows/docs/WORKFLOW_CONTRACTS.md`, "Externalized App Config",
for why a shared reusable workflow is the wrong place for this. Instead,
`push-config` does it inline:

1. Declares one `env:` line per secret this service needs
   (`SECRET_KAFKA_USERNAME: ${{ ... }}`, etc.) — the only part that grows with
   secret count.
2. A generic bash loop (`for VAR_NAME in $(compgen -v | grep '^SECRET_')`)
   substitutes matching `%%SECRET:NAME%%` tokens — unchanged no matter how many
   secrets exist.
3. Fails loudly if any `%%SECRET:...%%` token remains unresolved, rather than
   pushing a broken placeholder to Render as if it were real.
4. Pushes the resolved content to Render directly via `curl`, in that same
   script — never written to a file, never exposed as a job output, so it can't
   hit the masked-output-blanking issue described above.

Adding a 5th secret: one new `SECRET_<NAME>` line in `push-config`'s `env:`
block, plus the matching `<NAME>_DEV`/`_STAGING`/`_PRODUCTION` repo secrets. No
changes needed to `deployment-workflows` at all.

## Render Free Plan note

This organization currently has 2 Render services available (free plan), so
`RENDER_SERVICE_ID_DEV` and `RENDER_SERVICE_ID_STAGING` currently hold the **same**
value. `RENDER_SERVICE_ID_PRODUCTION` points at a separate service. When upgrading
to a paid Render plan, create a dedicated staging service and update the
`RENDER_SERVICE_ID_STAGING` secret to its ID — no workflow or environments/*.yaml
changes are needed.

## How deployments happen

1. **Dev:** `data-ingestion`'s `main.yml` workflow (via the `ci-main.yml` template)
   builds an image and dispatches `deploy.yml` here with `environment=dev`.
2. **Staging:** `data-ingestion`'s `release.yml` workflow (via `ci-release.yml`)
   re-tags the existing image for the released commit and dispatches `deploy.yml`
   here with `environment=staging`. No image is rebuilt.
3. **Production:** open a PR here updating `environments/production.yaml` with the
   digest/tag/commit that was validated in staging. Requires CODEOWNERS approval.
   Merging triggers `deploy.yml` with `environment=production` (wire a
   `push`-to-`main`-with-path-filter trigger, or run `deploy.yml` manually via
   `workflow_dispatch`, once the promotion flow is finalized).

`deploy.yml` runs three jobs in sequence: `update-desired-state` (updates and
commits the target `environments/<env>.yaml`), `push-config` (resolves
`config/<env>.yaml` and mounts it on Render — see above), then `deploy` (calls
`deploy-render.yml` in `deployment-workflows` to actually trigger and verify the
image deployment).

## Rollback

1. Find the previous known-good digest in this repo's Git history
   (`git log -p environments/<env>.yaml`).
2. Manually run `deploy.yml` (`workflow_dispatch`) with that digest, the
   corresponding source commit, and a tag noting the rollback (e.g.
   `rollback-v1.2.0`) — or open a PR restoring the previous file content for
   `production.yaml`.
3. No rebuild happens; the same image digest is redeployed.
4. Record the reason in the commit/PR description.
