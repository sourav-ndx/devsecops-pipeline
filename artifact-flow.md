# Artifact Flow Between Stages

Data produced in one pipeline stage is consumed by downstream stages. This document maps every artifact — what produces it, what consumes it, and how GitLab transfers it.

---

## How GitLab Artifact Transfer Works

```yaml
# Producer stage declares what it outputs
build:
  artifacts:
    paths:
      - target/*.jar
      - target/classes/

# Consumer stage declares what it needs
sonarqube:
  dependencies:
    - build      # GitLab downloads build's artifacts before this job starts
    - unit-test  # downloads jacoco.xml too
```

When a job lists `dependencies`, GitLab automatically downloads those artifacts to the runner workspace before the job script starts. No manual copy commands needed — the files just appear.

`dependencies: []` explicitly means "download nothing" — used for jobs that only need a live URL, not files.

---

## Artifact Map

```
┌─────────────────────────────────────────────────────────────┐
│ STAGE 2 — BUILD                                             │
│                                                             │
│ Produces:                                                   │
│   target/*.jar          ──► consumed by: containerize       │
│   target/classes/       ──► consumed by: sonarqube          │
│                                                             │
│ expire_in: 1 hour                                           │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ STAGE 3 — UNIT TEST                                         │
│                                                             │
│ Consumes:                                                   │
│   target/classes/       ◄── from build (mvn test needs it) │
│                                                             │
│ Produces:                                                   │
│   target/site/jacoco/jacoco.xml  ──► consumed by: sonarqube│
│   target/surefire-reports/       ──► consumed by: sonarqube│
│                                       + GitLab UI (test tab)│
│                                                             │
│ Special: reports.junit path tells GitLab to parse JUnit XML │
│          and display pass/fail counts in the pipeline UI    │
│                                                             │
│ expire_in: 1 hour                                           │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ STAGE 4 — SONARQUBE                                         │
│                                                             │
│ Consumes:                                                   │
│   target/classes/               ◄── from build             │
│   target/site/jacoco/jacoco.xml ◄── from unit-test         │
│   target/surefire-reports/      ◄── from unit-test         │
│                                                             │
│ Produces: nothing to downstream stages                      │
│   (result communicated via SonarQube server API)           │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ STAGE 6 — CONTAINERIZE                                      │
│                                                             │
│ Consumes:                                                   │
│   target/*.jar  ◄── from build (Dockerfile COPY uses this) │
│                                                             │
│ Produces:                                                   │
│   Docker image in local daemon on runner                    │
│   (NOT a GitLab artifact — lives in Docker daemon)         │
│   Referenced by name: $IMAGE_NAME:$IMAGE_TAG               │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ STAGE 7 — TRIVY SCAN                                        │
│                                                             │
│ Consumes:                                                   │
│   Docker image by name from local daemon                    │
│   (no dependencies: block — image is in daemon not files)  │
│                                                             │
│ Produces:                                                   │
│   trivy-report-critical.txt  ─► GitLab artifact (always)  │
│   trivy-report-high.txt      ─► GitLab artifact (always)  │
│                                                             │
│ when: always — report saved even when scan fails           │
│ expire_in: 7 days — longer retention for security audit    │
└─────────────────────────────────────────────────────────────┘
```

---

## Why Docker Image Is Not a GitLab Artifact

GitLab artifacts are files in the runner workspace. A Docker image lives in the Docker daemon — it's not a file you can path-reference in `artifacts.paths`.

Because stages 6, 7, and 8 (containerize, trivy, push) all run on the same self-hosted shell runner on the bastion, they all share the same Docker daemon. The image built in stage 6 is accessible by name in stages 7 and 8 without any transfer — it's already there.

This is a property of the **shell executor on a persistent runner**. With ephemeral runners (Kubernetes runner, GitHub hosted runners), you would need to either:
- Save the image to a tar file and upload as artifact, or
- Use a shared registry as intermediate store

---

## Commit SHA as Universal Version Key

`$CI_COMMIT_SHORT_SHA` (GitLab) / `${{ github.sha }}` (GitHub Actions) threads through every stage as the version key:

```
Build artifact:   myapp-abc1234.jar          (Artifactory)
Docker image tag: quay.example.com/.../myapp:abc1234
Trivy report:     trivy-reports-abc1234
Deployed image:   $IMAGE_NAME:abc1234        (in OCP deployment)
```

Given any deployment in OCP, you can trace back to the exact commit, the exact Trivy report, the exact SonarQube analysis, and the exact JAR in Artifactory. Full end-to-end traceability.
