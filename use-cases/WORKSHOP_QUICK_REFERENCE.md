# Kyverno Workshop - Quick Reference Guide

## Policy Scopes & Types

| Feature | Scope | Policy Type | When to Use |
|---------|-------|-------------|------------|
| **Validate** | Global or NS | `ClusterPolicy` or `Policy` | Enforce compliance rules |
| **Mutate** | Global or NS | `ClusterPolicy` or `Policy` | Auto-fix configurations |
| **Generate** | Global | `ClusterPolicy` only | Auto-create resources |

---

## ClusterPolicy vs Policy

### ClusterPolicy (Cluster-wide)
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: my-cluster-policy
# Applies to ALL namespaces (except excluded ones)
```

**Use when**: You need cluster-wide enforcement

### Policy (Namespace-scoped)
```yaml
apiVersion: kyverno.io/v1
kind: Policy
metadata:
  name: my-namespace-policy
  namespace: production  # Specific namespace
# Applies ONLY to this namespace
```

**Use when**: You need namespace-specific rules

---

## Key Concepts at a Glance

### Validation
```yaml
spec:
  validationFailureAction: audit  # audit: log only | enforce: block
  rules:
  - name: check-rule
    match:
      resources:
        kinds: [Pod, Deployment]
    validate:
      message: "Error message to user"
      pattern:
        spec:
          # Pattern to match - use "?*" for required fields
```

### Mutation
```yaml
spec:
  mutationFailureAction: audit
  rules:
  - name: mutate-rule
    match:
      resources:
        kinds: [Pod]
    mutate:
      patchStrategicMerge:
        spec:
          containers:
          - (name): "*"  # Match all containers
            image: "new-image:v1"
```

### Generation
```yaml
spec:
  rules:
  - name: generate-rule
    match:
      resources:
        kinds: [Namespace]
    generate:
      kind: NetworkPolicy
      name: auto-created-policy
      namespace: "{{ request.object.metadata.name }}"
      data:
        # Resource definition
```

---

## Common Patterns

### 1. Apply to Specific Namespaces

```yaml
match:
  resources:
    kinds: [Pod]
  namespaceSelector:
    matchLabels:
      policy-enabled: "true"
```

```bash
# Label namespace to enable policy
kubectl label namespace myns policy-enabled=true
```

### 2. Exclude System Namespaces

```yaml
match:
  resources:
    kinds: [Pod]
  excludeResources:
    namespaces:
    - kube-system
    - kube-public
    - kyverno
```

### 3. Match by Label

```yaml
match:
  resources:
    kinds: [Pod]
    selector:
      matchLabels:
        require-policy: "true"
```

### 4. Apply to Multiple Kinds

```yaml
match:
  resources:
    kinds:
    - Pod
    - Deployment
    - StatefulSet
    - DaemonSet
```

### 5. Use Variable Substitution

```yaml
generate:
  namespace: "{{ request.object.metadata.name }}"
  
# Other variables available:
# {{ serviceAccountName }}
# {{ request.object.metadata.labels.key }}
# {{ request.namespace }}
```

---

## Policy Enforcement Strategy

### Phase 1: Discovery & Planning
```bash
# Create policies in audit mode
validationFailureAction: audit

# Collect data for 1-2 weeks
kubectl get policyreport -A
```

### Phase 2: Testing in Dev
```bash
# Move to dev/staging
validationFailureAction: audit  # Still audit for feedback

# Let teams adapt and create exceptions
```

### Phase 3: Gradual Enforcement
```bash
# Switch to enforce in specific namespaces
validationFailureAction: enforce

# Monitor exceptions and violations
```

### Phase 4: Full Enforcement
```bash
# Apply to all production namespaces
# Keep audit logs for compliance
```

---

## Essential kubectl Commands

```bash
# List all policies
kubectl get clusterpolicy
kubectl get policy -A

# View policy details
kubectl describe clusterpolicy policy-name
kubectl get clusterpolicy policy-name -o yaml

# Check policy reports
kubectl get policyreport -A
kubectl describe policyreport -n namespace-name

# Check Kyverno logs
kubectl logs -n kyverno-system -l app=kyverno -f

# Test a specific policy
kubectl apply -f test-pod.yaml -n test-namespace

# Check what happened to the pod
kubectl describe pod pod-name -n test-namespace
kubectl get pod pod-name -n test-namespace -o yaml

# Delete a policy
kubectl delete clusterpolicy policy-name
kubectl delete policy policy-name -n namespace-name

# Patch a policy (change mode)
kubectl patch clusterpolicy policy-name \
  -p '{"spec":{"validationFailureAction":"enforce"}}' \
  --type merge
```

---

## Policy Debugging Checklist

- [ ] Kyverno pods are running: `kubectl get pods -n kyverno-system`
- [ ] Webhooks are configured: `kubectl get validatingwebhookconfigurations`
- [ ] Namespace has correct labels: `kubectl get ns --show-labels`
- [ ] Policy selectors match your pods: Check `namespaceSelector`, `selector`
- [ ] Resource kinds are listed: Check policy `kinds`
- [ ] Not in exclude list: Check `excludeResources`
- [ ] Check logs: `kubectl logs -n kyverno-system -l app=kyverno`
- [ ] Policy report shows violations: `kubectl get policyreport -A`

---

## 20 Most Common Use-Cases

### Validate (Security & Compliance)
1. ✅ Disallow running as root
2. ✅ Require image digest (not tags)
3. ✅ Restrict image registries
4. ✅ Require resource requests/limits
5. ✅ Enforce security context

### Mutate (Auto-Remediation)
6. ✅ Add security context
7. ✅ Inject service mesh sidecar
8. ✅ Add default labels
9. ✅ Add resource requests
10. ✅ Add monitoring annotations
11. ✅ Add registry credentials
12. ✅ Add pod priority
13. ✅ Add tolerations
14. ✅ Add affinity rules
15. ✅ Add init containers

### Generate (Auto-Creation)
16. ✅ Create NetworkPolicy for new NS
17. ✅ Create ResourceQuota for new NS
18. ✅ Create LimitRange for new NS
19. ✅ Clone secrets to new NS
20. ✅ Create RBAC role bindings

---

## Failure Actions Reference

| Mode | Validation | Mutation | Behavior |
|------|-----------|----------|----------|
| **audit** | Log violation | Log error | Resource created/modified, violation reported |
| **enforce** | Block request | Block request | Request denied, error returned to client |

---

## Variable Reference in Policies

```yaml
# Substitute runtime values
generate:
  namespace: "{{ request.object.metadata.name }}"
  data:
    metadata:
      annotations:
        created-ns: "{{ request.object.metadata.name }}"
        created-user: "{{ serviceAccountName }}"

# Available variables
{{ request.object }}              # Full object
{{ request.object.metadata.name }}
{{ request.object.metadata.labels.app }}
{{ request.namespace }}
{{ serviceAccountName }}
```

---

## Useful kubectl Aliases for Workshop

```bash
# Add to ~/.bashrc or ~/.zshrc
alias kgcp='kubectl get clusterpolicy'
alias kdcp='kubectl describe clusterpolicy'
alias kgp='kubectl get policy -A'
alias kgpr='kubectl get policyreport -A'
alias kkvno='kubectl logs -n kyverno-system -l app=kyverno'
alias kgns='kubectl get ns --show-labels'
```

---

## Common Mistakes to Avoid

❌ **Mistake**: Apply enforce policy without testing  
✅ **Fix**: Use audit mode for 1-2 weeks first

❌ **Mistake**: Policy applies to kube-system  
✅ **Fix**: Add excludeResources for system namespaces

❌ **Mistake**: Mutate policy breaks existing workloads  
✅ **Fix**: Test in dev first, use conditional mutation

❌ **Mistake**: Generate policy creates resources every time pod restarts  
✅ **Fix**: Use namespace or resource creation triggers

❌ **Mistake**: Too broad policies = high webhook latency  
✅ **Fix**: Use namespace selectors and exclude rules

---

## Workshop Agenda (4 Hours)

| Time | Topic | Activity |
|------|-------|----------|
| 0:00-0:30 | Intro | Concept overview, use-cases |
| 0:30-1:30 | Validate | Lab 1 - Create and test policies |
| 1:30-2:30 | Mutate | Lab 2 - Auto-remediation |
| 2:30-3:30 | Generate | Lab 3 - Auto-creation |
| 3:30-4:00 | Q&A & Cleanup | Discussion, troubleshooting |

---

## Resources

- **Kyverno Docs**: https://kyverno.io/docs/
- **Policy Library**: https://kyverno.io/policies/
- **Kyverno CLI**: https://kyverno.io/docs/kyverno-cli/
- **Kyverno Playground**: https://kyverno.io/policies/playground/

---

## Useful Links for Reference

- KubeCon talks on Kyverno
- Kyverno GitHub: https://github.com/kyverno/kyverno
- Slack Community: #kyverno on Kubernetes Slack

