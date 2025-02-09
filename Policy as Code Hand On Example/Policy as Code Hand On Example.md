**Real-World Hands-on Example: Implementing Policy as Code Using Kyverno**

This guide will walk you through a Kyverno Policy as Code (PaC) workflow similar to the image you provided. We'll cover:

✅ Writing a Kyverno policy.

✅ Enforcing it in a Kubernetes CI/CD pipeline.

✅ Automating policy checks before merging code.

✅ Notifying developers on policy violations.

---

1. **Workflow Overview**

This example enforces a security policy that ensures no Kubernetes Pod runs as root.

🔹 Security Team defines policies using Kyverno.

🔹 DevOps Engineer raises a Pull Request (PR) with a Kubernetes deployment.

🔹 CI/CD pipeline runs Kyverno to check policies.

🔹 If the policy check fails, the PR is blocked, and the developer is notified.

🔹 If the policy check passes, the PR is merged and deployed to Kubernetes.

---

2. **Setup Kyverno in Kubernetes**

First, install Kyverno in your Kubernetes cluster.

```bash
kubectl create namespace kyverno
kubectl apply -f https://github.com/kyverno/kyverno/releases/latest/download/kyverno.yaml
```

Verify Kyverno is running:

```bash
kubectl get pods -n kyverno
```

---

3. **Define a Kyverno Policy**
   
This policy prevents Pods from running as root by enforcing a runAsNonRoot rule.

Create a YAML file (disallow-root-pods.yaml):

```bash
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-root-pods
spec:
  validationFailureAction: enforce  # Blocks deployment if policy fails
  background: true
  rules:
  - name: validate-run-as-non-root
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Running as root is not allowed. Set runAsNonRoot: true"
      pattern:
        spec:
          securityContext:
            runAsNonRoot: true
```

Apply the policy:

```bash
kubectl apply -f disallow-root-pods.yaml
```

Check if the policy is active:

```bash
kubectl get clusterpolicy
```

4. **Create a Faulty Kubernetes Deployment**

Create a deployment YAML file that violates the policy (nginx-root.yaml):

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-root
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      securityContext:
        runAsNonRoot: false  # 🚨 This violates the Kyverno policy 🚨
      containers:
      - name: nginx
        image: nginx
```

Try to apply it:

```bash
kubectl apply -f nginx-root.yaml
```
🚨 **Kyverno will block this deployment**

Check the violation logs:

```bash
kubectl get events --sort-by='.lastTimestamp'
```

---

5. **Automate Policy Checks in CI/CD Pipeline**

We'll integrate Kyverno policy checks into a GitHub Actions CI/CD pipeline.

**GitHub Actions Workflow**

Create .github/workflows/policy-check.yml:

```bash
name: Kyverno Policy Check

on:
  pull_request:
    branches:
      - main

jobs:
  policy-check:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Repository
      uses: actions/checkout@v2

    - name: Install Kyverno CLI
      run: |
        curl -LO https://github.com/kyverno/kyverno/releases/latest/download/kyverno-cli-linux-amd64.tar.gz
        tar -xvzf kyverno-cli-linux-amd64.tar.gz
        sudo mv kyverno /usr/local/bin/

    - name: Validate Kubernetes Manifests
      run: |
        kyverno apply policies/ --resource kubernetes-manifests/
```

📌 **How This Works:**

- Runs when a Pull Request (PR) is raised.

- Checks Kubernetes manifests against Kyverno policies.
  
- If policy violations are found, it blocks the PR.

---

6. **Developer Workflow with Policy Enforcement**

✅ **Case 1: Policy Passed**

**DevOps Engineer raises a PR** with a corrected manifest:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-secure
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      securityContext:
        runAsNonRoot: true  # ✅ This follows the Kyverno policy
      containers:
      - name: nginx
        image: nginx
```

1. **CI/CD pipeline runs Kyverno checks.**
  
2. **Policy check passes** ✅.
   
3. **PR gets approved and merged into main.**
  
4. **Deployment proceeds to Kubernetes.**
   
🚨 **Case 2: Policy Failed**

1. **DevOps Engineer raises a PR** with a policy-violating manifest (runAsNonRoot: false).

2. **CI/CD pipeline runs Kyverno checks.**
   
3. **Kyverno detects the violation** and blocks the PR ❌.

4. **Engineer is notified** via GitHub Actions.
   
5. **Fix is required before merging.**

---

7. **Monitor and Audit Policy Violations**

Kyverno logs all policy violations.

Check violations:
```bash
kubectl get policyreport -A
```

Get details of failed policy enforcement:

```bash
kubectl describe policyreport -n default
```

🔹 **Use Grafana or Prometheus** to monitor policy violations in real-time.

---

8. **Summary**

✅ **Kyverno prevents non-compliant Kubernetes deployments.**

✅ **Policy checks run automatically in CI/CD pipelines.**

✅ **PRs are blocked if security rules are violated.**

✅ **Developers are notified when policies fail.**

✅ **Policies are stored as code for version control and auditing.**

This approach automates security enforcement in DevOps workflows while ensuring infrastructure compliance.

9. **Next Steps**
   
🚀 Extend policies for network security, RBAC, and compliance.

🔍 Integrate with monitoring tools to track violations.

🔄 Use GitOps (ArgoCD/FluxCD) for policy-driven deployments.
