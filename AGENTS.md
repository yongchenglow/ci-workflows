# Repository guide

This repository provides reusable GitHub Actions workflows for projects using
Bun, Docker, GHCR, Helm, Kubernetes, and Cloudflare.

## Completion gates

Call work complete only after every applicable gate passes.

1. `actionlint -color .github/workflows/*.yml` exits successfully without warnings.
2. `npx --yes markdownlint-cli2@0.23.2 README.md AGENTS.md CLAUDE.md "docs/*.md"`
   exits successfully.
3. `git diff --check` exits successfully with no output.
4. Every changed workflow input, output, permission, secret, and default agrees
   with `docs/consumer-guide.md`.
5. Paired deploy and cleanup workflows derive identical resource names.
6. Documentation links resolve and Mermaid blocks parse.
7. The final diff contains only intended changes.

## Sources of truth

- Workflow behavior and caller interfaces live in `.github/workflows/*.yml`.
- Consumer configuration belongs in caller repositories.
- `docs/consumer-guide.md` explains the public interface.
- `docs/operations.md` explains deployment state and recovery.
- `docs/maintainer-guide.md` defines compatibility and release practices.

Read the consumer guide when changing `workflow_call`, permissions, secrets,
outputs, defaults, or caller examples. Read the operations guide when changing
Kubernetes resources, Cloudflare routes, review allocation, cleanup, or
rollback.

## Design rules

- Keep the workflow interface small. Derive safe values from
  `github.repository` and expose inputs only when callers genuinely vary.
- Keep project identity and deployment choices in callers. Do not add a
  consumer repository name, namespace, hostname, or production port here.
- Treat the four deployment workflows as two lifecycle pairs. A naming change
  in deploy must be mirrored in rollback or cleanup.
- Keep review allocation keys project-aware. Shared cluster state must not
  collide when repositories use the same pull request number.
- Preserve the Cloudflare catch-all rule as the final ingress entry.
- Use least-privilege job permissions. Pass sensitive values through secrets
  and shell environment variables.
- Preserve all published major-version contracts. Breaking inputs, outputs,
  defaults, resource names, or behavior require a new major version.

## Change workflow

1. Inspect the reusable workflow and every paired lifecycle workflow.
2. Update the implementation and its public documentation together.
3. Validate locally with actionlint.
4. If a caller contract changes, update a real caller as an integration example.
5. Review the diff for leaked credentials and unintended infrastructure changes.

Use Conventional Commits. Pull requests are the normal review path. Release
tags, pushes, deployments, and cluster mutations require explicit user
authorization.

## Documentation style

Write for maintainers and consumers who are new to the repository. Prefer
direct language, short sections, tables for exact contracts, and inline links
to authoritative sources. Use Mermaid only when relationships are clearer as a
diagram. Do not use em dashes. Use semicolons and colons sparingly.
