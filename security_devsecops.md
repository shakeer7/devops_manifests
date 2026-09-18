# Security & DevSecOps for Interviews (4+ Years Experience)

## 1. Basic Syntax

**Trivy (Container Scanning) and SonarQube (SAST)**

*Trivy CLI commands (Image Scanning)*
```bash
# Scan a local Docker image for OS and library vulnerabilities
trivy image myapp:latest

# Scan a Terraform/Kubernetes directory for misconfigurations (IaC scanning)
trivy config ./infrastructure/

# Output findings as JSON for CI/CD ingestion
trivy image --format json --output results.json myapp:latest
```

*SonarQube (Static Application Security Testing)*
```bash
# Example Sonar-Scanner command used in CI/CD
sonar-scanner \
  -Dsonar.projectKey=my_backend_api \
  -Dsonar.sources=src \
  -Dsonar.host.url=https://sonarqube.mycompany.com \
  -Dsonar.login=$SONAR_TOKEN
```

---

## 2. Intermediate Examples

**Secrets Management and Kubernetes Security**

*External Secrets Operator (Kubernetes)*
Instead of native K8s `Secrets` (which are just base64 encoded), use External Secrets to pull from AWS Secrets Manager.
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: "1h"
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials-k8s # The native K8s secret it will create
  data:
  - secretKey: password
    remoteRef:
      key: production/db/password
```

*Kubernetes Pod Security Context*
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: myapp
    image: myapp:v1
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
```

---

## 3. Advanced Examples

**CI/CD Pipeline Security (Azure DevOps / GitHub Actions)**

*DevSecOps Pipeline (YAML)*
```yaml
steps:
  # 1. SAST (Code Analysis)
  - task: SonarQubeAnalyze@5
  
  # 2. Secret Scanning (e.g., Gitleaks)
  - script: |
      gitleaks detect --source . -v
    displayName: 'Scan for hardcoded secrets'

  # 3. Build Image
  - script: docker build -t myapp:$(Build.BuildId) .
  
  # 4. Container Scanning (Trivy)
  - script: |
      trivy image --exit-code 1 --severity CRITICAL,HIGH myapp:$(Build.BuildId)
    displayName: 'Vulnerability Scan'
    # --exit-code 1 ensures the pipeline FAILS if High/Critical vulns are found

  # 5. Push Image (Only if all security checks pass)
  - script: docker push myapp:$(Build.BuildId)
```

---

## 4. Interview Coding Exercises

### Problem 1: RBAC Least Privilege
**Task:** Write a Terraform AWS IAM Policy that allows a developer to list S3 buckets, but only allows them to upload/download files to a specific bucket named `company-dev-assets`.

**Solution:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListAllMyBuckets"],
      "Resource": "arn:aws:s3:::*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::company-dev-assets/*"
    }
  ]
}
```
**Explanation:** The first block grants generic List permissions globally (required by the AWS console). The second block restricts actual data manipulation strictly to the contents (`/*`) of the specified bucket.

### Problem 2: IaC Misconfiguration
**Task:** Review this Terraform snippet. Identify the security flaw and fix it.
```hcl
resource "aws_security_group" "db_sg" {
  ingress {
    from_port   = 3306
    to_port     = 3306
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```
**Solution:** The database port (3306) is open to the entire internet (`0.0.0.0/0`).
**Fix:** Restrict it to the VPC CIDR or a specific Application Security Group.
```hcl
    cidr_blocks = ["10.0.0.0/16"] # Internal VPC only
    # OR better:
    # security_groups = [aws_security_group.app_sg.id]
```

---

## 5. Troubleshooting Exercises

### Broken Configuration 1: Read-Only Filesystem
**Scenario:** You implemented K8s `readOnlyRootFilesystem: true` for security. Now the application pod is crashlooping with the error: `mkdir /app/logs: Read-only file system`.
**Answer:** The app needs to write logs, but the root filesystem is locked.
**Fix:** Mount a temporary, writable volume (emptyDir) specifically for the logs folder.
```yaml
        volumeMounts:
        - mountPath: /app/logs
          name: tmp-logs
      volumes:
      - name: tmp-logs
        emptyDir: {}
```

---

## 6. Common Interview Questions

**Q: "What is the difference between SAST, DAST, and SCA?"**
*Answer:*
- **SAST (Static Application Security Testing):** Scans source code for flaws (e.g., SQL injection, buffer overflows) without running the code. (e.g., SonarQube, Checkmarx).
- **DAST (Dynamic Application Security Testing):** Interacts with the running application from the outside, throwing simulated attacks at the web interface/APIs to find vulnerabilities. (e.g., OWASP ZAP).
- **SCA (Software Composition Analysis):** Scans `package.json`, `pom.xml`, etc., to identify vulnerable open-source third-party dependencies. (e.g., Snyk, Dependabot).

**Q: "How do you handle a zero-day vulnerability in a Docker image?"**
*Answer:*
1. Identify all running containers using the vulnerable image via K8s dashboards or security tools.
2. Wait for (or create) a patch/updated base image.
3. Update the Dockerfile `FROM` instruction to the patched version.
4. Rebuild the image, push to the registry.
5. Restart/Rollout the K8s deployments to pull the new image.
6. Verify via Trivy/Security scanners that the vulnerability is gone.

**Q: "What is Mutual TLS (mTLS) and how does a Service Mesh provide it?"**
*Answer:* mTLS ensures that traffic between two microservices is encrypted and that *both* sides verify each other's cryptographic certificates. A service mesh (like Istio or Linkerd) injects a sidecar proxy into every pod. The proxies handle the mTLS encryption/decryption transparently, so the application code doesn't need to manage certificates.

---

## 7. Cheat Sheet

| Concept | Explanation for Interviews |
| :--- | :--- |
| **Shift Left** | Integrating security checks early in the CI/CD pipeline rather than waiting for production. |
| **RBAC** | Role-Based Access Control. Grant permissions to roles, assign users to roles. |
| **Least Privilege** | Granting only the bare minimum permissions required to perform a task. |
| **OIDC / Workload Identity** | Eliminates static cloud credentials. Allows CI/CD or K8s pods to authenticate to AWS/Azure using temporary federated tokens. |
