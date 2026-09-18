# Kubernetes for DevOps Interviews (4+ Years Experience)

## 1. Basic Syntax

**Deployments and Services**

*Deployment (Manages Pods and Replicas)*
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
        ports:
        - containerPort: 80
```

*Service (Exposes the Deployment)*
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx # Matches the Deployment labels
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP # Internal access only
```

---

## 2. Intermediate Examples

**ConfigMaps, Secrets, Requests/Limits, and Probes**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend-api
        image: myapi:v2
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: database_host
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: backend-secrets
              key: database_password
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
```

---

## 3. Advanced Examples

**DaemonSets, Taints/Tolerations, Node Affinity, and Network Policies**

*Ensuring a pod runs on a specific node (Affinity)*
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: heavy-worker
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-type
                operator: In
                values:
                - memory-optimized
      containers:
      - name: worker
        image: worker:latest
```

*Network Policy (Default Deny ingress except from specific namespace)*
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-network-policy
  namespace: database
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          project: myapp
    ports:
    - protocol: TCP
      port: 5432
```

---

## 4. Interview Coding Exercises

### Problem 1: Injecting Configs as Files
**Task:** You have a ConfigMap named `app-settings`. Mount it as a file inside the container at `/etc/config/settings.json`.

**Solution:**
```yaml
# Inside the pod spec:
      volumes:
      - name: config-volume
        configMap:
          name: app-settings
      containers:
      - name: myapp
        image: myapp:v1
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
```

### Problem 2: RBAC (Role-Based Access Control)
**Task:** Write a Role and RoleBinding to allow a ServiceAccount named `ci-bot` to read (get, list, watch) pods in the `dev` namespace.

**Solution:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
- kind: ServiceAccount
  name: ci-bot
  namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 5. Troubleshooting Exercises

### Broken Configuration 1: Service not routing to Pod
**Scenario:** A Deployment has pods running, but a `curl` to the ClusterIP Service returns Connection Refused.
**What is wrong?**
```yaml
# Deployment labels:
  labels:
    app: frontend
    tier: web

# Service spec:
  selector:
    app: frontend-web
```
**Answer:** The Service selector (`app: frontend-web`) does not match the Pod labels (`app: frontend`). Update the service selector to exactly match the pod labels.

### Broken Configuration 2: CrashLoopBackOff
**Scenario:** A pod is in `CrashLoopBackOff` state. How do you troubleshoot?
**Answer (Interview Script):**
1. Run `kubectl describe pod <pod-name>` to check events (OOMKilled? Failed to pull image? Liveness probe failed?).
2. Run `kubectl logs <pod-name>` to check application standard output/error.
3. If logs crash too fast, run `kubectl logs <pod-name> --previous` to see logs from the previously crashed container.

---

## 6. Common Interview Questions

**Q: "What is the difference between a StatefulSet and a Deployment?"**
*Answer:* 
Deployments are for stateless applications. Pods are interchangeable and get random hashes in their names. 
StatefulSets are for stateful apps (databases, message queues). They provide sticky, unique network identifiers (pod-0, pod-1), ordered deployment/scaling, and stable persistent storage (using volumeClaimTemplates so each pod keeps its specific disk).

**Q: "Explain Liveness vs. Readiness vs. Startup Probes."**
*Answer:*
- **Liveness:** Checks if the container is dead (e.g., deadlock). If it fails, K8s restarts the container.
- **Readiness:** Checks if the app is ready to serve traffic. If it fails, K8s stops sending traffic to it (removes it from Service endpoints).
- **Startup:** Used for slow-starting legacy apps. Disables liveness/readiness until the startup probe passes.

**Q: "How does Ingress differ from a LoadBalancer Service? What Ingress controllers have you worked with?"**
*Answer:* A LoadBalancer service spins up a 1:1 cloud load balancer (e.g., AWS ALB/NLB) per service, which is expensive. Ingress is a K8s resource that provides HTTP/HTTPS routing rules based on path or host. In AWS EKS I have used the AWS Load Balancer Controller to translate Ingress resources into ALBs. I have also worked with NGINX Ingress, but for new platform designs in 2026 I would evaluate Gateway API or a supported controller because the open-source ingress-nginx project has been retired.

**Q: "Are you aware the open-source Nginx Ingress controller is being retired — does this affect an ALB controller setup that underlies Nginx Ingress?"**
*Answer:* No. The retirement of ingress-nginx does not mean the AWS Load Balancer Controller is retired. They are separate projects and architectures. If an environment uses AWS Load Balancer Controller to manage ALBs from Kubernetes Ingress resources, that controller is not inherently affected by ingress-nginx retirement. The retirement matters only if the cluster actually runs ingress-nginx. For 2026, I would plan migration to Gateway API or another supported controller.

**Q: "How do you handle programmatic access to Kubernetes clusters without a password (RBAC, ServiceAccount, token)?"**
*Answer:* For automation I use a dedicated ServiceAccount or, on EKS, a workload identity mechanism such as IRSA. The identity gets only the RBAC it needs through Role/RoleBinding or ClusterRole/ClusterRoleBinding. I avoid long-lived static tokens where possible and use projected, short-lived credentials and external identity integration.

**Q: "If you want to implement Kubernetes RBAC plus IAM integration, what would be your approach?"**
*Answer:* For human access, IAM authenticates the user and EKS maps that identity to Kubernetes authorization, using EKS access entries where supported or legacy aws-auth where still required. For workloads, I use ServiceAccounts plus IRSA or EKS Pod Identity and then Kubernetes RBAC for Kubernetes API permissions. IAM should control AWS APIs; Kubernetes RBAC should control Kubernetes APIs. I avoid merging those responsibilities into a single broad role.

**Q: "If someone deletes the aws-auth ConfigMap, how would you recover?"**
*Answer:* In 2026 I would treat `aws-auth` as legacy because EKS access entries are the recommended access mechanism. If a legacy cluster loses `aws-auth`, I use the cluster creator or another existing access path to restore the ConfigMap, following AWS's documented recovery procedure. If access entries are enabled, I prefer migrating user access to access entries so future recovery does not depend on editing the ConfigMap manually.

---

## 7. Cheat Sheet

| Command | Usage (4+ Yrs Experience Focus) |
| :--- | :--- |
| `kubectl get events --sort-by='.metadata.creationTimestamp'` | Find out exactly what K8s is doing chronologically. |
| `kubectl auth can-i create pods --as ci-bot` | Test RBAC permissions. |
| `kubectl port-forward svc/my-db 5432:5432` | securely access internal DB from localhost. |
| `kubectl rollout restart deploy/my-app` | Gracefully restart pods in a deployment. |
| `kubectl rollout undo deploy/my-app` | Rollback to the previous ReplicaSet quickly during an outage. |
| `kubectl top pods` | Check real-time CPU/Memory usage (requires Metrics Server). |
| `kubectl get po -o wide` | See which node a pod is scheduled on. |
