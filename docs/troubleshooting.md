# Troubleshooting Guide

---

## 1. Pod is in `CrashLoopBackOff`. What do you check?

The container keeps starting and crashing. Kubernetes is restarting it in a loop.

**Start here:**
```bash
kubectl describe pod <pod-name> -n logicmatrix
kubectl logs <pod-name> -n logicmatrix --previous
```

**Common causes:**
- Application crashes on startup — check logs for panic or fatal errors
- A required environment variable or secret is missing
- The secret or ConfigMap it references doesn't exist
- Wrong startup command or entrypoint in the Dockerfile
- The pod is running out of memory (`OOMKilled` in `describe`)

---

## 2. Deployment is successful but app is not reachable. What do you check?

Work through each layer from the pod upward:

**1. Are pods running and ready?**
```bash
kubectl get pods -n logicmatrix
```

**2. Is the Service selector matching pod labels?**
```bash
kubectl describe svc frontend-svc -n logicmatrix
```

**3. Is the Ingress configured correctly?**
```bash
kubectl describe ingress -n logicmatrix
```

**4. Test connectivity from inside the cluster:**
```bash
kubectl run test --rm -it --image=busybox -- wget -qO- http://frontend-svc.logicmatrix.svc.cluster.local
```

**Common causes:**
- Pod is running but not ready — readiness probe is failing
- Service selector label doesn't match the pod's labels
- Ingress has a wrong hostname, path, or missing `pathType`
- A NetworkPolicy is blocking traffic between services

---

## 3. Difference between Readiness and Liveness probe?

| | Readiness Probe | Liveness Probe |
|---|---|---|
| **Question it answers** | Is the pod ready for traffic? | Is the pod still alive? |
| **On failure** | Pod is removed from load balancer | Pod is killed and restarted |
| **Use case** | App needs time to warm up | App is stuck or deadlocked |

> **Simple rule:** Always set a readiness probe. Only add a liveness probe if your app can hang without crashing on its own.

---

## 4. Docker build works locally but fails in pipeline. Why?

**Most common reasons:**

- **Missing `.dockerignore`** — wrong files are included or excluded from the build context
- **No build cache in CI** — first runs are slower; use `cache-from: type=gha` (already in this pipeline)
- **File not committed to git** — a file referenced in `COPY` exists locally but isn't tracked by git
- **Platform mismatch** — local machine is `arm64` (e.g. Apple M1) but CI runs `amd64`
- **Credentials not set** — a secret available locally hasn't been added to GitHub Actions secrets

---

## 5. Pipeline fails during Docker build. What do you check?

Open the failing step in the GitHub Actions log and look for:

- **Which line in the Dockerfile failed** (`FROM`, `RUN`, `COPY`, etc.)
- **The exact error message** — it usually tells you exactly what's wrong

**Common causes:**
- Base image tag doesn't exist on Docker Hub
- A `COPY` file path is wrong or the file isn't committed
- A `RUN` command fails (build error, missing package, test failure)
- Docker Hub login failed — `DOCKERHUB_TOKEN` secret is missing or expired

---

## 6. Certificate renewal failed. What do you check?

```bash
kubectl describe certificate <name> -n <namespace>
kubectl logs -n cert-manager deploy/cert-manager
```

**Common causes:**
- DNS is not pointing to the cluster's IP address
- Port 80 is blocked — Let's Encrypt can't reach the HTTP-01 challenge endpoint
- The `cert-manager.io/cluster-issuer` annotation is missing from the Ingress
- Let's Encrypt rate limit hit — wait 1 hour or use the staging issuer for testing

---

## 7. Ingress returns 502 or 504. What do you check?

| Code | Meaning |
|---|---|
| **502** | Ingress reached the pod but got a bad response (pod crashed or wrong port) |
| **504** | Ingress reached the pod but it timed out (app is too slow or stuck) |

```bash
kubectl get endpoints -n logicmatrix
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller | tail -30
```

**Common causes:**
- No healthy pod endpoints — pods are crashing or failing readiness checks
- `targetPort` in the Service doesn't match `containerPort` in the Deployment
- App is responding too slowly — increase the Ingress timeout annotation

---

## 8. Vendor SFTP connection to port 22 times out. What do you check?

**Test connectivity from inside the cluster first:**
```bash
kubectl run net-test --rm -it --image=busybox -- nc -zv <vendor-host> 22
```

**Checklist:**
- Is outbound TCP port 22 allowed in your cloud firewall or security group rules?
- Is there a Kubernetes NetworkPolicy blocking egress on port 22?
- Does the vendor hostname resolve? (`nslookup <vendor-host>`)
- Has the vendor whitelisted your cluster's outbound IP address?

```bash
# Find your cluster's egress IP
kubectl run net-test --rm -it --image=busybox -- wget -qO- ifconfig.me
```

---

## 9. Terraform plan wants to recreate the cluster. What do you check?

This is serious — do not run `apply` without understanding why.

```bash
terraform plan -out=tfplan
terraform show tfplan | grep "must be replaced"
```

**Common causes:**

- An **immutable field** was changed — cluster name, region, or node machine type
- A **Kubernetes version upgrade** that requires node pool replacement
- **State drift** — someone changed the cluster manually in the cloud console, and Terraform's state no longer matches reality
- A **provider version upgrade** that changed how a resource is managed

**What to do:**
- If the change is unintentional — revert the Terraform config to match the current state
- If state is drifted — run `terraform refresh` to resync state before planning again
- Never apply a cluster recreation without a backup plan and team approval
