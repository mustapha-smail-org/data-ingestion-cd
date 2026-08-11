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
.github/
  CODEOWNERS                  Requires data-platform-team review for production.yaml
  workflows/deploy.yml        Thin wrapper: updates state, calls deploy-render.yml
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
  deliberately **not** stored in `environments/*.yaml` — real Render IDs are
  effectively an access handle to that service, so `deploy.yml` resolves the
  right one per environment from these secrets at deploy time instead of keeping
  them in Git history. See `.github/workflows/deploy.yml`'s "Resolve Render
  service ID" step.
- The `main` branch ruleset must list `city-pulse-automation` in its **bypass
  list** with mode **Always** (`Settings → Rules → Rulesets`), or every
  `deploy.yml` run fails to push the desired-state commit.

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

`deploy.yml` updates the target `environments/<env>.yaml` file, commits it with the
automation identity, then calls `deploy-render.yml` in `deployment-workflows` to
actually trigger and verify the Render deployment.

## Rollback

1. Find the previous known-good digest in this repo's Git history
   (`git log -p environments/<env>.yaml`).
2. Manually run `deploy.yml` (`workflow_dispatch`) with that digest, the
   corresponding source commit, and a tag noting the rollback (e.g.
   `rollback-v1.2.0`) — or open a PR restoring the previous file content for
   `production.yaml`.
3. No rebuild happens; the same image digest is redeployed.
4. Record the reason in the commit/PR description.
