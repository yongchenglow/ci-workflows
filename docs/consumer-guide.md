# Consumer guide

This guide defines the caller contract for v2. The workflow YAML remains the
source of truth. When this guide and a workflow disagree, treat the mismatch as
a defect and update both in the same change.

GitHub explains the underlying call model in
[Reuse workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows).
A caller invokes a reusable workflow at the job level with `uses`. Inputs are
passed with `with`. Secrets may be passed by name or with `secrets: inherit`.

## Supported project profile

The library is generic within this standard stack.

- Bun with a committed `bun.lock`
- A Dockerfile that accepts the build arguments described below
- GHCR or another compatible container registry
- A Helm chart with the expected values
- A Kubernetes cluster reachable with a server URL and bearer token
- A remotely managed Cloudflare Tunnel and a Cloudflare DNS zone

The deployment workflows expect the chart to accept
`image.repository`, `image.tag`, and `service.nodePort`. Deployments must
carry the standard `app.kubernetes.io/instance` label for rollout
verification. Helm documents the command behavior in
[`helm upgrade`](https://helm.sh/docs/helm/helm_upgrade/).

## Caller responsibilities

The caller owns event triggers, job ordering, environments, repository
variables, secrets, and project-specific deployment values. The called
workflow owns the shared implementation.

Calling jobs must grant permissions that are at least as broad as the called
job. Permissions can be maintained or reduced through a reusable workflow call.
They cannot be elevated by the called workflow.

### Secrets

| Secret | Used by | Purpose |
| --- | --- | --- |
| `DHI_REGISTRY_USERNAME` | Docker build | Authenticate to Docker Hardened Images when `dhi_login` is enabled. |
| `DHI_REGISTRY_PASSWORD` | Docker build | Authenticate to Docker Hardened Images when `dhi_login` is enabled. |
| `KUBECONFIG_SERVER` | Deploy, rollback, cleanup | Kubernetes API server URL. |
| `KUBECONFIG_TOKEN` | Deploy, rollback, cleanup | Kubernetes bearer token. Needs `list` on services in all namespaces so review deploys can skip NodePorts the cluster already holds. |
| `CLOUDFLARE_API_TOKEN` | Deploy and cleanup | Update tunnel configuration and DNS records. |
| `CLOUDFLARE_ACCOUNT_ID` | Deploy and cleanup | Select the Cloudflare account. |
| `CLOUDFLARE_TUNNEL_ID` | Deploy and cleanup | Select the remotely managed tunnel. |

`GITHUB_TOKEN` is provided by GitHub. Pass repository secrets with
`secrets: inherit` when calling workflows that require them. Cloudflare
recommends scoped API tokens for API access. Its
[DNS record API](https://developers.cloudflare.com/api/resources/dns/subresources/records/methods/create/)
documents the required DNS write permission.

### Variables

Version 2 does not read repository variables directly. Callers map their own
variables into workflow inputs. A repository may keep
`CLOUDFLARE_DOMAIN=example.com` and pass it as `cloudflare_zone` and
`production_hostname`. Use separate values when production is hosted on a
subdomain or review apps use a dedicated domain.

### Common permissions

| Workflow | Required caller permissions |
| --- | --- |
| Build | `contents: read`, `security-events: write` |
| Secret scan | `contents: read`, `security-events: write` |
| Helm lint | `contents: read` |
| Docker build | `contents: read`, `packages: write` |
| Image scan | `contents: read`, `packages: read`, `security-events: write` |
| Production deploy | `contents: read`, `packages: read`, `deployments: write` |
| Production rollback | `contents: read`, `deployments: read` |
| Review deploy | `contents: read`, `packages: read`, `deployments: write` |
| Review cleanup | `contents: read`, `deployments: write` |

## Composite actions

### `bun-version`

Reusable workflows take `bun_version` as an input, and `with:` cannot run a
script. This action reads the value from the caller's `package.json`
`packageManager` field so each caller does not repeat the lookup.

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `working-directory` | No | `.` | Directory containing `package.json`. |

| Output | Meaning |
| --- | --- |
| `bun_version` | Version from `packageManager`, without the name prefix. |

The action reads a file, so the caller checks out first. A sparse checkout of
`package.json` is enough when the job needs nothing else.

```yaml
bun-version:
  runs-on: ubuntu-latest
  permissions:
    contents: read
  outputs:
    bun_version: ${{ steps.bun.outputs.bun_version }}
  steps:
    - uses: actions/checkout@v7
      with:
        sparse-checkout: package.json
        sparse-checkout-cone-mode: false
    - id: bun
      uses: yongchenglow/ci-workflows/.github/actions/bun-version@v2
```

## Build and scanning workflows

### `reusable-build.yml`

The workflow installs dependencies with `bun install --frozen-lockfile` and
runs four matrix legs. The required package scripts are `lint`, `typecheck`,
`test`, and `knip`. Bun describes reproducible installation in its
[`bun install` documentation](https://bun.sh/docs/pm/cli/install).

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `bun_version` | No | Latest resolved by setup-bun | Bun version used for checks. |

The workflow also scans the filesystem with Trivy. Critical vulnerabilities and
misconfigurations fail the job. Findings are uploaded to GitHub code scanning.

### `reusable-secret-scan.yml`

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `fetch_depth` | No | `0` | Git history depth. Zero scans full history. |

Gitleaks scans the checked-out Git history. A shallow value speeds up scanning
but can miss secrets in older commits.

### `helm-lint.yml`

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `chart_path` | Yes | None | Chart directory in the caller repository. |
| `profiles` | Yes | None | Newline separated `release\|values_file\|namespace` entries. |
| `strict` | No | `true` | Treat chart warnings as failures. |
| `helm_version` | No | `latest` | Version passed to `azure/setup-helm`, which requires it. |
| `image_tag` | No | Placeholder branch and SHA tag | Tag substituted while rendering. |
| `image_repository` | No | `ghcr.io/example/app` | Repository substituted while rendering. |

Each profile is linted and rendered independently, so a values file that is
valid for one environment cannot mask a failure in another. The job collects
every failure before exiting rather than stopping at the first.

`helm lint` parses templates but does not prove they render. The workflow also
runs `helm template`, which catches errors that appear only once values are
substituted, such as a missing required value or an invalid resource field.

A chart may validate the image tag format. Committed values files often carry a
placeholder that such a chart rejects, because the deploy workflows always
supply a real tag. `image_tag` and `image_repository` reproduce that
substitution so rendering matches deployment. Override them when a chart
enforces a specific format.

```yaml
helm-validate:
  permissions:
    contents: read
  uses: yongchenglow/ci-workflows/.github/workflows/helm-lint.yml@v2
  with:
    chart_path: helm/app
    profiles: |
      web|helm/app/values-production.yaml|personal-site
      web|helm/app/values-review.yaml|personal-site-review
```

### `reusable-docker.yml`

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `bun_version` | No | Empty | Bun version exposed as a Docker build argument. |
| `dhi_login` | No | `false` | Authenticate to Docker Hardened Images before building. |
| `registry` | No | `ghcr.io` | Destination container registry. |
| `image_name` | No | Caller `owner/repository` | Image path without the registry. |
| `platforms` | No | `linux/amd64` | Docker Buildx target platforms. |
| `tag_prefix` | No | Current branch | Prefix for the generated immutable tag. |
| `push` | No | `true` | Whether to publish the image. |

| Output | Meaning |
| --- | --- |
| `image_tag` | Generated tag ending in the caller commit SHA. |
| `image_name` | Lowercase image path without the registry. |

The caller Dockerfile must accept `BUN_VERSION`, `DHI_BUN_TAG`,
`BUILD_DATE`, `REVISION`, and `VERSION` if it uses those values.
`package.json` supplies the package version.

#### Building before the gates finish

`push` lets a caller overlap the image build with its quality gates instead of
queuing behind them. Call the workflow twice: once with `push: false` alongside
the gates, then once with `push: true` after they pass.

The layer cache makes the second call cheap. `cache-to` runs whether or not the
image is published, and both calls read the same branch scope, so the publishing
call restores every layer the first one built.

```yaml
docker:
  needs: bun-version
  uses: yongchenglow/ci-workflows/.github/workflows/reusable-docker.yml@v2
  with:
    push: false

docker-push:
  needs: [docker, build, secret-scan, helm-validate]
  uses: yongchenglow/ci-workflows/.github/workflows/reusable-docker.yml@v2
  with:
    push: true
```

Both calls must receive identical inputs. Differing build arguments produce a
cache miss, and differing tag inputs publish a tag the gates never covered.

### `reusable-security-scan.yml`

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `image_tag` | Yes | None | Published image tag to scan. |
| `registry` | No | `ghcr.io` | Source container registry. |
| `image_name` | No | Caller `owner/repository` | Image path without the registry. |

Trivy reports critical and high vulnerabilities to code scanning. Critical
findings fail the job. High findings remain visible without blocking delivery.

## Production workflows

### `production-deploy.yml`

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `image_tag` | Yes | None | Image tag to deploy. |
| `application_name` | No | Caller repository name | Lowercase DNS-safe project identity. Maximum 40 characters. |
| `namespace` | No | `application_name` | Production Kubernetes namespace. |
| `registry` | No | `ghcr.io` | Source container registry. |
| `image_name` | No | Caller `owner/repository` | Image path without the registry. |
| `release_name` | No | `app` | Helm release name. |
| `chart_path` | No | `./helm/app` | Chart path in the caller repository. |
| `values_file` | No | `./helm/app/values-production.yaml` | Production values file. |
| `production_hostname` | Yes | None | Fully qualified public hostname. |
| `cloudflare_zone` | Yes | None | Cloudflare zone that contains the hostname. |
| `production_node_port` | Yes | None | Dedicated port from `30000` through `32767`. |

| Output | Meaning |
| --- | --- |
| `deployment_url` | HTTPS URL built from `production_hostname`. |

The caller must reserve a unique production NodePort. Kubernetes documents the
default range and collision behavior in
[Service type NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport).

### `production-rollback.yml`

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `application_name` | No | Caller repository name | Used to derive the namespace. |
| `namespace` | No | `application_name` | Production Kubernetes namespace. |
| `release_name` | No | `app` | Helm release to roll back. |
| `production_hostname` | Yes | None | URL attached to the GitHub environment. |

Rollback requires a previous Helm revision and a latest successful GitHub
deployment in the `production` environment.

## Review workflows

Review deployment and cleanup must receive the same
`application_name`, `release_name`, `review_domain`, and
`cloudflare_zone`. Mismatched values leave resources behind.

### `review-deploy.yml`

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `image_tag` | Yes | None | Image tag to deploy. |
| `review_id` | Yes | None | Identifier matching `pr-[0-9]+`. |
| `application_name` | No | Caller repository name | Lowercase DNS-safe project identity. Maximum 40 characters. |
| `production_namespace` | No | `application_name` | Namespace containing the image pull secret. |
| `registry` | No | `ghcr.io` | Source container registry. |
| `image_name` | No | Caller `owner/repository` | Image path without the registry. |
| `release_name` | No | `app` | Helm release name. |
| `chart_path` | No | `./helm/app` | Chart path in the caller repository. |
| `values_file` | No | `./helm/app/values-review.yaml` | Review values file. |
| `review_domain` | Yes | None | Domain appended to the review ID. |
| `cloudflare_zone` | Yes | None | Cloudflare zone containing the review domain. |
| `registry_secret_name` | No | `ghcr-secret` | Pull secret copied into the review namespace. |

For `application_name: storefront`, `review_id: pr-42`, and
`review_domain: review.example.com`, the workflow creates namespace
`storefront-pr-42` and hostname `pr-42.review.example.com`.

### `review-cleanup.yml`

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `review_id` | Yes | None | Identifier matching `pr-[0-9]+`. |
| `application_name` | No | Caller repository name | Project identity used during deployment. |
| `release_name` | No | `app` | Helm release used during deployment. |
| `review_domain` | Yes | None | Review domain used during deployment. |
| `cloudflare_zone` | Yes | None | Cloudflare zone used during deployment. |

See the [operations guide](operations.md) before manually changing review
namespaces, allocation records, or tunnel ingress.

## Complete caller examples

### Production delivery

```yaml
jobs:
  docker:
    permissions:
      contents: read
      packages: write
    uses: yongchenglow/ci-workflows/.github/workflows/reusable-docker.yml@v2
    with:
      bun_version: 1.4.2
    secrets: inherit

  scan:
    needs: docker
    permissions:
      contents: read
      packages: read
      security-events: write
    uses: yongchenglow/ci-workflows/.github/workflows/reusable-security-scan.yml@v2
    with:
      image_tag: ${{ needs.docker.outputs.image_tag }}
    secrets: inherit

  deploy:
    needs: [docker, scan]
    permissions:
      contents: read
      packages: read
      deployments: write
    uses: yongchenglow/ci-workflows/.github/workflows/production-deploy.yml@v2
    with:
      image_tag: ${{ needs.docker.outputs.image_tag }}
      application_name: storefront
      production_hostname: storefront.example.com
      cloudflare_zone: example.com
      production_node_port: 30001
    secrets: inherit
```

### Review lifecycle

```yaml
jobs:
  deploy:
    if: github.event.action != 'closed'
    permissions:
      contents: read
      packages: read
      deployments: write
    uses: yongchenglow/ci-workflows/.github/workflows/review-deploy.yml@v2
    with:
      image_tag: ${{ needs.docker.outputs.image_tag }}
      review_id: pr-${{ github.event.number }}
      application_name: storefront
      review_domain: review.example.com
      cloudflare_zone: example.com
    secrets: inherit

  cleanup:
    if: github.event.action == 'closed'
    permissions:
      contents: read
      deployments: write
    uses: yongchenglow/ci-workflows/.github/workflows/review-cleanup.yml@v2
    with:
      review_id: pr-${{ github.event.number }}
      application_name: storefront
      review_domain: review.example.com
      cloudflare_zone: example.com
    secrets: inherit
```

Fork pull requests must not receive deployment credentials. Keep build and
secret checks available to forks, then guard image publication and deployment
with `github.event.pull_request.head.repo.fork == false`.

## Choosing a version

Use an immutable semantic version tag when reproducibility matters most. Use
the floating major tag when controlled compatible upgrades are preferred.

```yaml
uses: yongchenglow/ci-workflows/.github/workflows/reusable-build.yml@v2.0.0
```

GitHub recommends full commit SHA pinning for the strongest supply chain
protection in its
[secure use reference](https://docs.github.com/en/actions/reference/security/secure-use).
