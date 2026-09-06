# Interview Recall Notes
> Personal reference — locked answers for common interview questions on this pipeline

---

## Walk me through your CI/CT pipeline

"We have an 11-stage CI/CT pipeline on a self-hosted shell runner on a bastion host in an air-gapped environment.

It starts with lint — we catch syntax errors, Dockerfile issues, and Helm chart problems in seconds before an expensive build runs. Then Maven build produces the JAR. Unit tests run with JUnit — JaCoCo generates the coverage report which flows into the next stage.

SonarQube is our main quality gate — we run static analysis and then poll the SonarQube server API for the quality gate result. The gate is configured on the server: 90% coverage, zero critical bugs, zero unresolved security hotspots. If it fails, the pipeline stops there.

After quality gate, we package the JAR to Artifactory versioned by commit SHA, then Docker build on UBI8 base. Then Trivy CVE scan — we run in client mode connecting to our internal Trivy server since we're air-gapped. CRITICAL CVEs hard-block with --exit-code 1. Only a clean image reaches Quay.

After push, we do an automatic rolling deploy to the dev namespace — oc rollout status acts as the runtime gate, readiness probes must pass. Then integration tests against the live dev URL. Finally, lab deployment is a manual gate — dev lead clicks play in GitLab UI.

Production is outside this pipeline entirely. It goes through a formal change management process — change ticket, approval, dedicated deployment team, maintenance window. That's by design for a platform carrying live telecom traffic."

---

## Why CI/CT and not CI/CD?

"We operate a SIP-based contact center platform for a carrier. It handles real-time voice calls. Automated production deployment of that platform is not acceptable — a bad deploy during business hours directly impacts live calls. So the pipeline owns everything through lab — that's the continuous part. Production is a controlled, change-managed release. Same principle as automotive software — you don't continuously deploy to a car's braking system. CI/CT is the correct model for safety and availability-critical systems."

---

## How do you gate between stages?

"Every gate is an exit code. Non-zero exit fails the job, stops the pipeline. For Trivy, --exit-code 1 makes it return non-zero when CRITICAL CVEs are found. For SonarQube, we explicitly poll the API and call exit 1 if the quality gate is not OK — we don't rely on the scanner's exit code because the server evaluates the gate asynchronously after the scanner returns. For lab deployment, when: manual in the rules block pauses the pipeline until a human approves. OCP readiness probes are the runtime gate — oc rollout status times out and fails the job if pods don't become healthy."

---

## How does data flow between stages?

"GitLab artifact paths. The build stage declares target/*.jar and target/classes/ as artifacts. Downstream stages list build in their dependencies block — GitLab downloads those files to the runner workspace automatically before the job starts. JaCoCo XML from the test stage flows to SonarQube the same way. Docker image is different — it lives in the Docker daemon on the bastion, so all three stages that touch it — build, trivy, push — reference it by name rather than as a file artifact. That works because they all run on the same shell runner sharing the same daemon. Commit SHA threads through everything as the version key — JAR filename, image tag, Trivy report name — so you can trace any production deployment back to its exact commit."

---

## Why shell runner on bastion?

"Air-gapped environment — the bastion has controlled, audited access to both the internal tooling network and the OCP cluster. Shell executor means all tools — Maven, Docker, Trivy client, SonarQube scanner, oc CLI — are version-locked on the bastion. Simpler to manage and audit than Docker-in-Docker or Kubernetes runners in an air-gapped setup. The tradeoff is tool versions are fixed on the bastion, but for a stable enterprise platform that's acceptable."

---

## How does Trivy work in air-gapped mode?

"Trivy supports a client-server architecture. An internal VM runs trivy server — it maintains an offline-synced copy of the CVE database, refreshed daily via a controlled process. Our pipeline connects to that server using the --server flag instead of downloading the database from the internet. The pipeline is a thin client — it sends the image to the server, the server does the matching against its local database, returns the results. This keeps CVE coverage current without opening internet egress from the build network."

---

## How does SonarQube quality gate work?

"Two parts — server configuration and pipeline polling. The quality gate thresholds — 90% coverage, zero critical bugs, zero security hotspots — are configured once on the SonarQube server UI. The pipeline doesn't define these thresholds. What the pipeline does is run sonar-scanner to upload the analysis, then poll the SonarQube REST API to wait for the server to finish evaluating. The scanner returns before the server is done — so you have to poll. We call GET /api/qualitygates/project_status and if it returns anything other than OK, we exit 1. That's what actually stops the pipeline."

---

## How do you handle rollback?

"Two mechanisms. For direct deployments — oc rollout undo deployment/myapp which reverts to the previous ReplicaSet. OCP keeps the last 10 by default. For the GitOps IPSG pipeline — git revert the image tag commit in the manifests repo, push, ArgoCD detects it and syncs back to the previous tag automatically. Git history is the source of truth. No manual oc commands needed for GitOps rollback."

---

## What is the GitOps model for IPSG?

"IPSG has multiple interdependent resources — Deployment, Services, Route, ConfigMap, Secret. Managing those as separate YAMLs was getting unwieldy. We packaged everything into a Helm chart with environment-specific values files. The CI pipeline's last stage, instead of deploying directly, updates one line in the manifests repo — the image tag — using yq. ArgoCD watches the manifests repo, detects the change, renders the Helm chart with the new tag, and syncs to the cluster. CI owns everything up to updating git. ArgoCD owns everything after. Rollback is a git revert — ArgoCD auto-syncs back."

---

## Key numbers to remember

| Thing | Value |
|---|---|
| Pipeline stages | 11 |
| Runner type | Self-hosted shell executor |
| Runner host | Bastion (air-gapped) |
| Coverage gate (pipeline) | 80% |
| Coverage gate (SonarQube) | 90% |
| Trivy block severity | CRITICAL |
| Trivy report severity | HIGH (no block) |
| Rollout timeout | 5 minutes |
| ArgoCD poll interval | 3 minutes |
| Trivy DB refresh | Daily |
| Image base | UBI8 |
| Registry | Quay (internal) |
| Artifact store | Artifactory |

---

## One-liners for rapid recall

**On CI/CT vs CI/CD:** "Continuous through lab. Production is change-managed. By design."

**On shell runner:** "Air-gapped. Bastion has cluster access. Tools version-locked. Simple."

**On UBI8:** "Red Hat backports security patches. Fewer CVEs. Trivy finds less."

**On Trivy air-gap:** "Client-server. Internal server holds offline CVE DB. Daily sync."

**On SonarQube gate:** "Thresholds on server. Pipeline polls API. Server decides pass/fail."

**On artifact flow:** "dependencies: block. GitLab downloads automatically. SHA ties everything."

**On GitOps rollback:** "git revert the tag commit. ArgoCD detects. Syncs back. Done."
