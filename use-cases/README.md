# Kyverno Use Cases & Workshop Material

Welcome! This directory contains comprehensive workshop material for Kyverno, including:
- 20 production-ready use-cases
- Hands-on lab exercises
- Policy templates
- Quick reference guides

## 🚀 Quick Start

**New to this workshop?** Start here: [INDEX.md](INDEX.md)

**Want to see examples?** Read: [WORKSHOP_USE_CASES.md](WORKSHOP_USE_CASES.md)

**Ready to practice?** Follow: [WORKSHOP_LAB.md](WORKSHOP_LAB.md)

**Need quick answers?** Check: [WORKSHOP_QUICK_REFERENCE.md](WORKSHOP_QUICK_REFERENCE.md)

**Want to customize policies?** Read: [CUSTOMIZATION_GUIDE.md](CUSTOMIZATION_GUIDE.md) ⭐

**Quick start (5 min)?** Try: [QUICKSTART_CUSTOMIZATION.md](QUICKSTART_CUSTOMIZATION.md) ⚡

---

## 📚 Documentation

| Document | Purpose | Duration |
|----------|---------|----------|
| **INDEX.md** | Overview & navigation | 5 min read |
| **WORKSHOP_USE_CASES.md** | 20 production use-cases with explanations | 30 min read |
| **WORKSHOP_LAB.md** | 5 hands-on labs with step-by-step instructions | 3-4 hours to complete |
| **WORKSHOP_QUICK_REFERENCE.md** | Cheat sheet & quick lookup | Reference |
| **CUSTOMIZATION_GUIDE.md** | ⭐ How to customize ALL policies via Helm values | 20 min read |
| **README.md** | This file | - |

---

## ⚙️ Policy Customization (NEW!)

**All policies are fully customizable via Helm values!**

### What Can You Customize?

✅ **Enable/Disable** individual policies  
✅ **Change failure modes** (audit vs enforce)  
✅ **Modify labels** to add to pods (mutate) or require on pods (validate)  
✅ **Adjust resource kinds** checked/mutated (Pod, Deployment, StatefulSet, etc.)  
✅ **Adjust resource values** (CPU, memory)  
✅ **Change Istio version** for injection  
✅ **Customize security settings**  
✅ **Configure namespace selectors** (or leave empty for cluster-wide)  
✅ **Change approved registries** for image restriction  
✅ **Override quotas and limits**  
✅ **Add custom annotations**  
✅ **...and much more!**

### Quick Example

```bash
# Edit values for your organization
vi templates-chart/values.yaml

# Customize resource defaults
policies:
  mutate:
    addDefaultResources:
      resources:
        requests:
          cpu: "100m"      # Your value
          memory: "128Mi"  # Your value

# Deploy with your customizations
helm upgrade kyverno ./templates-chart
```

### See [CUSTOMIZATION_GUIDE.md](CUSTOMIZATION_GUIDE.md) for:
- **10+ detailed examples** of common customizations
- **Scenario-based configs** (dev, staging, prod)
- **Policy-by-policy guide** for all customizable options
- **Copy-paste templates** for values-custom.yaml

---

### Validate Policies (5 use-cases)
These policies **enforce compliance** by checking resources against rules.
⭐ **Fully customizable**: resource kinds, namespace selector, required labels, approved registries — all via values.yaml.

1. **Disallow Running as Root** - Prevent containers running as root user
2. **Require Image Digest** - Force images referenced by digest (sha256:...), not tags
3. **Enforce Resource Requests/Limits** - Require CPU and memory specifications
4. **Restrict Image Registries** - Only allow approved registries (gcr.io, docker.io, etc.)
5. **Require Labels** - Enforce mandatory labels (app, version, managed-by, etc.)

**Key Files**:
- `templates-chart/templates/cpol-val-disallow-root-user.yaml`
- `templates-chart/templates/cpol-val-require-image-digest.yaml`
- `templates-chart/templates/cpol-val-enforce-resources.yaml`
- `templates-chart/templates/cpol-val-restrict-registries.yaml`

### Mutate Policies (10 use-cases)
These policies **automatically fix** resources to meet compliance:

1. **Inject Istio Sidecar** - Auto-inject service mesh proxy
2. **Add Security Context** - Apply restrictive security settings
3. **Add Default Labels** - Tag resources for tracking & management
4. **Add Registry Credentials** - Inject imagePullSecrets
5. **Add Network Labels** - Tag for network policy targeting
6. **Add Default Resources** - Set CPU/memory requests & limits
7. **Add Prometheus Annotations** - Enable metrics collection
8. **Add Pod Priority** - Ensure critical pods aren't evicted
9. **Add Init Container** - Wait for dependencies (database, etc.)
10. **Add Audit Logging Sidecar** - Compliance logging

**Key Files**:
- `templates-chart/templates/cpol-mut-add-security-context.yaml`
- `templates-chart/templates/cpol-mut-inject-istio.yaml`
- `templates-chart/templates/cpol-mut-add-default-resources.yaml`
- `templates-chart/templates/cpol-mut-add-monitoring.yaml`

### Generate Policies (5 use-cases)
These policies **auto-create resources** based on triggers:
⭐ **Fully customizable**: trigger resource kinds, namespace selectors, generated resource names, quotas, limits, secrets, and NetworkPolicy settings via values.yaml.

1. **Auto-Create NetworkPolicy** - Default-deny network isolation
2. **Clone Registry Secrets** - Distribute credentials to new namespaces
3. **Create RBAC ServiceAccount** - Auto-setup service accounts
4. **Create ResourceQuota** - Limit namespace resource consumption
5. **Create LimitRange** - Set default container resource limits

**Key Files**:
- `templates-chart/templates/cpol-gen-network-policy.yaml`
- `templates-chart/templates/cpol-gen-clone-secrets.yaml`
- `templates-chart/templates/cpol-gen-resource-quota.yaml`
- `templates-chart/templates/cpol-gen-limit-range.yaml`

---

## 📂 Directory Structure

```
use-cases/
├── README.md                          # This file
├── INDEX.md                           # Complete navigation guide
├── WORKSHOP_USE_CASES.md              # 20 use-cases with explanations
├── WORKSHOP_LAB.md                    # 5 hands-on labs
├── WORKSHOP_QUICK_REFERENCE.md        # Quick lookup cheat sheet
│
└── templates-chart/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── _helpers.tpl
        ├── NOTES.txt
        │
        ├── cpol-val-*.yaml            # Validate policies
        ├── pol-val-*.yaml             # NS-scoped validate
        │
        ├── cpol-mut-*.yaml            # Mutate policies
        ├── pol-mut-*.yaml             # NS-scoped mutate
        │
        └── cpol-gen-*.yaml            # Generate policies
```

---

## 🎓 How to Use This Material

### For Learning
1. Read [WORKSHOP_USE_CASES.md](WORKSHOP_USE_CASES.md) to understand each use-case
2. Study the pain-point each policy solves
3. Review policy-type (ClusterPolicy vs Policy)
4. Check scope (cluster-wide vs namespace)

### For Practicing
1. Follow [WORKSHOP_LAB.md](WORKSHOP_LAB.md) step-by-step
2. Deploy policies in your cluster
3. Test with sample workloads
4. Monitor policy violations
5. Adapt policies to your needs

### For Reference
1. Use [WORKSHOP_QUICK_REFERENCE.md](WORKSHOP_QUICK_REFERENCE.md) during practice
2. Copy policy templates from `templates-chart/templates/`
3. Modify templates for your organization
4. Deploy in audit mode first, then enforce

---

## 🏃 5-Minute Getting Started

### 1. Understand the Core Concepts

**Validate**: Check compliance
```yaml
- Disallow root user? ✗ BLOCK
- Has resource limits? ✗ LOG VIOLATION
```

**Mutate**: Auto-fix configuration
```yaml
- Missing labels? ✓ ADD "managed-by: kyverno"
- No resources? ✓ ADD "requests: 100m"
```

**Generate**: Auto-create resources
```yaml
- New namespace created? ✓ CREATE "NetworkPolicy"
- New namespace created? ✓ CREATE "ResourceQuota"
```

### 2. Choose Your First Policy

**Start simple**:
- Validate: Disallow root user
- Mutate: Add default labels
- Generate: Create NetworkPolicy

### 3. Deploy & Test

```bash
# Copy a policy template
kubectl apply -f templates-chart/templates/cpol-val-disallow-root-user.yaml

# Test with a pod
kubectl run test-pod --image=nginx
# Check if it was allowed/blocked

# View violations
kubectl get policyreport -A
```

---

## 📊 Policy Comparison Table

| Feature | Validate | Mutate | Generate |
|---------|----------|--------|----------|
| **Purpose** | Enforce rules | Auto-fix config | Auto-create resources |
| **Action** | Block or log | Modify before create | Create on trigger |
| **Scope** | ClusterPolicy or Policy | ClusterPolicy or Policy | ClusterPolicy only |
| **Best for** | Compliance | Standardization | Automation |
| **Example** | "Disallow root" | "Add security context" | "Create NetworkPolicy" |

---

## 🔧 Key Concepts

### Failure Actions
- **audit**: Log violations but allow request (test mode)
- **enforce**: Block request if violates policy (production mode)

### Policy Types
- **ClusterPolicy**: Applies cluster-wide (all namespaces except excluded)
- **Policy**: Applies only in its namespace

### Common Selectors
- `namespaceSelector`: Apply to labeled namespaces only
- `selector`: Match pods/deployments with labels
- `excludeResources`: Skip specific namespaces, resources, etc.

---

## ⚡ Common Commands

```bash
# Install Kyverno
helm install kyverno kyverno/kyverno -n kyverno-system --create-namespace

# List all policies
kubectl get clusterpolicy
kubectl get policy -A

# View a policy
kubectl describe clusterpolicy disallow-root-user

# Check violations
kubectl get policyreport -A

# View logs
kubectl logs -n kyverno-system -l app=kyverno -f

# Test a policy
kubectl apply -f test-pod.yaml -n test-namespace

# Modify policy (change audit to enforce)
kubectl patch clusterpolicy policy-name \
  -p '{"spec":{"validationFailureAction":"enforce"}}' \
  --type merge
```

---

## 🎯 Workshop Schedule (4 Hours)

| Time | Topic | Files |
|------|-------|-------|
| 0:00-0:30 | Intro & Concepts | WORKSHOP_USE_CASES.md (Intro) |
| 0:30-1:15 | Validate Lab | WORKSHOP_LAB.md (Lab 1) |
| 1:15-2:00 | Mutate Lab | WORKSHOP_LAB.md (Lab 2) |
| 2:00-2:45 | Generate Lab | WORKSHOP_LAB.md (Lab 3) |
| 2:45-3:00 | Break | - |
| 3:00-3:30 | Management | WORKSHOP_LAB.md (Labs 4-5) |
| 3:30-4:00 | Q&A & Practice | WORKSHOP_QUICK_REFERENCE.md |

---

## ✅ Pre-Workshop Checklist

Before starting the workshop, ensure:
- [ ] Kubernetes cluster is running (1.16+)
- [ ] kubectl is configured
- [ ] Helm 3 is installed
- [ ] Enough permissions to create ClusterPolicies
- [ ] Kyverno is installed (or will be installed as part of lab)
- [ ] Network connectivity is stable

---

## 📖 Next Steps

**Start here**: [INDEX.md](INDEX.md) - Complete navigation guide

**Read first**: [WORKSHOP_USE_CASES.md](WORKSHOP_USE_CASES.md) - Understand use-cases

**Do this**: [WORKSHOP_LAB.md](WORKSHOP_LAB.md) - Follow hands-on labs

**Keep nearby**: [WORKSHOP_QUICK_REFERENCE.md](WORKSHOP_QUICK_REFERENCE.md) - Quick lookup

---

## 🤝 Contributing

Have a new use-case? Submit it!

Template:
1. Name: Clear use-case name
2. Pain Point: What problem does it solve?
3. Scope: Cluster or namespace?
4. Policy Type: ClusterPolicy or Policy?
5. Example YAML: Runnable template

---

## 📝 License & Attribution

These materials are provided as-is for educational and training purposes.

---

**Ready to start?** ➜ [Go to INDEX.md](INDEX.md)
