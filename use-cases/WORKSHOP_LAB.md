# Kyverno Workshop - Practical Hands-On Lab

## Preparation

### Prerequisites
- Kubernetes cluster (1.16+)
- kubectl CLI configured
- Helm 3+
- Basic understanding of K8s manifests

### Setup

```bash
# 1. Create namespaces for testing
kubectl create namespace kyverno-system
kubectl create namespace kyverno-test
kubectl create namespace dev
kubectl create namespace prod

# 2. Install Kyverno
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno --namespace kyverno-system

# 3. Verify Kyverno is running
kubectl get pods -n kyverno-system
kubectl get crd | grep kyverno
```

---

## Lab 1: VALIDATE POLICIES

### Objective
Learn how to enforce compliance rules using validation policies.

### Scenario
Your organization requires:
- All containers must run as non-root
- All pods in production must have resource requests/limits
- Images must come from approved registries only

### Step 1.1: Create Namespace Labels

```bash
# Label namespaces for policies
kubectl label namespace kyverno-test require-resources=true
kubectl label namespace kyverno-test restrict-registries=true
kubectl label namespace prod require-resources=true env=production
```

### Step 1.2: Deploy Validation Policy - No Root User

```bash
# Apply policy from template
kubectl apply -f templates-chart/templates/cpol-val-disallow-root-user.yaml

# Verify policy is created
kubectl get clusterpolicy
```

### Step 1.3: Test the Policy - Validation Mode

Create a test pod that violates the policy:

```bash
cat << 'EOF' | kubectl apply -f - -n kyverno-test
apiVersion: v1
kind: Pod
metadata:
  name: test-root-pod
spec:
  containers:
  - name: app
    image: nginx:latest
    securityContext:
      runAsNonRoot: false  # VIOLATES POLICY
EOF
```

**Expected Result**: 
- In `audit` mode: Pod is created but policy violation is logged
- In `enforce` mode: Pod creation is DENIED with error message

### Step 1.4: Create Compliant Pod

```bash
cat << 'EOF' | kubectl apply -f - -n kyverno-test
apiVersion: v1
kind: Pod
metadata:
  name: test-compliant-pod
spec:
  containers:
  - name: app
    image: nginx:latest
    securityContext:
      runAsNonRoot: true  # COMPLIES
      runAsUser: 1000
EOF
```

**Expected Result**: Pod is created successfully

### Step 1.5: Deploy Image Registry Validation

```bash
# Deploy registry restriction policy
kubectl apply -f templates-chart/templates/cpol-val-restrict-registries.yaml

# Test with unapproved registry
cat << 'EOF' | kubectl apply -f - -n kyverno-test
apiVersion: v1
kind: Pod
metadata:
  name: test-untrusted-registry
spec:
  containers:
  - name: app
    image: docker.io/untrusted/app:v1  # From approved registry
EOF
```

### Step 1.6: Enforce Resources

```bash
# Deploy resource enforcement
kubectl apply -f templates-chart/templates/cpol-val-enforce-resources.yaml

# Test pod without resources
cat << 'EOF' | kubectl apply -f - -n kyverno-test
apiVersion: v1
kind: Pod
metadata:
  name: test-no-resources
spec:
  containers:
  - name: app
    image: nginx:latest
    # Missing resources: requests/limits
EOF
```

**Expected Result**: Creation fails or audit log shows violation

### Step 1.7: Monitor Policy Violations

```bash
# Check policy violations in audit logs
kubectl get policyreport -A
kubectl describe policyreport -n kyverno-test

# Check Kyverno controller logs
kubectl logs -n kyverno-system -l app=kyverno -f
```

---

## Lab 2: MUTATE POLICIES

### Objective
Learn how to automatically modify resources to enforce compliance.

### Scenario
Your organization needs to:
- Auto-add security context to all containers
- Inject Istio sidecar to microservices
- Add monitoring annotations for Prometheus

### Step 2.1: Prepare Namespace with Labels

```bash
# Enable auto-mutation features
kubectl label namespace kyverno-test auto-sec-context=true
kubectl label namespace kyverno-test istio-injection=enabled
kubectl label namespace kyverno-test prometheus-monitoring=enabled
```

### Step 2.2: Deploy Security Context Mutation

```bash
# Apply mutation policy
kubectl apply -f templates-chart/templates/cpol-mut-add-security-context.yaml

# Create pod without security context
cat << 'EOF' | kubectl apply -f - -n kyverno-test
apiVersion: v1
kind: Pod
metadata:
  name: test-mutation-sec-context
spec:
  containers:
  - name: app
    image: nginx:latest
    # No security context specified
EOF

# Check what Kyverno added
kubectl get pod test-mutation-sec-context -n kyverno-test -o yaml | grep -A 10 securityContext
```

**Expected Result**: Kyverno automatically adds security context with:
- `runAsNonRoot: true`
- `runAsUser: 1000`
- `readOnlyRootFilesystem: true`
- `capabilities: drop ALL`

### Step 2.3: Deploy Default Resource Requests

```bash
# Label namespace for auto-resources
kubectl label namespace dev auto-add-requests=true

# Apply mutation policy
kubectl apply -f templates-chart/templates/cpol-mut-add-default-resources.yaml

# Create deployment without resources
cat << 'EOF' | kubectl apply -f - -n dev
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-no-resources
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: nginx:latest
        # No resources specified
EOF

# Verify resources were added
kubectl get pod -n dev -o json | jq '.items[0].spec.containers[0].resources'
```

**Expected Result**: Pods now have default requests/limits:
```json
{
  "requests": {"memory": "128Mi", "cpu": "100m"},
  "limits": {"memory": "512Mi", "cpu": "500m"}
}
```

### Step 2.4: Deploy Monitoring Annotations

```bash
# Apply monitoring mutation
kubectl apply -f templates-chart/templates/cpol-mut-add-monitoring.yaml

# Create pod
cat << 'EOF' | kubectl apply -f - -n kyverno-test
apiVersion: v1
kind: Pod
metadata:
  name: test-mutation-monitoring
spec:
  containers:
  - name: app
    image: prometheus-app:latest
EOF

# Check annotations
kubectl get pod test-mutation-monitoring -n kyverno-test -o yaml | grep -A 5 annotations
```

**Expected Result**: Annotations added automatically:
```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"
  prometheus.io/scheme: "http"
```

### Step 2.5: Mutation with Background Processing

```bash
# Change policy mode to enforce (not just audit)
kubectl patch clusterpolicy add-resource-requests-defaults -p \
  '{"spec":{"mutationFailureAction":"enforce"}}' --type merge

# Redeploy a pod - it will be mutated automatically
kubectl delete deployment app-no-resources -n dev
# Redeploy again
```

---

## Lab 3: GENERATE POLICIES

### Objective
Learn how to auto-create resources based on triggers.

### Scenario
Your organization needs to:
- Auto-create network policies for new namespaces
- Auto-create resource quotas for new namespaces
- Clone registry secrets to all namespaces

### Step 3.1: Deploy NetworkPolicy Generation

```bash
# Apply generation policy
kubectl apply -f templates-chart/templates/cpol-gen-network-policy.yaml

# Create new namespace with label
kubectl create namespace test-gen-ns
kubectl label namespace test-gen-ns network-policy=enabled

# Verify NetworkPolicy was auto-created
kubectl get networkpolicy -n test-gen-ns
kubectl describe networkpolicy default-deny-ingress -n test-gen-ns
```

**Expected Result**: NetworkPolicy with default-deny ingress is created automatically.

### Step 3.2: Deploy Resource Quota Generation

```bash
# Apply quota generation policy
kubectl apply -f templates-chart/templates/cpol-gen-resource-quota.yaml

# Create dev namespace
kubectl create namespace test-dev-quota
kubectl label namespace test-dev-quota env=development

# Verify ResourceQuota was created
kubectl get resourcequota -n test-dev-quota
kubectl describe resourcequota dev-quota -n test-dev-quota
```

**Expected Result**: Dev ResourceQuota is auto-created with limits.

### Step 3.3: Deploy LimitRange Generation

```bash
# Apply LimitRange generation
kubectl apply -f templates-chart/templates/cpol-gen-limit-range.yaml

# Create another namespace
kubectl create namespace test-limits

# Verify LimitRange was created
kubectl get limitrange -n test-limits
kubectl describe limitrange default-limits -n test-limits
```

**Expected Result**: LimitRange with default container limits is created.

### Step 3.4: Test Generated Policies Impact

```bash
# Try to create pod without resources in quota-limited namespace
cat << 'EOF' | kubectl apply -f - -n test-dev-quota
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: app
    image: nginx:latest
EOF

# Check if defaults from LimitRange were applied
kubectl get pod test-pod -n test-dev-quota -o yaml | grep -A 5 resources
```

### Step 3.5: Deploy Secret Cloning (Optional)

```bash
# First create source secret in default namespace
cat << 'EOF' | kubectl apply -f - -n default
apiVersion: v1
kind: Secret
metadata:
  name: docker-credentials
type: kubernetes.io/dockercfg
data:
  .dockercfg: eyJkb2NrZXIuaW8iOnsidXNlcm5hbWUiOiJ1c2VyIiwicGFzc3dvcmQiOiJwYXNzIiwiYXV0aCI6ImIzQnlaWE56WTI5aWEzIn19Cg==
EOF

# Apply secret cloning policy
kubectl apply -f templates-chart/templates/cpol-gen-clone-secrets.yaml

# Create namespace with label
kubectl create namespace test-clone-secrets
kubectl label namespace test-clone-secrets clone-secrets=true

# Verify secret was cloned
kubectl get secret -n test-clone-secrets
kubectl describe secret docker-credentials -n test-clone-secrets
```

---

## Lab 4: Policy Management

### Step 4.1: List All Policies

```bash
# List all ClusterPolicies
kubectl get clusterpolicy
kubectl get clusterpolicy -o wide

# List all namespace-scoped Policies
kubectl get policy -A
```

### Step 4.2: Check Policy Status

```bash
# Detailed policy info
kubectl describe clusterpolicy disallow-root-user

# Check policy rule validation
kubectl get clusterpolicy disallow-root-user -o yaml
```

### Step 4.3: View Policy Reports

```bash
# List all policy reports
kubectl get policyreport -A

# Check violations in a specific namespace
kubectl describe policyreport -n kyverno-test

# Export policy report to JSON
kubectl get policyreport -n kyverno-test -o json
```

### Step 4.4: Audit Logs and Violations

```bash
# Check Kyverno webhook logs for audit violations
kubectl logs -n kyverno-system -l app=kyverno -f | grep -i "violation\|audit"

# Check policy reports for details
kubectl get policyreport -n kyverno-test -o yaml
```

### Step 4.5: Modify Policies

```bash
# Change from audit to enforce mode
kubectl patch clusterpolicy disallow-root-user \
  -p '{"spec":{"validationFailureAction":"enforce"}}' \
  --type merge

# Add/remove rules
kubectl patch clusterpolicy restrict-untrusted-registries \
  --type json \
  -p='[
    {
      "op": "add",
      "path": "/spec/rules/0/match/resources/kinds/-",
      "value": "Job"
    }
  ]'

# Disable a policy temporarily
kubectl patch clusterpolicy disallow-root-user \
  -p '{"spec":{"validationFailureAction":"audit"}}' \
  --type merge
```

### Step 4.6: Delete Policies

```bash
# Delete a single policy
kubectl delete clusterpolicy disallow-root-user

# Delete all policies in a namespace
kubectl delete policy -n kyverno-test --all

# Delete all policies cluster-wide
kubectl delete clusterpolicy --all
```

---

## Lab 5: Troubleshooting & Best Practices

### Step 5.1: Policy Not Being Applied

```bash
# Check if Kyverno webhook is running
kubectl get pods -n kyverno-system

# Check webhook configuration
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations

# Check if your namespace/pod matches the policy selectors
kubectl get namespace kyverno-test --show-labels
```

### Step 5.2: Policy Performance

```bash
# Monitor Kyverno performance
kubectl top pod -n kyverno-system

# Check webhook latency
kubectl logs -n kyverno-system -l app=kyverno | grep -i "latency\|duration"

# Check if rules are excluded properly
kubectl get clusterpolicy disallow-root-user -o yaml | grep -A 10 "excludeResources"
```

### Step 5.3: Debug Policy Matching

```bash
# Create a test pod and check detailed logs
kubectl run test-pod --image=nginx -n kyverno-test

# Check what policies were evaluated
kubectl logs -n kyverno-system -l app=kyverno | grep test-pod

# Describe the pod to see any events
kubectl describe pod test-pod -n kyverno-test
```

### Step 5.4: Best Practices

1. **Start with audit mode**: Deploy policies in audit mode first
   ```bash
   validationFailureAction: audit
   mutationFailureAction: audit
   ```

2. **Use namespace selectors**: Don't apply to all namespaces
   ```yaml
   namespaceSelector:
     matchLabels:
       policy-enabled: "true"
   ```

3. **Exclude system namespaces**:
   ```yaml
   excludeResources:
     namespaces:
     - kube-system
     - kube-public
     - kyverno
   ```

4. **Test policies before enforcement**
   ```bash
   # Keep in audit mode for 1-2 weeks
   kubectl logs -n kyverno-system | grep "audit"
   ```

5. **Monitor policy reports regularly**
   ```bash
   # Cron job to export reports
   kubectl get policyreport -A -o json > policy-reports.json
   ```

---

## Lab Complete! 🎉

### Cleanup

```bash
# Remove test resources
kubectl delete namespace kyverno-test dev prod test-gen-ns test-dev-quota test-limits test-clone-secrets

# (Optional) Remove Kyverno
helm uninstall kyverno -n kyverno-system
kubectl delete namespace kyverno-system
```

### What You Learned
- ✅ How Validate policies enforce compliance
- ✅ How Mutate policies auto-fix configurations  
- ✅ How Generate policies create resources automatically
- ✅ ClusterPolicy vs Policy (cluster vs namespace scope)
- ✅ audit vs enforce mode and when to use each
- ✅ How to monitor and troubleshoot policies

### Next Steps
1. Review the use-case documentation
2. Adapt policies to your organization's needs
3. Implement policies in staging first
4. Monitor policy reports and violations
5. Gradually enforce policies as teams adapt

