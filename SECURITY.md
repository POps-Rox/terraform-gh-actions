# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability in `terraform-gh-actions`, please report
it privately via GitHub Security Advisories on this repository, or email the
maintainers listed in `catalog-info.yaml`. Do not file public issues for
suspected vulnerabilities.

We aim to acknowledge reports within 2 business days and provide a remediation
timeline within 5 business days, depending on severity.

## Supply-Chain Integrity

The Docker image consumed by this action is pinned to a specific SHA256 digest
in `image/Dockerfile`, not a mutable tag like `:latest`. This prevents an
attacker who compromises the upstream registry from silently substituting a
malicious base image. Bumps to the digest are performed via Dependabot or by a
maintainer after manual review of the upstream change set.

## SSH Host Key Verification

The image ships with pre-populated `known_hosts` entries for `github.com`
generated at build time via `ssh-keyscan`. `StrictHostKeyChecking` is **not**
disabled; SSH-based Terraform module sources hosted on GitHub work out of the
box, while connections to unknown hosts will correctly fail closed.

If you need to reach a non-GitHub SSH host (for example, a self-hosted Git
server), inject your own `known_hosts` entries via a workflow step or a custom
image layer. Do not re-enable `StrictHostKeyChecking no`.

## `TERRAFORM_PRE_RUN` — Arbitrary Code Execution Escape Hatch

The `TERRAFORM_PRE_RUN` environment variable is read by `image/actions.sh` and
executed as a Bash script inside the action container, with full access to all
secrets and credentials mounted into the job (cloud provider credentials, the
`GITHUB_TOKEN`, Terraform Cloud tokens, etc.).

This is an intentional escape hatch — it allows callers to install extra tools,
configure private module registries, or perform other one-off setup — **but it
is also a remote code execution primitive** if the value can be influenced by
an untrusted party.

### Safe usage

- ✅ Set `TERRAFORM_PRE_RUN` as a literal string in your workflow YAML
  (committed to the default branch, protected by branch protection rules and
  required code review).
- ✅ Set `TERRAFORM_PRE_RUN` from a repository or organization secret managed by
  trusted administrators.
- ✅ Use it only in workflows triggered by `push`, `workflow_dispatch`,
  `schedule`, or `pull_request` events from collaborators with write access.

### Unsafe usage — do NOT do this

- 🚫 Do not derive `TERRAFORM_PRE_RUN` from `github.event.*` fields that an
  untrusted contributor can control (PR title, body, branch name, comment
  body, etc.).
- 🚫 Do not use `TERRAFORM_PRE_RUN` in workflows triggered by
  `pull_request_target` or `issue_comment` from forks without strict input
  validation.
- 🚫 Do not log or echo the value of `TERRAFORM_PRE_RUN` in a way that exposes
  secrets it may reference.

If you cannot meet these requirements, do not set `TERRAFORM_PRE_RUN`. Install
required tooling via a custom base image instead.

## Debug Output and Secret Hygiene

When debug mode is enabled (`ACTIONS_STEP_DEBUG=true`), `image/actions.sh`
dumps environment variables to the job log. Variables matching the following
patterns are filtered out before printing:

- `TF_VAR_*`
- `ARM_*`
- `AZURE_*`
- `GITHUB_TOKEN`
- Anything containing `SECRET`, `PASSWORD`, `KEY`, or `TOKEN`

This filter is best-effort. If you store secrets under variable names that do
not match these patterns, they may appear in debug logs. Prefer secret names
that explicitly include `SECRET`, `TOKEN`, or `KEY`.

## Reporting Scope

In-scope for this policy:

- The Docker image built from `image/Dockerfile`.
- The action entrypoints under `terraform-*/` and `image/entrypoints/`.
- The shell and Python helpers under `image/`.

Out of scope:

- Vulnerabilities in upstream Terraform, Terraform providers, or third-party
  tools installed in the base image — report those to the respective projects.
- Misconfiguration of caller workflows (see the `TERRAFORM_PRE_RUN` section
  above for the most common pitfall).
