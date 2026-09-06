# GitOps Variant — ArgoCD + Helm for IPSG

For the IPSG microservice, the standard pipeline's deploy stage is replaced with a manifests repo update. ArgoCD handles the actual deployment. This is the two-repo GitOps model.

---

## Why GitOps for IPSG?

IPSG manages multiple interdependent Kubernetes resources — Deployment, ClusterIP Service, MetalLB LoadBalancer Service, Route, ConfigMap, Secret. Managing these as separate YAMLs across environments became error-prone. Packaging them into a Helm chart with environment-specific values files (`values-dev.yaml`, `values-lab.yaml`) and letting ArgoCD manage the sync gave us:

- **One-command full-stack deployment** via Helm
- **Git as the source of truth** — the cluster state is always derivable from the manifests repo
- **Automatic drift detection and self-healing** via ArgoCD
- **Rollback = git revert** — no cluster state to manually unwind

---

## Two-Repo Model

```
Repo 1: Application source repo
├── src/
├── Dockerfile
├── .gitlab-ci.yml     ← CI/CT pipeline lives here
└── pom.xml

Repo 2: Manifests repo (separate repo)
├── helm/
│   └── ipsg-chart/
│       ├── Chart.yaml
│       ├── values.yaml          ← base values
│       ├── values-dev.yaml      ← dev overrides
│       └── values-lab.yaml      ← lab overrides
└── argocd-apps/
    ├── ipsg-dev.yaml            ← ArgoCD Application CR for dev
    └── ipsg-lab.yaml            ← ArgoCD Application CR for lab
```

**Why separate repos:**
- App code changes and infrastructure/config changes have separate git histories
- ArgoCD watches only the manifests repo — no noise from code commits
- Different teams can have different access to each repo
- Rollback on manifests repo is clean — only one line ever changes per CI run

---

## Modified Pipeline — Last Stage

Everything up to push-image is identical to the standard pipeline. Only the deploy stage changes:

```yaml
# Replaces deploy-dev stage for IPSG
update-manifests:
  stage: deploy-dev
  script:
    - echo "Updating manifests repo — new image tag: $IMAGE_TAG"

    # Clone the separate manifests repo
    - git clone https://gitlab.example.com/platform/ipsg-manifests.git
    - cd ipsg-manifests

    # Update only the image tag — one line change in values file
    # yq is a YAML command-line editor
    - yq e '.image.tag = "'$IMAGE_TAG'"' -i helm/ipsg-chart/values-dev.yaml

    # Commit and push
    - git config user.email "ci-bot@example.com"
    - git config user.name "GitLab CI"
    - git add helm/ipsg-chart/values-dev.yaml
    - git commit -m "ci: update image tag to $IMAGE_TAG [skip ci]"
    - git push origin main

    - echo "Manifests updated — ArgoCD will detect and sync"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

**`[skip ci]` in commit message** — prevents GitLab from triggering another pipeline when CI bot pushes to the manifests repo. Without this, you get an infinite loop: CI pushes → pipeline triggers → CI pushes → pipeline triggers...

---

## What Happens After the Push

```
CI pipeline ends — its job is done

ArgoCD (running in openshift-gitops namespace) polls manifests repo
  every 3 minutes (or immediately via webhook if configured)
     │
     ▼
ArgoCD detects values-dev.yaml changed (image.tag is different)
     │
     ▼
ArgoCD renders Helm chart with updated values
helm template ipsg-chart -f values-dev.yaml
     │
     ▼
ArgoCD compares rendered manifests to current cluster state
(this is the "diff" — what changed)
     │
     ▼
ArgoCD syncs — applies the diff to the cluster
oc apply -f <rendered manifests>
     │
     ▼
OCP rolling update — new pods with new image tag come up
Old pods terminate after new pods pass readiness
     │
     ▼
ArgoCD shows: Synced + Healthy
```

---

## ArgoCD Application CRs

One Application CR per environment, all stored in manifests repo under `argocd-apps/`:

```yaml
# argocd-apps/ipsg-dev.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ipsg-dev
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://gitlab.example.com/platform/ipsg-manifests.git
    targetRevision: main
    path: helm/ipsg-chart
    helm:
      valueFiles:
        - values-dev.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: ipsg-dev
  syncPolicy:
    automated:
      prune: true        # delete resources removed from manifests
      selfHeal: true     # revert manual cluster changes back to git state
    syncOptions:
      - CreateNamespace=true
```

```yaml
# argocd-apps/ipsg-lab.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ipsg-lab
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://gitlab.example.com/platform/ipsg-manifests.git
    targetRevision: main
    path: helm/ipsg-chart
    helm:
      valueFiles:
        - values-lab.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: ipsg-lab
  syncPolicy:
    automated: {}        # no selfHeal for lab — manual sync only
```

**Bootstrap — applied once manually:**
```bash
oc apply -f argocd-apps/ -n openshift-gitops
```
After this, ArgoCD manages itself and the applications. Never need to touch it again unless adding a new environment.

---

## Rollback

**CI pipeline rollback:** If the latest deploy broke something in dev, find the previous good commit SHA from the manifests repo git log:

```bash
git log --oneline helm/ipsg-chart/values-dev.yaml
# abc1234 ci: update image tag to abc1234
# def5678 ci: update image tag to def5678  ← previous good one

git revert abc1234
git push origin main
```

ArgoCD detects the revert, syncs, deploys the previous image tag. No manual oc commands needed.

**ArgoCD UI rollback:** ArgoCD keeps history of all previous syncs. You can click "History and Rollback" in the ArgoCD UI and roll back to any previous sync directly.

---

## Responsibility Split

```
CI Pipeline owns:
  lint → build → test → quality gate → scan → push image → update git tag

ArgoCD owns:
  detect git change → render helm chart → sync to cluster → self-heal → rollback
```

CI is done once the image tag is in git. ArgoCD is the deployment engine. They never overlap.
