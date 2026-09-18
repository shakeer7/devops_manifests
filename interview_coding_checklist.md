# DevOps Engineer Interview Coding Checklist (4+ Years Exp)

Use this checklist to verify your readiness for live coding or whiteboard rounds. For a Mid/Senior level role, you should be able to write or confidently explain the following **from memory**.

## 1. Terraform
- [ ] Write a `provider` block.
- [ ] Write a basic `resource` block (e.g., AWS S3 or Azure Resource Group).
- [ ] Define variables (`variable` block) with type, description, and default.
- [ ] Reference variables using `var.my_var`.
- [ ] Use `locals` to manipulate data or create tags.
- [ ] Write a `data` block to fetch existing infrastructure.
- [ ] Conditionally create a resource using `count = var.enabled ? 1 : 0`.
- [ ] Loop over a list or map using `for_each = toset(var.list)` and `each.key`/`each.value`.
- [ ] Write a `dynamic` block (e.g., for security group ingress rules).
- [ ] Define an `output` block.
- [ ] Explain CLI commands: `init`, `plan`, `apply`, `destroy`, `import`, `state mv`.

## 2. Kubernetes
- [ ] Write a `Deployment` YAML from scratch (apiVersion, kind, metadata, spec.replicas, selector, template, container image, ports).
- [ ] Write a `Service` YAML (ClusterIP or NodePort) that correctly selects the Deployment via labels.
- [ ] Add `requests` and `limits` to a container spec.
- [ ] Add `livenessProbe` and `readinessProbe` to a container spec.
- [ ] Mount a `ConfigMap` or `Secret` as an environment variable (`valueFrom`).
- [ ] Mount a `ConfigMap` as a volume file (`volumeMounts` and `volumes`).
- [ ] Write a basic `Ingress` rule mapping a host/path to a backend Service.
- [ ] Explain CLI commands: `kubectl get`, `describe`, `logs`, `exec`, `port-forward`, `rollout restart`.

## 3. Docker
- [ ] Write a standard `Dockerfile` (FROM, WORKDIR, COPY, RUN, EXPOSE, CMD).
- [ ] Write a Multi-stage `Dockerfile` (using `AS builder` and `COPY --from=builder`).
- [ ] Run a container via CLI mapping ports and volumes (`docker run -p 80:80 -v /host:/container image`).
- [ ] Write a basic `docker-compose.yml` with two linked services (web and db).

## 4. CI/CD (Azure DevOps / GitHub Actions)
- [ ] Write a pipeline trigger (e.g., branch filter).
- [ ] Define jobs and steps.
- [ ] Write a bash script step to print variables.
- [ ] Pass variables from one job to another.
- [ ] Use conditions (`condition: eq(...)` or `if:`).
- [ ] Reference secrets securely.

## 5. Python
- [ ] Read and write a JSON file.
- [ ] Make an HTTP GET/POST request (using `requests`).
- [ ] Execute a shell command and capture the output (`subprocess.run`).
- [ ] Parse text using basic Regex (`re.search` or `re.findall`).
- [ ] Use a `try...except` block for error handling.
- [ ] Create a basic loop/dictionary manipulation script.

## 6. Bash
- [ ] Define and use variables (`$VAR`).
- [ ] Read arguments passed to the script (`$1`, `$2`).
- [ ] Write an `if/else` statement testing if a file exists (`-f`) or string is empty (`-z`).
- [ ] Write a `for` loop to iterate over files in a directory.
- [ ] Parse a log file stream using `grep`, `awk`, and `sort`.
- [ ] Redirect standard error to standard out (`2>&1`).

## 7. Git
- [ ] Clone, checkout new branch, add, commit, and push.
- [ ] Resolve a merge conflict manually.
- [ ] `cherry-pick` a commit.
- [ ] `rebase` a feature branch onto main.
- [ ] Undo the last commit using `git reset`.

## 8. Helm & ArgoCD
- [ ] Use Go templating `{{ .Values.myvar }}` in a Helm YAML file.
- [ ] Write an `{{ if }}` block in Helm.
- [ ] Write an ArgoCD `Application` YAML targeting a Git repo and K8s namespace.

## 9. Observability & Security
- [ ] Write a PromQL query using `rate()` and `sum by()`.
- [ ] Explain how to mount an external secret in K8s.
- [ ] Write an IAM policy (JSON) granting Least Privilege access.

## General Interview Advice
1. **The Interview Formula:** For scenario questions, use the structure: Situation -> decision -> implementation -> validation -> trade-off -> monitoring/rollback.
2. **Think out loud:** Interviewers care more about *how* you debug or structure a problem than perfect syntax.
3. **Start simple:** Build the MVP (Minimum Viable Product) script/config first, then add the error handling, loops, and security hardening.
4. **Say "I don't know, but here is how I would find out":** If you forget the exact YAML syntax for a Kubernetes Volume, say you would use `kubectl explain pod.spec.volumes` or check the official docs.
