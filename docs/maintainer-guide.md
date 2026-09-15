# Maintainer guide

This repository is a shared delivery module. A small caller interface hides the
implementation for checks, image publication, security scanning, Kubernetes
deployment, and Cloudflare routing.

## Repository layout

| Path | Responsibility |
| --- | --- |
| `.github/workflows/reusable-*.yml` | Build, publication, and security interfaces. |
| `.github/workflows/production-*.yml` | Production deployment lifecycle. |
| `.github/workflows/review-*.yml` | Review deployment lifecycle and shared allocation. |
| `.github/workflows/lint.yml` | Workflow validation in GitHub Actions. |
| `.github/actionlint.yaml` | Local and CI actionlint configuration. |
| `.markdownlint-cli2.yaml` | Markdown lint rules and the one-line `CLAUDE.md` exception. |
| `renovate.json` | GitHub Actions dependency updates. |
| `docs/consumer-guide.md` | Public caller contract. |
| `docs/operations.md` | Runtime state and recovery procedures. |

Reusable workflows must stay directly under `.github/workflows`. GitHub does
not support subdirectories for reusable workflows. See
[Reuse workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows)
for platform constraints.

## Design principles

### Keep the interface deep

Add an input only when callers genuinely need different behavior. Prefer a
safe value derived from `github.repository` when every caller already has the
required information.

Project identity, public hostnames, and production NodePorts vary by caller.
Lock names, review port ranges, timeout behavior, and Cloudflare update logic
belong to the shared implementation.

### Keep lifecycle pairs symmetric

Treat these files as pairs.

| Create or update | Reverse or recover |
| --- | --- |
| `production-deploy.yml` | `production-rollback.yml` |
| `review-deploy.yml` | `review-cleanup.yml` |

A resource name derived during deployment must be derived from the same inputs
during cleanup. Verify namespaces, Helm releases, hostnames, GitHub
environments, ConfigMap keys, and concurrency groups after every naming change.

### Preserve shared state

Review workflows coordinate through two ConfigMaps in the `default`
namespace.

- `ci-workflows-review-ports` stores project-aware review ID to NodePort mappings.
- `ci-workflows-infra-lock` serializes changes to port mappings and
  Cloudflare tunnel ingress.

The lock is cluster-wide because these resources are shared across projects.
Production tunnel changes use the same lock as review lifecycle changes.
The stale-lock recovery path checks ownership before deletion. Keep that
compare-before-delete behavior.

### Keep caller data out of shell source

Pass input and secret values through a step `env` block. Quote shell
expansions. Validate names before using them as Kubernetes or Cloudflare
identifiers.

Use least-privilege job permissions. GitHub explains permission behavior for
called workflows in
[Reuse workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows#calling-a-reusable-workflow).

## Making a change

1. Read the workflow and its lifecycle pair.
2. Identify every caller-visible change.
3. Update the workflow interface and implementation.
4. Update the consumer guide in the same change.
5. Update the operations guide when runtime state or recovery changes.
6. Migrate a real caller when the public interface changes.
7. Run every validation gate.
8. Review the diff for unrelated edits and sensitive values.

Use Conventional Commits. Examples include `feat(ci): add registry input`,
`fix(review): isolate allocation keys`, and `docs: clarify rollback`.

## Validation

Run the repository checks from its root.

```sh
actionlint -color .github/workflows/*.yml
npx --yes markdownlint-cli2@0.23.2 README.md AGENTS.md CLAUDE.md "docs/*.md"
git diff --check
```

[actionlint](https://github.com/rhysd/actionlint) validates workflow syntax,
expression types, and embedded shell scripts. Its configuration enables
ShellCheck and does not define self-hosted runner labels.

When a deployment interface changes, also run actionlint in each migrated
caller. Lint every affected caller chart against its production and review
values.

```sh
helm lint helm/app -f helm/app/values-production.yaml
helm lint helm/app -f helm/app/values-review.yaml
```

Static checks cannot prove access to secrets, cluster permissions, free
NodePorts, Cloudflare permissions, or runtime connectivity. Verify those
conditions through a controlled caller after the new major tag exists.

## Compatibility

The caller interface includes more than input names.

- Input types, required status, defaults, and validation
- Workflow outputs
- Required secrets and permissions
- Expected chart paths and values
- Derived Kubernetes, Helm, Cloudflare, and GitHub environment names
- Failure thresholds and cleanup behavior

An additive optional input is normally compatible. Changing a default can be
breaking when it changes deployed state. Renaming a resource is breaking when
cleanup or rollback can no longer find the old resource.

The v1 tags preserve the original consumer-specific deployment behavior. The
generic interface begins at v2.

## Release process

1. Merge a validated Conventional Commit to `main`.
2. Confirm the target commit contains the expected workflow and documentation
   changes.
3. Create a new immutable semantic version tag such as `v2.0.0`.
4. Push the immutable tag.
5. Move the floating `v2` tag to the same verified commit.
6. Update controlled consumers from `@v1` to `@v2`.
7. Run one controlled caller through build, scan, review deploy, cleanup, and
   production deployment.

Do not move immutable version tags. A breaking contract change starts a new
major version. Compatible fixes increment the patch version. Compatible
features increment the minor version.

Tag creation, tag movement, pushes, and deployments are explicit release
actions. Do not perform them as part of an ordinary code change.

## Dependency updates

Renovate manages GitHub Actions versions each Monday before 09:00 in
Asia/Singapore. It groups most action updates and keeps Trivy updates separate.

Review dependency changes for release notes, input changes, runner
requirements, and permissions. Run actionlint after accepting an update.
GitHub's
[secure use reference](https://docs.github.com/en/actions/reference/security/secure-use)
recommends full commit SHA pinning when immutable dependency resolution is
required.

## Documentation maintenance

Keep exact contracts in tables and explanations near the first place a reader
needs them. Link to authoritative documentation instead of copying external
reference material.

Use Mermaid only for relationships or state transitions that are difficult to
explain linearly. Do not use em dashes. Use semicolons and colons sparingly.
