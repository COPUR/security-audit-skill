---
name: devsecops
description: DevSecOps guidance for designing, reviewing, and fixing the security controls of a software delivery pipeline - secret scanning, SAST, dependency and container scanning, IaC policy, SBOM, provenance and signing, deploy-time admission, CI hardening, and vulnerability management. Use when adding or reviewing security gates in CI/CD, hardening GitHub Actions, GitLab CI or Jenkins, planning a DevSecOps rollout, or turning audit findings into pipeline controls. For finding vulnerabilities in source code, use the security-audit skill instead.
---

# DevSecOps

Make the delivery pipeline prove, on every change, that what ships is what was reviewed, built from known inputs, free of known blocking issues, and deployed only by the identity allowed to deploy it. This skill builds and reviews those controls. It does not hunt for vulnerabilities in application source; that is the [security-audit](../security-audit/SKILL.md) skill, whose [SUPPLY-CHAIN-AND-RELEASE.md](../security-audit/SUPPLY-CHAIN-AND-RELEASE.md) and [CLOUD-AND-DEPLOYMENT.md](../security-audit/CLOUD-AND-DEPLOYMENT.md) companions hunt for exploitable flaws in the pipeline and runtime this skill hardens.

## Operating modes

- **Review mode** (default): read the repository's pipeline, build, and deployment configuration and report each control from [CONTROLS.md](CONTROLS.md) as `present`, `partial`, `missing`, or `not_observable`, with the file and line that shows it. Change nothing.
- **Implement mode**: when the user asks to add or fix controls, change pipeline files in the smallest steps that each leave the pipeline green, in the rollout order below. Never weaken an existing gate to make a new one pass.

If the request could mean either, review first and propose the changes.

## Core principles

1. **Evidence over presence.** A scanner step that runs with `continue-on-error`, `|| true`, `allow_failure`, a non-blocking exit code, or results nobody reads is `partial`, not `present`. A control is present only when a failing result stops the merge, the release, or the deploy.
2. **The pipeline is authorization code.** Every workflow states which event runs it, whose code it runs, which secrets and token scopes it holds, and what it may publish. Least privilege applies to CI identities exactly as to production ones.
3. **Short-lived identity, no stored cloud keys.** Prefer workload identity federation (OIDC from the CI provider to the cloud, bound to repository, branch or environment claims) over long-lived access keys in CI secrets.
4. **Pin what you execute.** Third-party actions, orbs, base images, and build tools are pinned to an immutable digest or commit SHA, and updated by a bot that opens reviewable changes.
5. **Build once, promote the same artifact.** The artifact deployed to production is byte-identical to the one tested, identified by digest, with provenance and a signature verified at deploy time.
6. **Gate on risk, not on count.** Blocking thresholds name a severity and reachability rule (for example: block on critical or high with a fix available in a shipped dependency). Exceptions are time-boxed, owned, and recorded in the repository.
7. **Respect what the repository cannot show.** Branch protection, required checks, environment approvals, registry policy, and cluster admission settings often live outside source. Report them `not_observable` with the exact fact an owner must confirm; do not assume them present or absent.

## Workflow

1. **Inventory.** List CI systems and workflow files, triggers, runners, secrets and token permissions, build tools, artifact types (packages, images, charts, IaC), registries, and deploy targets. Note which files the pipeline trusts from forks or pull requests.
2. **Assess.** Walk every control in [CONTROLS.md](CONTROLS.md) for each artifact path from commit to production. Record status, evidence (`file:line` or the missing fact), and the risk if absent.
3. **Prioritize.** Order gaps by what an attacker or an accident can reach: CI hardening and secret exposure first, then integrity of what ships (pinning, provenance, signing, admission), then detection gates (SAST, SCA, container and IaC scanning), then runtime and response.
4. **Implement** (implement mode only). Follow the rollout order: add each scanner in report-only mode, fix or triage the existing backlog, then make it blocking with a written threshold. Add a regression check where possible (a fixture that the gate must reject).
5. **Report.** Produce the control table, the prioritized gap list with the smallest fix for each, the items that need an owner to confirm, and what was changed with how it was verified.

## Rollout order for a new pipeline

1. CI hardening: least-privilege tokens, pinned actions, no untrusted code in privileged triggers, protected environments.
2. Secret scanning on push and in history, with rotation for anything found.
3. Dependency (SCA) and license scanning on pull requests.
4. SAST on pull requests.
5. IaC and Kubernetes manifest policy checks.
6. Container image scanning and minimal, pinned base images.
7. SBOM, build provenance, and artifact signing at build.
8. Signature and policy verification at deploy (admission control).
9. DAST against an ephemeral environment; runtime detection and alert routing.
10. Vulnerability management: SLAs by severity, exception register, metrics.

## Output format

```
| Stage | Control | Status | Evidence | Gap and smallest fix |
|-------|---------|--------|----------|----------------------|
```

Follow the table with: the top five gaps in priority order, the `not_observable` facts with who can confirm them, and, in implement mode, each change with the command or pipeline run that proved it.

## Safety

Do not run scanners or builds that execute untrusted repository code outside a sandbox, push to protected branches, change branch protection, rotate or print secrets, or trigger deployments. Propose those steps for the owner. Never paste a discovered secret into a report; cite its location and type only.

## Anti-patterns

1. Counting scanner steps instead of checking that they block.
2. Adding every scanner as blocking on day one, which teaches teams to bypass the pipeline.
3. Long-lived cloud keys in CI secrets when federation is available.
4. `pull_request_target` or equivalent privileged triggers that check out and run pull-request code.
5. Signing artifacts without verifying signatures at deploy.
6. Rebuilding for each environment instead of promoting one digest.
7. Permanent, unowned scanner exceptions.
8. Reporting a missing control as a vulnerability. A missing gate is a gap; only an exploitable path is a finding for security-audit.
