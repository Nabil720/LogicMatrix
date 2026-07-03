# Future Improvements

---

## 1. Secret Management — HashiCorp Vault

**Problem:** As the app grows, we'll need to store sensitive data like API keys, passwords, and certificates securely.

**Solution:** Use **HashiCorp Vault** with the **Kubernetes Vault Agent Injector**.

**How it works:**
- When a pod starts, a Vault sidecar is automatically injected into it
- The sidecar authenticates to Vault using the pod's Kubernetes service account
- Secrets are fetched and written to a shared in-memory volume inside the pod
- The app reads secrets as files — they never appear in environment variables or `kubectl describe`

**Why it's better than plain Kubernetes Secrets:**
- Secrets are always encrypted
- Full audit log of who accessed what and when
- Supports automatic secret rotation
- Fine-grained access policies per application

---

## 2. Pipeline Security — SonarQube + Trivy

### SonarQube (Code Scanning)

Scans source code **before** the Docker image is built to catch issues early.

**What it finds:**
- Hardcoded credentials in code
- Security vulnerabilities (injection, insecure functions)
- Code quality issues and dead code

**Where it runs in the pipeline:**
```
Checkout → Tests → SonarQube Scan → Build Image → Push
```

---

### Trivy (Image Scanning)

Scans the built Docker image for known vulnerabilities **before** it is pushed to the registry.

**What it finds:**
- CVEs in base OS packages (e.g. Alpine, Debian)
- Vulnerable library versions
- Dockerfile misconfigurations

**Where it runs in the pipeline:**
```
Build Image → Trivy Scan → Push to Registry
```

> If HIGH or CRITICAL vulnerabilities are found, the pipeline fails and the image is not pushed.

---

## 3. Monitoring — Prometheus + Grafana

**Prometheus** collects metrics from the cluster and application pods on a regular schedule.

**Grafana** displays those metrics as dashboards and sends alerts when something goes wrong.

**What you can monitor:**
- Pod CPU and memory usage
- Request rate, error rate, and response time
- Node health and cluster resource usage

**Example alert:** Notify on Slack/email if the error rate goes above 5% for more than 5 minutes.

---

## 4. Log Analysis — Elastic APM

**APM (Application Performance Monitoring)** gives deeper insight than metrics alone.

**What it provides:**
- Full request traces from start to finish across all services
- Grouped error reports with stack traces
- Slow endpoint detection
- Link a log line directly to the request that caused it
- Visual service dependency map

By adding the Elastic APM agent to the Go backend, every request is automatically traced with no manual instrumentation needed.

---

## 5. GitOps — Argo CD

**Problem:** After CI updates the image tag in the manifest files, someone still has to run `kubectl apply` manually.

**Solution:** **Argo CD** watches the git repository and automatically applies any changes to the cluster.

**How it works:**
1. CI pipeline builds and pushes the new image
2. Pipeline commits the updated image tag to `K8s-manifest/`
3. Argo CD detects the git change
4. Argo CD applies the updated manifests to the cluster automatically

**Benefits:**
- Every deployment is a git commit — full history and auditability
- Argo CD alerts when the cluster drifts from what's in git
- Rollback is just a `git revert`
- No one needs direct `kubectl` access to deploy

---

## Roadmap

| Phase | Improvement |
|---|---|
| 1 | Vault + Agent Injector for secret management |
| 2 | SonarQube for code scanning in CI |
| 3 | Trivy for image scanning in CI |
| 4 | Prometheus + Grafana + Elastic APM for observability |
| 5 | Argo CD for automated GitOps deployments |
