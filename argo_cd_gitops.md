# Argo CD & GitOps for DevOps Interviews (4+ Years Experience)

## 1. Basic Syntax

**Application Manifests**

GitOps relies on defining everything declaratively. Instead of running `kubectl apply` or `helm install`, you define an Argo CD `Application` Custom Resource (CR).

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/myorg/gitops-repo.git'
    targetRevision: HEAD
    path: apps/myapp/overlays/production
  destination:
    server: 'https://kubernetes.default.svc' # Deploy to the same cluster Argo is on
    namespace: myapp-prod
  syncPolicy:
    automated:
      prune: true     # Delete resources that are removed from Git
      selfHeal: true  # Revert manual changes made in the K8s cluster
```

---

## 2. Intermediate Examples

**Helm Integrations and Ignore Differences**

Argo CD natively understands Helm. You don't need to run `helm template`.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: redis-cluster
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://charts.bitnami.com/bitnami'
    chart: redis
    targetRevision: 17.11.3
    helm:
      valueFiles:
      # These must exist in a separate git repo mapped in Argo, or use inline values
      - values-production.yaml 
      values: |
        architecture: replication
        auth:
          enabled: true
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: database
  ignoreDifferences:
    # Ignore dynamic fields injected by mutating webhooks (like Linkerd/Istio)
    - group: apps
      kind: Deployment
      jsonPointers:
      - /spec/template/metadata/annotations/linkerd.io~1inject
```

---

## 3. Advanced Examples

**ApplicationSets (Managing Multi-Cluster Deployments)**

ApplicationSets allow you to template Argo CD Applications. This is used for deploying one app to 50 clusters, or deploying 50 apps to one cluster dynamically.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-addons
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - cluster: dev-cluster
        url: https://10.0.1.5
      - cluster: prod-cluster
        url: https://10.0.1.6
  template:
    metadata:
      name: '{{cluster}}-promtail'
    spec:
      project: default
      source:
        repoURL: 'https://github.com/myorg/gitops-repo.git'
        targetRevision: HEAD
        path: addons/promtail
      destination:
        server: '{{url}}'
        namespace: logging
      syncPolicy:
        automated:
          prune: true
```

---

## 4. Interview Coding Exercises

### Problem 1: Structuring a GitOps Repository
**Task:** An interviewer asks you to design the folder structure for a GitOps repository that manages a frontend and backend app across `dev` and `prod` environments using Kustomize.

**Solution:**
```text
gitops-repo/
├── apps/
│   ├── frontend/
│   │   ├── base/               # Kustomization base (Deployments, Services)
│   │   ├── overlays/
│   │   │   ├── dev/            # dev specific patches (replicas=1)
│   │   │   └── prod/           # prod specific patches (replicas=5)
│   └── backend/
│       ├── base/
│       └── overlays/
│           ├── dev/
│           └── prod/
└── argocd-apps/                # The ArgoCD Application CRDs
    ├── dev-frontend-app.yaml
    ├── prod-frontend-app.yaml
    └── ...
```

### Problem 2: Sync Waves and Hooks
**Task:** You are deploying an app that requires a database schema migration to run *before* the new application pods start. How do you do this in Argo CD?
**Solution:**
Add Argo CD Sync Wave and Hook annotations to your K8s manifests in Git.
*Migration Job (runs first):*
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/sync-wave: "1"
```
*Application Deployment (runs after):*
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "2"
```

---

## 5. Troubleshooting Exercises

### Broken Configuration 1: Out of Sync State
**Scenario:** Your Argo CD dashboard shows an Application as "Out of Sync". The Git commit matches what is deployed, but Argo keeps trying to remove a field from a Deployment, and a K8s controller keeps adding it back.
**Answer:** This is a classic GitOps conflict with Mutating Admission Webhooks or controllers (like HPA changing `replicas` on a Deployment). 
**Fix:** Add the conflicting field to the `ignoreDifferences` block in the Argo CD Application spec (e.g., ignore the `replicas` field if HPA is managing it).

### Broken Configuration 2: Self-Heal Overwriting Hotfixes
**Scenario:** An incident occurs at 2 AM. An SRE manually edits a Deployment in the cluster via `kubectl edit` to bump memory limits. Two minutes later, the pods crash again because the memory limit reverted back.
**Answer:** Argo CD `selfHeal` was enabled. It noticed the K8s cluster state drifted from the Git state and automatically reverted the manual change.
**Fix:** The SRE must temporarily pause syncing in the Argo CD UI/CLI, or make the change directly in the Git repository so Argo CD applies it as the source of truth.

---

## 6. Common Interview Questions

**Q: "What are the core principles of GitOps?"**
*Answer:* 
1. The entire system is described declaratively.
2. The canonical desired system state is versioned in Git.
3. Approved changes to the desired state are automatically applied to the system.
4. Software agents (like Argo CD/Flux) ensure correctness and alert on divergence (drift).

**Q: "Push vs. Pull based CI/CD. Where does Argo CD fit?"**
*Answer:* Traditional CI/CD (like Jenkins or ADO pushing to K8s) is **Push-based**. The CI tool needs cluster credentials to execute `kubectl apply`. Argo CD is **Pull-based**. Argo runs *inside* the K8s cluster, securely pulls manifests from Git, and applies them. This is vastly more secure because the CI server doesn't have god-mode credentials to production clusters.

**Q: "How do you handle secrets in GitOps since you shouldn't commit plaintext secrets to Git?"**
*Answer:* Common patterns include:
1. **Sealed Secrets (Bitnami):** Encrypt secrets locally using a public key; commit the encrypted object. A controller in the cluster decrypts it.
2. **External Secrets Operator:** Define an ExternalSecret CRD in Git. The operator fetches the real secret from AWS Secrets Manager / Azure Key Vault / HashiCorp Vault at runtime.

---

## 7. Cheat Sheet

| Command | Usage (4+ Yrs Experience Focus) |
| :--- | :--- |
| `argocd app create <name> --repo <url> ...` | Create an app via CLI. |
| `argocd app sync <name>` | Manually trigger a sync. |
| `argocd app diff <name>` | View the K8s manifest diff between Git and the Cluster. |
| `argocd app history <name>` | View deployment history. |
| `argocd app rollback <name> <id>` | Rollback to a specific historical sync ID. |
| `argocd admin initial-password` | Retrieve default admin password on first install. |
