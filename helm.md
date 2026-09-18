# Helm for DevOps Interviews (4+ Years Experience)

## 1. Basic Syntax

**Chart Structure, Values, and Templates**

*Standard Chart Structure*
```text
mychart/
  Chart.yaml          # Metadata (name, version)
  values.yaml         # Default configuration values
  charts/             # Chart dependencies
  templates/          # K8s manifest templates
  templates/_helpers.tpl # Named templates / functions
```

*values.yaml*
```yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.24"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
```

*templates/deployment.yaml*
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
```

---

## 2. Intermediate Examples

**Control Structures (`if`, `with`, `range`)**

*Using `if` for conditional resources*
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
spec:
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ .Release.Name }}-service
                port:
                  number: 80
{{- end }}
```
*Note: The `-` in `{{- if }}` strips whitespace/newlines.*

*Using `with` to scope variables*
```yaml
spec:
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

*Using `range` to loop over arrays*
```yaml
# values.yaml:
# env:
#   - name: FOO
#     value: bar

# template:
env:
{{- range .Values.env }}
  - name: {{ .name }}
    value: {{ .value | quote }}
{{- end }}
```

---

## 3. Advanced Examples

**Named Templates, Helpers, and Chart Dependencies**

*templates/_helpers.tpl*
```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "mychart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "mychart.labels" -}}
helm.sh/chart: {{ include "mychart.chart" . }}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

*Applying helpers in a template*
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mychart.name" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
```

*Dependencies (`Chart.yaml`)*
```yaml
dependencies:
  - name: redis
    version: 17.11.3
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

---

## 4. Interview Coding Exercises

### Problem 1: Templating a ConfigMap
**Task:** You need to create a ConfigMap that injects an entire script into a pod. In `values.yaml`, the script is written as a multi-line string. Write the template.

**Solution:**
*values.yaml*
```yaml
initScript: |
  #!/bin/bash
  echo "Initializing DB"
  createdb myapp
```
*templates/configmap.yaml*
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-init-script
data:
  init.sh: |-
{{ .Values.initScript | indent 4 }}
```
**Explanation:** The `|-` yaml operator keeps the multiline string but strips the final trailing newline. The `indent 4` pipe ensures the script is indented correctly underneath the `init.sh:` key.

### Problem 2: Dynamic Environment Variables
**Task:** Convert a dictionary of variables from `values.yaml` into pod environment variables.
*values.yaml*
```yaml
appConfig:
  DB_HOST: "mysql.local"
  LOG_LEVEL: "debug"
```
**Solution:**
```yaml
        env:
          {{- range $key, $val := .Values.appConfig }}
          - name: {{ $key }}
            value: {{ $val | quote }}
          {{- end }}
```

---

## 5. Troubleshooting Exercises

### Broken Configuration 1: Indentation Errors
**Scenario:** Running `helm install` returns: `error converting YAML to JSON: yaml: line 12: mapping values are not allowed in this context`.
**Answer:** This is almost always caused by bad indentation resulting from a template function.
**Fix:** Ensure you are using `nindent` or `indent` properly. For example, `{{ toYaml .Values.resources | nindent 12 }}` ensures that the YAML block generated is correctly indented at 12 spaces on a new line.

### Broken Configuration 2: Scope loss in `with` block
**Scenario:** Inside a `with` block, you try to access the release name `{{ .Release.Name }}` but Helm throws an error.
**Answer:** The `with` block changes the context (the `.` variable) to whatever is passed to `with`. The root context is lost.
**Fix:** Use `$` to access the root context from anywhere: `{{ $.Release.Name }}`.

---

## 6. Common Interview Questions

**Q: "What is the difference between `helm upgrade --install` and `helm install`?"**
*Answer:* `helm install` will fail if a release with that name already exists. `helm upgrade --install` will install the chart if it doesn't exist, and upgrade it if it does. This makes it highly idempotent and the standard command used in CI/CD pipelines.

**Q: "How do you manage Helm secrets?"**
*Answer:* Helm natively stores values in plaintext. To handle secrets, DevOps engineers typically use tools like `helm-secrets` (wrapping Mozilla SOPS), HashiCorp Vault K8s injector, or External Secrets Operator, which fetch secrets at runtime rather than storing them in the Helm chart.

**Q: "How does Helm rollback work?"**
*Answer:* Helm keeps a history of releases as K8s Secrets (in Helm 3) in the cluster. When you run `helm rollback <release> <revision>`, Helm retrieves the state of the K8s manifests from that specific revision and applies them to the cluster.

---

## 7. Cheat Sheet

| Command | Usage (4+ Yrs Experience Focus) |
| :--- | :--- |
| `helm create <name>` | Scaffold a new chart directory structure. |
| `helm template <name> ./chart` | Render templates locally for debugging/validation without sending to K8s. |
| `helm lint ./chart` | Analyze chart for syntax errors and best practices. |
| `helm upgrade --install myapp . --values values-prod.yaml` | Standard deployment command. |
| `helm history <release>` | View past deployments and revisions. |
| `helm rollback <release> 1` | Rollback to revision 1. |
| `helm get manifest <release>` | View the actual YAML applied to the cluster for a running release. |
