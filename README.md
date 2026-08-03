# WebGoat — with an Added DevSecOps CI/CD Pipeline

Fork of [OWASP WebGoat](https://github.com/WebGoat/WebGoat), a deliberately insecure Java/Spring Boot web application. A full DevSecOps CI/CD pipeline has been added on top of the upstream project: static analysis, dependency scanning, container image scanning, and a Quality Gate that blocks critical findings from reaching the container registry.

Part of a 3-repo DevSecOps project: [webgoat-infra](https://github.com/kenze-p/webgoat-infra) (Terraform infrastructure), **WebGoat** (this repo, application + security pipeline), [webgoat-helm](https://github.com/kenze-p/webgoat-helm) (GitOps deployment).

> The application itself is unmodified upstream WebGoat — see the [original README](https://github.com/WebGoat/WebGoat#readme) for what WebGoat is and how the lessons work. This document covers only what was added: [`.github/workflows/pipeline.yml`](.github/workflows/pipeline.yml) and [`.trivyignore`](.trivyignore).

## Pipeline Overview

```
push/PR ──► build-and-quality ──► ┬─► sast (Semgrep) ─────────┐
                                   └─► sca-dependencies (Trivy) ┤
                                                                ▼
                                                    build-and-push-image
                                                    (Docker build → Trivy
                                                     image scan → push to ECR)
```

- **build-and-quality** — Maven build, unit tests, Checkstyle (non-blocking — see Security Notes)
- **sast** — [Semgrep](https://semgrep.dev/), `p/owasp-top-ten` ruleset
- **sca-dependencies** — Trivy filesystem scan against `pom.xml`
- **build-and-push-image** — builds the Docker image, runs a Trivy image scan, and pushes to ECR only if both scans pass

## Quality Gate

Both scanning jobs run with `exit-code: 1` on any HIGH/CRITICAL finding, which fails the job and (via `needs:`) prevents the image-build job from running at all — a vulnerable image never reaches the registry.

## Accepted-Risk Findings (`.trivyignore`)

Two sets of findings are intentionally suppressed, both tied to WebGoat's own teaching material rather than to the pipeline's own code:

- **23 CVEs in `com.thoughtworks.xstream` 1.4.5** (1 CRITICAL — CVE-2013-7285, RCE via insecure deserialization), deliberately kept outdated as the subject of WebGoat's `VulnerableComponentsLesson`
- **2 CVEs in `org.bouncycastle:bcprov-jdk18on`** (CVE-2025-14813, CVE-2026-5598), a shaded/relocated transitive dependency pulled in by WebGoat's own JWT-related libraries; the exact source jar couldn't be pinpointed after shading stripped its Maven metadata, and upgrading the parent libraries was out of scope

Each entry in `.trivyignore` is commented with the reasoning above, rather than silently suppressed.

## Supply-Chain Incident

The pipeline originally pinned `aquasecurity/trivy-action@0.24.0`. That tag no longer resolves because it was among ~75 tags overwritten in a March 2026 supply-chain compromise of the action (malicious code exfiltrating CI secrets). Because the tag failed to resolve *before* any code executed, this pipeline was never actually exposed. Fixed by pinning to a specific, verified-safe commit SHA instead of a mutable tag:

```yaml
uses: aquasecurity/trivy-action@57a97c7e7821a5776cebc9bb87c984fa69cba8f1 # v0.35.0
```

## Security Notes

- Checkstyle is run with `continue-on-error: true` — the version of the Checkstyle plugin pinned in upstream WebGoat's `pom.xml` is incompatible with the ruleset it ships (a pre-existing upstream issue, also visible in WebGoat's own CI, which doesn't run Checkstyle at all). It still runs and reports, it just doesn't block the pipeline.
- Docker images are tagged with the Git commit SHA, never `latest`, matching the registry's `IMMUTABLE` tag policy.
- AWS credentials for the ECR push step are stored as GitHub Actions secrets, never committed.

## Project Status

The application is deployed via the companion [webgoat-helm](https://github.com/kenze-p/webgoat-helm) repo, which is destroyed between working sessions along with the rest of the infrastructure to avoid ongoing AWS cost.
