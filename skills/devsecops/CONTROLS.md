# DevSecOps Control Catalog

Each control lists what `present` requires, what to look for in source, and what usually lives outside the repository (`not_observable` until an owner confirms it). Assess every control for each artifact path from commit to production.

## 1. Source and review

**Protected default branch**
Present: merges to the default and release branches need a pull request, required status checks, and at least one review; force-push and deletion are blocked. Look for: `CODEOWNERS`, ruleset or branch-protection-as-code files. Outside source: the hosting platform's branch protection or rulesets.

**Code ownership for security-sensitive paths**
Present: `CODEOWNERS` assigns workflow files, build scripts, deployment manifests, and dependency manifests to an owning team, and code-owner review is required. Outside source: whether code-owner review is enforced.

**Signed or verified commits** (optional, by policy)
Present: required where the organization mandates it. Do not report its absence as a gap unless policy requires it.

## 2. Secrets

**Secret scanning**
Present: a scanner (for example gitleaks, trufflehog, or the platform's push protection) runs on every push and pull request and fails on new findings, plus a one-time history scan. Look for: the scanner step, its config, its allowlist and whether the allowlist is narrow.

**No secrets in source or images**
Present: no credentials in tracked files, build args, image layers, or CI logs; secrets come from a secret manager or CI secret store at runtime. Look for: `.env` files, `ARG`/`ENV` with credentials in Dockerfiles, `echo` of secrets in workflows.

**Rotation on exposure**
Present: a documented procedure to revoke and rotate a leaked credential. Outside source: whether it was exercised.

## 3. CI hardening

**Least-privilege tokens**
Present: workflow-level `permissions:` default to read (GitHub Actions: `permissions: contents: read` or `{}`), with write scopes granted per job only where needed. GitLab: protected variables only on protected branches. Jenkins: credentials bound per stage.

**Pinned third-party steps**
Present: third-party actions and orbs referenced by full commit SHA; base and tool images by digest; an update bot (Dependabot, Renovate) keeps pins current.

**Untrusted code isolation**
Present: workflows triggered by forks or pull requests (`pull_request`, merge-request pipelines) run without secrets or write tokens; privileged triggers (`pull_request_target`, `workflow_run`, comment-triggered jobs) never check out and execute pull-request code. Untrusted inputs (`github.event.*.title`, branch names, commit messages) are passed via environment variables, never interpolated into `run:` scripts.

**Cloud access by federation**
Present: CI reaches cloud accounts through OIDC federation with trust conditions bound to repository and branch or environment (`sub` claim), not long-lived keys. Outside source: the cloud-side trust policy.

**Protected deployment environments**
Present: production deploy jobs target an environment with required reviewers or a protected branch restriction, and only that environment holds production credentials. Outside source: environment protection settings.

**Runner isolation**
Present: self-hosted runners are ephemeral and not shared between untrusted pull requests and privileged jobs. Outside source: runner configuration.

## 4. Detection gates on change

**Dependency scanning (SCA)**
Present: lockfiles exist and are used (`npm ci`, `--frozen-lockfile`, Gradle dependency locking or verification); an SCA scanner (for example OSV-Scanner, Trivy, Grype, Snyk, OWASP Dependency-Check, Dependabot alerts) fails the pull request on new blocking findings by a written threshold.

**License policy**
Present: disallowed licenses are listed and checked, when the organization has a license policy.

**Static analysis (SAST)**
Present: a SAST tool suited to the languages in use (for example CodeQL, Semgrep, SpotBugs with FindSecBugs, gosec, Bandit) runs on pull requests, with results triaged and new high-confidence findings blocking.

**IaC and manifest policy**
Present: Terraform, CloudFormation, Helm and Kubernetes manifests are checked (for example Checkov, tfsec/Trivy config, KICS, Conftest/OPA, kube-linter) for public exposure, missing encryption, privileged pods, host mounts, and wildcard IAM, and failures block.

**Container image scanning**
Present: built images are scanned before push or promotion; base images are minimal (distroless, slim, or scratch), pinned by digest, and run as non-root with a read-only root filesystem where possible.

## 5. Artifact integrity

**Build once, promote by digest**
Present: one build produces the artifact; later stages and environments reference it by digest, never by rebuilding or by mutable tag.

**SBOM**
Present: an SBOM (SPDX or CycloneDX, for example via Syft) is generated per artifact and stored or attached as an attestation.

**Build provenance**
Present: SLSA provenance is generated by the build platform (not by a user-controlled step) and attached to the artifact. Record the SLSA build level the setup supports.

**Signing**
Present: artifacts and images are signed (for example Sigstore cosign keyless with the CI OIDC identity, or a KMS-held key), and the signing identity is restricted to the release workflow.

## 6. Deploy-time verification

**Signature and provenance verification**
Present: the deploy target verifies signature, signer identity, and provenance before running the artifact (for example Kyverno or Sigstore policy-controller admission policies, or a verify step in the deploy job that cannot be skipped). Outside source: whether the admission policy is enforcing rather than auditing.

**Deploy policy**
Present: admission or deploy-time policy rejects privileged workloads, unapproved registries, mutable tags, and missing resource limits.

**Drift detection**
Present: GitOps reconciliation or scheduled drift checks report out-of-band changes to infrastructure and workloads.

## 7. Runtime and response

**DAST**
Present: a dynamic scan (for example OWASP ZAP baseline) runs against an ephemeral or staging environment with dummy data, never against production or shared systems.

**Runtime detection and alerting**
Present: runtime security signals (for example Falco, cloud threat detection, WAF logs) route to an owned alert channel with a runbook.

**Vulnerability management**
Present: severity SLAs (for example critical 7 days, high 30 days), an exception register in the repository with owner and expiry, and a recurring report of open findings and mean time to remediate.

## Status rules

- `present`: the control exists in source, blocks on failure, and its scope covers every artifact path.
- `partial`: it exists but does not block, covers only some paths, or has broad allowlists or permanent exceptions.
- `missing`: no evidence in source and the control is source-controllable.
- `not_observable`: the decisive fact lives outside the repository; name the fact and who can confirm it.
