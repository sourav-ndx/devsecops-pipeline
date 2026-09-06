# Gating Strategy

Every stage in this pipeline gates the next via job exit codes. A non-zero exit code fails the job and stops the pipeline. Nothing moves forward without passing every gate before it.

---

## Gate 1 — Lint (syntax/style)

**What it checks:** YAML syntax, Dockerfile best practices, Helm chart validity, Java code style.

**How it gates:** Tools like `hadolint` and `yamllint` return exit code 1 on any violation. Maven checkstyle:check fails the build if style rules are violated. Any non-zero exit = pipeline stops immediately.

**Why first:** Lint is the cheapest check — runs in seconds. Catching a Dockerfile error here saves a 10-minute Maven build from running pointlessly.

---

## Gate 2 — Build (compile)

**What it checks:** Source code compiles cleanly with no errors.

**How it gates:** Maven exits non-zero on compile failure. Pipeline stops — nothing to test if build fails.

**Artifact produced:** `target/*.jar` and `target/classes/` — referenced by downstream stages via `dependencies:` block.

---

## Gate 3 — Unit Test (correctness + coverage)

**Two sub-gates:**

**3a. JUnit:** Any test failure = non-zero exit = pipeline stops. Configured via `mvn test` — Maven returns non-zero if any test fails.

**3b. Coverage:** Python script parses `jacoco.xml`, computes line coverage percentage, explicitly calls `exit 1` if below 80%. This is a pipeline-level gate in addition to the SonarQube quality gate.

**Why two coverage gates:** The pipeline coverage check (80%) is a quick sanity check. SonarQube (90%) is the authoritative quality gate. Belt and suspenders.

**Artifact produced:** `jacoco.xml` — consumed by SonarQube stage for coverage data.

---

## Gate 4 — SonarQube Quality Gate

**What it checks:** Static analysis (bugs, code smells, security hotspots, duplications) + coverage — all evaluated server-side against the Quality Gate configuration.

**Quality Gate thresholds (configured on SonarQube server, not in YAML):**
- Line coverage ≥ 90%
- Critical bugs = 0
- Blocker issues = 0
- Security hotspots unreviewed = 0
- Code duplication < 3%

**How it gates — two-step:**

1. `sonar-scanner` uploads analysis to SonarQube server (async — server processes after scanner returns)
2. Pipeline polls SonarQube API: `GET /api/ce/task?id=<taskId>` — waits for task status = SUCCESS
3. Then checks: `GET /api/qualitygates/project_status?projectKey=<key>` — if status ≠ OK, explicitly `exit 1`

**Why API polling instead of relying on scanner exit code:** The scanner submits analysis and returns before the server finishes evaluating. Relying on scanner exit code alone can miss quality gate failures that the server catches asynchronously. Explicit API polling is the reliable method.

---

## Gate 5 — Trivy CVE Scan

**What it checks:** All packages in the container image against CVE databases.

**How it gates:**
```bash
trivy image --exit-code 1 --severity CRITICAL ...
```
`--exit-code 1` makes Trivy return non-zero when any CRITICAL CVE is found. GitLab/GitHub Actions treat non-zero as job failure = pipeline stops = image never reaches registry.

HIGH severity uses `--exit-code 0` — reported in artifact but does not block. Reviewed separately by the security team.

**Air-gapped mode:** `--server $TRIVY_SERVER` connects to an internal Trivy server instead of downloading the CVE database from the internet. The internal server syncs from upstream on a controlled daily schedule.

**Report always saved:** `when: always` in GitLab artifacts config ensures the Trivy report is saved even when the scan fails. Required for audit trail — security team needs to see what was found even if the pipeline blocked.

---

## Gate 6 — OCP Readiness Probes (Deploy DEV)

**What it checks:** New pods pass their readiness probe before rollout completes.

**How it gates:**
```bash
oc rollout status deployment/myapp --timeout=5m
```
This command blocks until all pods in the new ReplicaSet are Ready, or until timeout. If pods fail readiness probes (health check endpoint returns non-200, or pod crashes), the rollout stalls, the timeout fires, the command exits non-zero, the job fails.

**Why this matters:** A bad image that passes all static gates but crashes at runtime gets caught here. The rollout gate is the runtime safety net.

---

## Gate 7 — Integration Tests (against live DEV)

**What it checks:** Application works end-to-end in a real environment — API contracts, service dependencies, configuration correctness.

**How it gates:** Test runner (Maven with integration profile / Newman / pytest) returns non-zero on failure. Pipeline stops — lab deploy blocked if integration tests fail.

**Why after deploy:** Integration tests need a running application. They cannot run against a JAR or an image — they need the full stack running in the dev namespace with its dependencies.

---

## Gate 8 — Manual Approval (Deploy LAB)

**How it gates:** `when: manual` in GitLab CI rules block. The job is created but paused — it sits in "manual" state in the pipeline UI. A designated person (dev lead) must click the play button to trigger it.

**Why manual here:** Lab is where QA begins — feature testing, stress testing, regression. Promoting to lab is a deliberate decision, not an automatic consequence of passing tests. It also signals to QA that a new build is ready for their test cycles.

---

## Production — Out of Pipeline

Production deployment is intentionally not part of this pipeline.

**Why:**
- Platform carries live SIP voice traffic for a carrier network
- Automated deployment to production during business hours is unacceptable risk
- Regulatory and operational requirements mandate change management
- Deployment requires a formal change ticket, architecture/security review, approval, and a scheduled maintenance window
- Dedicated onshore deployment team owns production — separate from the platform engineering team that owns this pipeline

**This is the correct model for safety/availability-critical systems.** It mirrors what Bosch does for automotive software (CI/CT not CI/CD) and what any telecoms carrier platform requires.
