# Enterprise DevSecOps CI/CT Pipeline

Production-grade CI/CT pipeline I designed and operate for a fleet of microservices running on OpenShift 4.x at enterprise scale. Built on GitLab CI with a self-hosted shell runner in an air-gapped environment. Covers the full software factory lifecycle from commit to lab promotion, with security gates at every stage.

This is not a tutorial. The architecture decisions, gating strategy, artifact flow, and GitOps variant documented here are all production-tested.

## What This Is

The pipeline follows a CI/CT model (Continuous Integration / Continuous Testing) rather than traditional CI/CD. Production deployments are intentionally outside the pipeline scope and go through a formal change management process. This is a deliberate design choice for platforms where availability and safety constraints make automated production deployments unacceptable.

Two pipeline variants are documented here:

**Standard pipeline** covers the full application lifecycle from build through lab promotion, with direct deployment to each environment via `oc rollout`.

**GitOps variant** replaces the deploy stage with a manifests repo update. ArgoCD watches the manifests repo and handles the actual deployment. This variant is covered separately under Part 2.

## Architecture

```
Developer push
     |
     v
.-------------.
|    LINT      |  yamllint · hadolint · helm lint · checkstyle
'------+-------'
       | pass
       v
.-------------.
|    BUILD     |  Maven compile → JAR artifact
'------+-------'
       | artifact: target/*.jar, target/classes/
       v
.-------------.
|  UNIT TEST   |  JUnit + JaCoCo coverage gate (>=80%)
'------+-------'
       | artifact: jacoco.xml, surefire-reports/
       v
.-------------.
|  SONARQUBE   |  SAST · Quality Gate: coverage >=90%, 0 critical bugs,
|  QUALITY     |  0 security hotspots, duplications <3%
|    GATE      |  Polls SonarQube API · hard block on fail
'------+-------'
       | pass
       v
.-------------.
|  ARTIFACTORY |  Versioned JAR stored with commit SHA
'------+-------'
       | pass
       v
.-------------.
|  DOCKER      |  UBI8 base image · multi-stage build
|   BUILD      |  Non-root user · minimal final image
'------+-------'
       | image in local daemon
       v
.-------------.
|   TRIVY      |  Air-gapped mode → internal Trivy server
|  CVE SCAN    |  CRITICAL: hard block (exit-code 1)
|              |  HIGH: report only
'------+-------'
       | clean image
       v
.-------------.
|  PUSH TO     |  Internal container registry
|   REGISTRY   |  Tagged with commit SHA
'------+-------'
       | pass
       v
.-------------.
|  DEPLOY DEV  |  Automatic rolling deploy → dev namespace
|  (auto)      |  oc rollout status --timeout=5m (readiness gate)
'------+-------'
       | healthy
       v
.-------------.
| INTEGRATION  |  API tests against live dev URL
|    TEST      |  Health check + full integration suite
'------+-------'
       | pass
       v
.-------------.
|  DEPLOY LAB  |  MANUAL GATE · dev lead approves in GitLab UI
|  (manual)    |  Feature testing · stress testing · QA cycles
'------+-------'
       |
       v
  PRODUCTION     Change ticket · dedicated deployment team
  (out of        Maintenance window · formal sign-off
   pipeline)     Not automated · by design
```

## Pipeline Stages

| Stage | Type | Gate Condition | Artifact Out |
|---|---|---|---|
| Lint | Auto | Any syntax/style error | none |
| Build | Auto | Compile failure | target/*.jar, target/classes/ |
| Unit Test | Auto | Test fail or coverage < 80% | jacoco.xml, surefire-reports/ |
| SonarQube | Auto | Quality gate fail (API poll) | none |
| Artifactory | Auto | Upload failure | Versioned JAR in Artifactory |
| Docker Build | Auto | Build error | Image in local daemon |
| Trivy Scan | Auto | CRITICAL CVE found | Trivy report (always saved) |
| Push to Registry | Auto | Push failure | Image in registry |
| Deploy DEV | Auto | Readiness probe timeout | none |
| Integration Test | Auto | API test failure | none |
| Deploy LAB | Manual | Human approval required | none |
| Production | Out of pipeline | Change management process | none |

## Key Design Decisions

**Why CI/CT and not CI/CD**

The platform runs availability-critical workloads. Automated production deployment is not acceptable as a bad deploy during business hours directly impacts live traffic. Production goes through a change management ticket, architecture and security review, dedicated deployment team sign-off, and an approved maintenance window. The pipeline owns everything up to lab. That is CI/CT by design, not by limitation.

**Why shell runner on a bastion host**

The environment is air-gapped. The bastion has controlled, audited access to both the internal tooling network and the OCP cluster via `oc` CLI. Shell executor means all tooling (Maven, Docker, Trivy client, SonarQube scanner, oc CLI) is installed and version-locked on the bastion. Simpler to manage and audit in an air-gapped setup than Docker-in-Docker or Kubernetes runners.

**Why UBI8 base image**

Red Hat Universal Base Image 8 has a significantly smaller CVE surface than general-purpose base images like ubuntu or debian. Red Hat backports security fixes, meaning Trivy finds fewer vulnerabilities on UBI8. For a pipeline with a CRITICAL CVE hard block, base image choice directly affects pipeline stability.

**Why internal Trivy server**

Air-gapped environment means no internet access from the runner. Trivy supports a client-server mode where an internal server hosts the synced CVE database. The server pulls updates on a controlled schedule and the pipeline runner connects to it as a client. This maintains CVE coverage without opening internet egress from the build network.

## Part 2 — GitOps Variant (ArgoCD + Helm)

For services with multiple interdependent Kubernetes resources, a GitOps approach was introduced as an alternative to direct pipeline deployment. The standard pipeline stages (lint through registry push) remain identical. Only the deploy stage changes.

Instead of `oc rollout`, the pipeline's last stage updates a single line in a separate manifests repo (the image tag) using `yq`. ArgoCD watches that repo, detects the change, renders the Helm chart with the updated values, and syncs to the cluster.

**Why a separate manifests repo**

App code changes and infrastructure/config changes have separate git histories. ArgoCD watches only the manifests repo with no noise from code commits. Rollback is a `git revert` on the manifests repo and ArgoCD auto-syncs back. The CI pipeline's only touch on the manifests repo is updating one line per run.

Full detail on the GitOps variant including the two-repo model, ArgoCD Application CRs, Helm chart structure, and rollback flow is in `docs/gitops-variant.md`.

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
| Deployment | OpenShift 4.x (oc rollout) |
| GitOps | ArgoCD + Helm |
| GitHub Actions variant | OIDC zero-credential auth, self-hosted runners |

## Repository Structure

```
.
├── README.md                        ← you are here
├── .gitlab-ci.yml                   ← full GitLab CI pipeline
├── .github/
│   └── workflows/
│       └── ci.yml                   ← GitHub Actions equivalent
├── docs/
│   ├── artifact-flow.md             ← how data moves between stages
│   ├── gating-strategy.md           ← every gate explained
│   └── gitops-variant.md            ← ArgoCD pipeline variant (Part 2)
├── helm/
│   └── sample-chart/                ← reference Helm chart structure
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-dev.yaml
│       └── values-lab.yaml
└── manifests/
    └── argocd-app.yaml              ← ArgoCD Application CR reference
```

## Related Projects

[aws-eks-platform](https://github.com/sourav-ndx/aws-eks-platform) — AWS EKS platform with IRSA/OIDC, ALB Ingress, CloudWatch, Terraform IaC

[k8s-kubeadm-gcp](https://github.com/sourav-ndx/k8s-kubeadm-gcp) — Kubernetes from scratch with kubeadm, Calico CNI, ArgoCD GitOps

## Author

Sourav Nandy — Platform Engineer | Solutions Architect  
[LinkedIn](https://linkedin.com/in/sourav-nandy-0115) · [GitHub](https://github.com/sourav-ndx)
