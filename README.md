# ci-workflows

Central reusable GitHub Actions workflows for repositories owned by
[`yongchenglow`](https://github.com/yongchenglow).

The repository follows the same structure as the NUH CI workflow library:
reusable workflows live directly in `.github/workflows`, their caller interface is
documented here, Renovate maintains action versions, and `actionlint` validates every
workflow change.

## Workflows

| Workflow | Purpose |
| --- | --- |
| `reusable-build.yml` | Run Bun lint, type-check, tests, dependency checks, and a Trivy filesystem scan. |
| `reusable-secret-scan.yml` | Scan Git history with Gitleaks. |
| `reusable-docker.yml` | Build and optionally push the application image. |
| `reusable-security-scan.yml` | Scan the published image and fail on critical vulnerabilities. |
| `production-deploy.yml` | Deploy the image with Helm and configure production Cloudflare routing. |
| `production-rollback.yml` | Validate and roll back the production Helm release. |
| `review-deploy.yml` | Allocate a stable NodePort and deploy a pull-request review environment. |
| `review-cleanup.yml` | Remove review Kubernetes and Cloudflare resources and release its NodePort. |

## Consumer usage

Caller repositories retain event triggers and job orchestration. A calling job must
grant permissions at least as broad as the called workflow and pass repository secrets
with `secrets: inherit` where required.

```yaml
jobs:
  build:
    permissions:
      contents: read
      security-events: write
    uses: yongchenglow/ci-workflows/.github/workflows/reusable-build.yml@main
    with:
      bun_version: 1.2.22

  docker:
    needs: build
    permissions:
      contents: read
      packages: write
    uses: yongchenglow/ci-workflows/.github/workflows/reusable-docker.yml@main
    with:
      bun_version: 1.2.22
      platforms: linux/amd64
    secrets: inherit
```

GitHub evaluates the `github` context against the caller and `actions/checkout` checks
out the caller repository. This lets deploy workflows use the consumer's Helm chart,
repository variables, environments, image package, and deployment history.

## Caller contract

### Build and scanning

- `reusable-build.yml` accepts optional `bun_version` and expects the scripts `lint`,
  `typecheck`, `test`, and `knip`, plus `bun.lock`.
- `reusable-secret-scan.yml` accepts optional numeric `fetch_depth` (default `0`, full
  history).
- `reusable-docker.yml` accepts `bun_version`, `registry` (default `ghcr.io`),
  `image_name`, `platforms` (default `linux/amd64`), `tag_prefix`, and `push` (default
  `true`). It outputs `image_tag` and `image_name`. The caller supplies
  `DHI_REGISTRY_USERNAME` and `DHI_REGISTRY_PASSWORD` through inherited secrets.
- `reusable-security-scan.yml` requires `image_tag`; `registry` and `image_name` are
  optional. It reports critical and high findings to code scanning and blocks critical
  vulnerabilities.

### Production operations

- `production-deploy.yml` requires `image_tag`, inherited Kubernetes and Cloudflare
  secrets, the `CLOUDFLARE_DOMAIN` repository variable, and `helm/app` in the caller.
  It deploys release `web` to namespace `personal-site` on NodePort `30000`.
- `production-rollback.yml` requires inherited Kubernetes secrets and the
  `CLOUDFLARE_DOMAIN` repository variable. It validates the latest GitHub deployment
  before rolling back release `web` in namespace `personal-site`.

### Review applications

- `review-deploy.yml` requires `image_tag` and a `review_id` matching `pr-[0-9]+`, plus
  inherited Kubernetes and Cloudflare secrets and `CLOUDFLARE_DOMAIN`. It allocates a
  stable NodePort in `31000-31999`, deploys the caller's `helm/app` chart, and configures
  Cloudflare DNS and tunnel ingress.
- `review-cleanup.yml` requires the same `review_id` and infrastructure credentials. It
  removes the review namespace, DNS record, tunnel ingress, port mapping, and GitHub
  deployment environment.

The exact secret names are `KUBECONFIG_SERVER`, `KUBECONFIG_TOKEN`,
`CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_TUNNEL_ID`,
`DHI_REGISTRY_USERNAME`, and `DHI_REGISTRY_PASSWORD`.

## Releases

Consumers currently use the floating `@main` ref so the first version can be bootstrapped
from this empty repository. After the initial commit is published and verified, create a
protected `v1` release tag and update consumers to `@v1`. Pinning an immutable commit SHA
provides the strongest supply-chain guarantee; a maintained major tag provides controlled
fleet-wide upgrades.
