# Kyverno Workshop - Complete Index

## 📚 Documentation Structure

### 1. **Main Documentation** (Start here)
- [WORKSHOP_USE_CASES.md](WORKSHOP_USE_CASES.md) - Detailed use-cases with explanations
  - 5 Validate examples
  - 10 Mutate examples  
  - 5 Generate examples
  - Summary table

### 2. **Hands-On Lab** (Practical)
- [WORKSHOP_LAB.md](WORKSHOP_LAB.md) - Step-by-step laboratory exercises
  - Lab setup and prerequisites
  - Lab 1: Validate policies
  - Lab 2: Mutate policies
  - Lab 3: Generate policies
  - Lab 4: Policy management
  - Lab 5: Troubleshooting

### 3. **Quick Reference** (Cheat Sheet)
- [WORKSHOP_QUICK_REFERENCE.md](WORKSHOP_QUICK_REFERENCE.md) - Quick lookup guide
  - Policy types & scopes
  - Common patterns
  - kubectl commands
  - Debugging checklist
  - 20 most common use-cases

### 4. **Customization Guide** ⭐ (NEW!)
- [CUSTOMIZATION_GUIDE.md](CUSTOMIZATION_GUIDE.md) - How to customize ALL policies
  - Enable/disable policies
  - Change resource kinds checked/mutated (Pod, Deployment, StatefulSet, etc.)
  - Customize generated quotas, limits, secrets, and NetworkPolicies
  - Change resource values
  - Modify labels and annotations (add for mutate, require for validate)
  - Customize namespace selectors (or leave empty for cluster-wide)
  - Customize per environment (dev, staging, prod)
  - 14+ practical examples
  - values-custom.yaml templates

### 5. **Policy Templates** (YAML Files)

#### Validate Policies
- `cpol-val-disallow-root-user.yaml` - No root containers
- `cpol-val-require-labels.yaml` - Require standard labels
- `cpol-val-require-image-digest.yaml` - Image digest enforcement
- `cpol-val-enforce-resources.yaml` - Resource requests/limits
- `cpol-val-restrict-registries.yaml` - Approved registries only
- `pol-val-restrict-image-registries.yaml` - NS-scoped registry restriction

#### Mutate Policies
- `cpol-mut-add-default-labels.yaml` - Auto-add labels
- `cpol-mut-set-memory-requests.yaml` - Default memory requests
- `cpol-mut-add-security-context.yaml` - Restrictive security context
- `cpol-mut-inject-istio.yaml` - Service mesh sidecar injection
- `cpol-mut-add-default-resources.yaml` - Default CPU/memory
- `cpol-mut-add-monitoring.yaml` - Prometheus annotations
- `pol-mut-inject-sidecar.yaml` - NS-scoped sidecar injection

#### Generate Policies
- `cpol-gen-network-policy.yaml` - Auto-create NetworkPolicy
- `cpol-gen-clone-secrets.yaml` - Clone registry secrets
- `cpol-gen-resource-quota.yaml` - Auto-create ResourceQuota
- `cpol-gen-limit-range.yaml` - Auto-create LimitRange
- `cpol-gen-clone-secret-on-ns-creation.yaml` - Clone on NS creation
- `cpol-gen-default-network-policy.yaml` - Default deny ingress

---

## 🎯 Workshop Flow

### For Facilitators
1. Start with [WORKSHOP_USE_CASES.md](WORKSHOP_USE_CASES.md) - Explain concepts
2. Walk through [WORKSHOP_LAB.md](WORKSHOP_LAB.md) - Hands-on exercises
3. Keep [WORKSHOP_QUICK_REFERENCE.md](WORKSHOP_QUICK_REFERENCE.md) handy for debugging

### For Participants
1. Read [WORKSHOP_USE_CASES.md](WORKSHOP_USE_CASES.md) - Understand use-cases
2. Follow [WORKSHOP_LAB.md](WORKSHOP_LAB.md) - Do the labs
3. Reference [WORKSHOP_QUICK_REFERENCE.md](WORKSHOP_QUICK_REFERENCE.md) - While practicing
4. Read [CUSTOMIZATION_GUIDE.md](CUSTOMIZATION_GUIDE.md) - Learn how to customize ⭐
5. Copy policy templates from `/templates/` folder - Adapt to your needs

---

## ⚙️ Customization Quick Start

**All policies are now fully customizable via Helm values!**

1. **Copy the template**: `cp templates-chart/values-custom.yaml my-values.yaml`
2. **Edit for your needs**: 
   ```yaml
   policies:
     mutate:
       addDefaultLabels:
         enabled: true
         labels:
           team: "my-team"
           cost-center: "engineering"
   ```
3. **Deploy**: `helm upgrade kyverno ./templates-chart -f my-values.yaml`

See [CUSTOMIZATION_GUIDE.md](CUSTOMIZATION_GUIDE.md) for 10+ examples!

---

## 📋 Pre-Workshop Checklist

- [ ] Kubernetes cluster is running (1.16+)
- [ ] kubectl is configured and working
- [ ] Helm 3 is installed
- [ ] Docker credentials set up (for private registry labs)
- [ ] Participants have sudo/admin on cluster
- [ ] Network policies are not already enforced
- [ ] Kyverno is not already installed (or document existing config)

---

## ⏱️ Recommended Schedule (4 hours)

| Time | Duration | Topic | Resource |
|------|----------|-------|----------|
| 0:00 | 30 min | Intro & Concepts | WORKSHOP_USE_CASES.md |
| 0:30 | 45 min | Validate Policies Lab | WORKSHOP_LAB.md - Lab 1 |
| 1:15 | 45 min | Mutate Policies Lab | WORKSHOP_LAB.md - Lab 2 |
| 2:00 | 45 min | Generate Policies Lab | WORKSHOP_LAB.md - Lab 3 |
| 2:45 | 15 min | Break | - |
| 3:00 | 30 min | Management & Troubleshooting | WORKSHOP_LAB.md - Labs 4&5 |
| 3:30 | 30 min | Q&A & Hands-on Practice | WORKSHOP_QUICK_REFERENCE.md |

---

## 🔍 Quick Navigation by Use-Case

### I want to learn about...

#### Security
- Disallow root: [WORKSHOP_USE_CASES.md#11-disallow-running-as-root-user](WORKSHOP_USE_CASES.md)
- Add security context: [WORKSHOP_USE_CASES.md#22-add-security-context-to-all-containers](WORKSHOP_USE_CASES.md)
- Restrict registries: [WORKSHOP_USE_CASES.md#14-restrict-container-image-registries](WORKSHOP_USE_CASES.md)

#### Resource Management
- Enforce quotas: [WORKSHOP_USE_CASES.md#13-enforce-resource-requests-and-limits](WORKSHOP_USE_CASES.md)
- Auto-add resources: [WORKSHOP_USE_CASES.md#26-add-resource-requests-for-memory-and-cpu](WORKSHOP_USE_CASES.md)
- Create limits: [WORKSHOP_USE_CASES.md#35-create-limitrange-for-container-resource-defaults](WORKSHOP_USE_CASES.md)

#### Automation
- Mutate policies: [WORKSHOP_USE_CASES.md#2-mutate-policies-10-use-cases](WORKSHOP_USE_CASES.md)
- Generate policies: [WORKSHOP_USE_CASES.md#3-generate-policies-5-use-cases](WORKSHOP_USE_CASES.md)

#### Networking
- NetworkPolicy: [WORKSHOP_USE_CASES.md#31-auto-create-networkpolicy-for-new-namespaces](WORKSHOP_USE_CASES.md)

#### Service Mesh
- Istio sidecar: [WORKSHOP_USE_CASES.md#21-inject-istio-sidecar-proxy](WORKSHOP_USE_CASES.md)

---

## 🛠️ How to Use the Policy Templates

### Copy a template
```bash
kubectl apply -f templates-chart/templates/cpol-val-disallow-root-user.yaml
```

### Modify for your needs
```bash
# Edit the template
vi templates-chart/templates/cpol-val-disallow-root-user.yaml

# Apply your modified version
kubectl apply -f templates-chart/templates/cpol-val-disallow-root-user.yaml
```

### Test in audit mode first
```bash
# Check current mode
kubectl get clusterpolicy disallow-root-user -o yaml | grep validationFailureAction

# Change to enforce after testing
kubectl patch clusterpolicy disallow-root-user \
  -p '{"spec":{"validationFailureAction":"enforce"}}' \
  --type merge
```

---

## 📊 Policy Coverage by Category

### Security Policies
- ✅ Disallow root user (Validate)
- ✅ Add security context (Mutate)
- ✅ Require image digest (Validate)
- ✅ Restrict registries (Validate)

### Resource Management
- ✅ Enforce resources (Validate)
- ✅ Add default resources (Mutate)
- ✅ Create ResourceQuota (Generate)
- ✅ Create LimitRange (Generate)

### Networking
- ✅ Create NetworkPolicy (Generate)
- ✅ Clone secrets (Generate)

### Observability
- ✅ Add monitoring annotations (Mutate)

### Service Mesh
- ✅ Inject Istio sidecar (Mutate)

---

## 🎓 Learning Outcomes

After completing this workshop, participants will:

1. ✅ Understand Kyverno's three core features:
   - Validate: Enforce compliance
   - Mutate: Auto-remediate
   - Generate: Auto-create resources

2. ✅ Know when to use:
   - ClusterPolicy vs Policy
   - audit vs enforce modes
   - Namespace vs cluster-wide policies

3. ✅ Be able to:
   - Deploy and test policies
   - Monitor policy violations
   - Troubleshoot policy issues
   - Adapt policies to specific use-cases

4. ✅ Understand best practices:
   - Start with audit mode
   - Use namespace selectors
   - Exclude system namespaces
   - Monitor and iterate

---

## 📞 Support & Resources

### During Workshop
- Facilitator Q&A
- WORKSHOP_QUICK_REFERENCE.md for common issues
- Kyverno logs: `kubectl logs -n kyverno-system -l app=kyverno`

### After Workshop
- Review [WORKSHOP_QUICK_REFERENCE.md](WORKSHOP_QUICK_REFERENCE.md)
- Consult policy templates in `templates/` folder
- Check [WORKSHOP_LAB.md](WORKSHOP_LAB.md) troubleshooting section
- Visit Kyverno documentation: https://kyverno.io/docs/

---

## 📝 Notes Section

Use this space to take notes during the workshop:

```
Personal Notes:
- Key learnings:
- Questions to follow up:
- Policies to implement:
- Teams to communicate with:
```

---

## Version Info
- **Workshop Version**: 1.0
- **Kyverno Version**: 1.10+ (tested on 1.17)
- **Kubernetes Version**: 1.16+
- **Last Updated**: 2024

---

**Ready to start? Begin with [WORKSHOP_USE_CASES.md](WORKSHOP_USE_CASES.md)** 🚀
