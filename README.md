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

## Render Free Plan note

This organization currently has 2 Render services available (free plan), so `dev`
and `staging` both point at `data-ingestion-dev-staging`. `production` has its own
service, `data-ingestion-prod`. When upgrading to a paid Render plan, create a
dedicated staging service and update `environments/staging.yaml`'s
`provider.serviceId` — no workflow changes are needed.

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
