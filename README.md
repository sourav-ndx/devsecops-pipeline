# Enterprise DevSecOps CI/CT Pipeline

> Production-grade CI/CT pipeline designed and implemented for a SIP-based contact center platform at enterprise telecom scale. Built on GitLab CI with a self-hosted shell runner in an air-gapped environment. Covers the full software factory lifecycle — from commit to lab promotion — with security gates at every stage.

---

## What This Is

This repo documents the end-to-end CI/CT pipeline I designed and operate for a fleet of SIP call processing microservices running on OpenShift 4.x. The platform serves a major US telecom carrier. Given the nature of the workload — real-time voice traffic — the pipeline is designed around **CI/CT** (Continuous Integration / Continuous Testing), not traditional CI/CD. Production deployments are intentionally outside the pipeline scope and go through a formal change management process.

This is not a tutorial. It is a reference implementation — the architecture decisions, gating strategy, artifact flow, and GitOps variant are all production-tested.

---

## Architecture

```
Developer push
     │
     ▼
┌─────────────┐
│    LINT      │  yamllint · hadolint · helm lint · checkstyle
└──────┬──────┘
       │ pass
       ▼
┌─────────────┐
│    BUILD     │  Maven compile → JAR artifact
└──────┬──────┘
       │ artifact: target/*.jar, target/classes/
       ▼
┌─────────────┐
│  UNIT TEST   │  JUnit + JaCoCo coverage gate (≥80%)
└──────┬──────┘
       │ artifact: jacoco.xml, surefire-reports/
       ▼
┌─────────────┐
│  SONARQUBE   │  SAST · Quality Gate: coverage ≥90%, 0 critical bugs,
│  QUALITY     │  0 security hotspots, duplications <3%
│    GATE      │  Polls SonarQube API — hard block on fail
└──────┬──────┘
       │ pass
       ▼
┌─────────────┐
│  ARTIFACTORY │  Versioned JAR stored with commit SHA
└──────┬──────┘
       │ pass
       ▼
┌─────────────┐
│  DOCKER      │  UBI8 base image · multi-stage build
│   BUILD      │  Non-root user · minimal final image
└──────┬──────┘
       │ image in local daemon
       ▼
┌─────────────┐
│   TRIVY      │  Air-gapped mode → internal Trivy server
│  CVE SCAN    │  CRITICAL: hard block (--exit-code 1)
│              │  HIGH: report only
└──────┬──────┘
       │ clean image
       ▼
┌─────────────┐
│  PUSH TO     │  Quay internal registry
│   QUAY       │  Tagged with $CI_COMMIT_SHORT_SHA
└──────┬──────┘
       │ pass
       ▼
┌─────────────┐
│  DEPLOY DEV  │  Automatic rolling deploy → dev namespace
│  (auto)      │  oc rollout status --timeout=5m (readiness gate)
└──────┬──────┘
       │ healthy
       ▼
┌─────────────┐
│ INTEGRATION  │  API tests against live dev URL
│    TEST      │  Health check + full integration suite
└──────┬──────┘
       │ pass
       ▼
┌─────────────┐
│  DEPLOY LAB  │  MANUAL GATE — dev lead approves in GitLab UI
│  (manual)    │  Feature testing · stress testing · QA cycles
└──────┬──────┘
       │
       ▼
  PRODUCTION     Change ticket · onshore deployment team
  (out of         Maintenance window · formal sign-off
   pipeline)      Not automated — by design
```

---

## Pipeline Stages

| Stage | Type | Gate Condition | Artifact Out |
|---|---|---|---|
| Lint | Auto | Any syntax/style error | — |
| Build | Auto | Compile failure | target/*.jar, target/classes/ |
| Unit Test | Auto | Test fail or coverage < 80% | jacoco.xml, surefire-reports/ |
| SonarQube | Auto | Quality gate fail (API poll) | — |
| Artifactory | Auto | Upload failure | Versioned JAR in Artifactory |
| Docker Build | Auto | Build error | Image in local daemon |
| Trivy Scan | Auto | CRITICAL CVE found | Trivy report (always saved) |
| Push to Quay | Auto | Push failure | Image in Quay registry |
| Deploy DEV | Auto | Readiness probe timeout | — |
| Integration Test | Auto | API test failure | — |
| Deploy LAB | **Manual** | Human approval required | — |
| Production | **Out of pipeline** | Change management process | — |

---

## Key Design Decisions

### Why CI/CT, not CI/CD?

This platform processes live SIP voice calls for a carrier network. Automated production deployment of a platform handling real-time voice traffic is not acceptable — a bad deploy during business hours affects active calls. Production goes through:

- Change management ticket
- Architecture and security review
- Dedicated onshore deployment team
- Approved maintenance window

The pipeline owns everything up to lab. That is CI/CT by design, not by limitation.

### Why shell runner on a bastion host?

The environment is air-gapped. The bastion has controlled, audited access to both the internal tooling network and the OCP cluster via `oc` CLI. Shell executor means all tooling (Maven, Docker, Trivy client, SonarQube scanner, oc CLI) is installed and version-locked on the bastion. Simpler to manage and audit in an air-gapped setup than Docker-in-Docker or Kubernetes runners.

### Why UBI8 base image?

Red Hat Universal Base Image 8 has a significantly smaller CVE surface than general-purpose base images (ubuntu, debian). Red Hat backports security fixes, meaning Trivy finds fewer vulnerabilities on UBI8. For a pipeline with a CRITICAL CVE hard block, base image choice directly affects pipeline stability.

### Why internal Trivy server?

Air-gapped environment means no internet access from the runner. Trivy supports a client-server mode where an internal server hosts the synced CVE database. The server pulls updates on a controlled schedule; the pipeline runner connects to the internal server as a client. This maintains CVE coverage without opening internet egress from the build network.

### Why two-repo GitOps model for IPSG?

Separating application source from deployment manifests gives clean audit trails. App changes and infrastructure/config changes have separate git histories. ArgoCD watches only the manifests repo — no noise from code commits. Rollback is a `git revert` on the manifests repo; ArgoCD auto-syncs back. The CI pipeline's only touch on the manifests repo is updating one line (image tag) via `yq`.

---

## Tech Stack

| Layer | Technology |
|---|---|
| CI/CT Platform | GitLab CI/CD |
| Runner | Self-hosted shell executor on bastion |
| Build | Maven |
| Unit Testing | JUnit 5 + JaCoCo |
| SAST | SonarQube |
| Artifact Storage | Artifactory |
| Container Build | Docker (UBI8 base) |
| CVE Scanning | Trivy (air-gapped client-server mode) |
| Registry | Quay (internal) |
| Deployment | OpenShift 4.x (`oc rollout`) |
| GitOps | ArgoCD + Helm |
| GitHub Actions variant | OIDC zero-credential auth, self-hosted runners |

---

## Repository Structure

```
.
├── README.md                        ← you are here
├── .gitlab-ci.yml                   ← full GitLab CI pipeline
├── .github/
│   └── workflows/
│       └── ci.yml                   ← GitHub Actions equivalent
├── docs/
│   ├── pipeline-stages.md           ← stage-by-stage deep dive
│   ├── artifact-flow.md             ← how data moves between stages
│   ├── gating-strategy.md           ← every gate explained
│   ├── gitops-variant.md            ← ArgoCD/IPSG pipeline variant
│   ├── sonarqube-setup.md           ← SonarQube quality gate config
│   └── trivy-airgap.md              ← Trivy air-gapped server setup
├── helm/
│   └── sample-chart/                ← reference Helm chart structure
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-dev.yaml
│       └── values-lab.yaml
└── manifests/
    └── argocd-app.yaml              ← ArgoCD Application CR reference
```

---

## Related Projects

- [aws-eks-platform](https://github.com/sourav-ndx/aws-eks-platform) — AWS EKS platform with IRSA/OIDC, ALB Ingress, CloudWatch, Terraform IaC
- [k8s-kubeadm-gcp](https://github.com/sourav-ndx/k8s-kubeadm-gcp) — Kubernetes from scratch with kubeadm, Calico CNI, ArgoCD GitOps

---

## Author

**Sourav Nandy** — Platform Engineer | Solutions Architect  
[LinkedIn](https://linkedin.com/in/sourav-nandy-0115) · [GitHub](https://github.com/sourav-ndx)
