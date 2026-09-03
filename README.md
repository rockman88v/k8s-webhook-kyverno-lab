# Kyverno Webhook Lab - Complete Workshop Kit

This repository contains a comprehensive workshop kit for learning and implementing Kyverno, the Kubernetes policy engine.

## 🎯 Quick Navigation

| Purpose | Resource | Time |
|---------|----------|------|
| **Start Here** | [use-cases/INDEX.md](use-cases/INDEX.md) | 5 min |
| **Learn Concepts** | [use-cases/WORKSHOP_USE_CASES.md](use-cases/WORKSHOP_USE_CASES.md) | 30 min |
| **Hands-On Labs** | [use-cases/WORKSHOP_LAB.md](use-cases/WORKSHOP_LAB.md) | 3-4 hours |
| **Quick Reference** | [use-cases/WORKSHOP_QUICK_REFERENCE.md](use-cases/WORKSHOP_QUICK_REFERENCE.md) | Reference |
| **Customize Policies** ⭐ | [use-cases/CUSTOMIZATION_GUIDE.md](use-cases/CUSTOMIZATION_GUIDE.md) | 20 min |
| **Summary** | [use-cases/SUMMARY.md](use-cases/SUMMARY.md) | 10 min |

---

## 📦 What's Inside

### Documentation (5 comprehensive guides)
- **INDEX.md** - Complete navigation & structure overview
- **WORKSHOP_USE_CASES.md** - 20 production-ready use-cases with explanations
- **WORKSHOP_LAB.md** - 5 hands-on labs with step-by-step instructions
- **WORKSHOP_QUICK_REFERENCE.md** - Cheat sheet & quick lookup guide
- **SUMMARY.md** - Completion report & implementation roadmap

### Policy Templates (24 YAML files)
- **5 Validate Policies** - Enforce security & compliance rules
- **10 Mutate Policies** - Auto-remediate & standardize configs
- **5 Generate Policies** - Auto-create resources
- **Helm Chart** - Ready to deploy via Helm

### Installation Package
- **kyverno-installation/** - Helm chart for Kyverno setup

---

## 📚 Directory Structure

```
k8s-webhook-kyverno-lab/
├── README.md                          # This file
├── kyverno-installation/              # Kyverno installation chart
│   ├── Chart.yaml
│   ├── values.yaml
│   └── values-custom.yaml
│
└── use-cases/                         # Workshop materials
    ├── INDEX.md                       # Navigation guide
    ├── README.md                      # Overview
    ├── SUMMARY.md                     # Completion report
    ├── WORKSHOP_USE_CASES.md          # 20 use-cases
    ├── WORKSHOP_LAB.md                # Hands-on labs
    ├── WORKSHOP_QUICK_REFERENCE.md    # Cheat sheet
    │
    └── templates-chart/               # Policy templates
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── cpol-val-*.yaml        # Validate policies
            ├── cpol-mut-*.yaml        # Mutate policies
            ├── cpol-gen-*.yaml        # Generate policies
            └── ... (24 YAML files total)
```

---

## 🚀 Quick Start (5 minutes)

### 1. Understand What You'll Learn

This workshop covers Kyverno's **three core features**:

- **Validate** (5 use-cases) - Enforce compliance & security rules
- **Mutate** (10 use-cases) - Automatically fix & standardize configurations
- **Generate** (5 use-cases) - Automatically create resources

### 2. Choose Your Path

**📖 Learn First** (Recommended for beginners)
1. Read [use-cases/WORKSHOP_USE_CASES.md](use-cases/WORKSHOP_USE_CASES.md)
2. Understand the 20 real-world use-cases
3. Review policy types & scopes

**🎓 Learn by Doing** (For hands-on learners)
1. Jump to [use-cases/WORKSHOP_LAB.md](use-cases/WORKSHOP_LAB.md)
2. Follow 5 progressive labs
3. Test policies in real K8s cluster

**⚡ Quick Start** (For experienced users)
1. Grab [use-cases/WORKSHOP_QUICK_REFERENCE.md](use-cases/WORKSHOP_QUICK_REFERENCE.md)
2. Copy policy templates from `use-cases/templates-chart/templates/`
3. Adapt and deploy in your cluster

### 3. Next Steps
```bash
# Open the index
cat use-cases/INDEX.md

# Or start with use-cases
cat use-cases/WORKSHOP_USE_CASES.md
```

---

## 🎯 Use-Cases at a Glance

### Validate: Enforce Compliance (5 examples)
1. Disallow running as root user
2. Require image digest (not tags)
3. Enforce resource requests/limits
4. Restrict untrusted registries
5. Require Pod Disruption Budget

### Mutate: Auto-Remediation (10 examples)
1. Inject Istio sidecar proxy
2. Add restrictive security context
3. Add default labels
4. Add registry credentials
5. Add network policy labels
6. Add default resources (CPU/memory)
7. Add Prometheus monitoring annotations
8. Add pod priority class
9. Add init containers
10. Add audit logging sidecar

### Generate: Auto-Creation (5 examples)
1. Auto-create NetworkPolicy for new namespaces
2. Clone registry secrets to new namespaces
3. Create RBAC ServiceAccount on namespace creation
4. Create ResourceQuota for new namespaces
5. Create LimitRange for container defaults

---

## 🎓 Workshop Format

**Duration**: 4 hours (full day) or split into 2-hour sessions

**Format**: 
- 30 min: Concepts & use-cases
- 2 hours: Hands-on labs
- 1 hour: Q&A & practice
- 30 min: Implementation planning

**Audience**: 
- DevOps engineers
- Platform teams
- Security teams
- Kubernetes operators

---

## 📋 Prerequisites

- Kubernetes 1.16+
- kubectl configured
- Helm 3+
- Basic K8s understanding (Pods, Deployments, Namespaces)
- Linux/Mac terminal access

---

## 💡 Key Concepts

### Policy Scopes
- **ClusterPolicy** - Applies cluster-wide (all namespaces except excluded)
- **Policy** - Namespace-scoped (specific namespace only)

### Failure Actions
- **audit** - Log violations, allow resource (testing/monitoring)
- **enforce** - Block violating resource (enforcement mode)

### Policy Types
- **Validate** - Check compliance, block/log violations
- **Mutate** - Modify resources before creation
- **Generate** - Auto-create resources on trigger

---

## 📖 Documentation

All documentation is in [use-cases/](use-cases/) directory:

| Document | Purpose | Read Time |
|----------|---------|-----------|
| INDEX.md | Navigation & overview | 5 min |
| README.md | Getting started | 10 min |
| WORKSHOP_USE_CASES.md | Detailed use-cases | 30 min |
| WORKSHOP_LAB.md | Hands-on exercises | 3-4 hours |
| WORKSHOP_QUICK_REFERENCE.md | Cheat sheet | Reference |
| SUMMARY.md | Completion report | 10 min |

---

## 🛠️ Policy Templates

All policy YAML templates are in [use-cases/templates-chart/templates/](use-cases/templates-chart/templates/)

**Deploy a policy:**
```bash
kubectl apply -f use-cases/templates-chart/templates/cpol-val-disallow-root-user.yaml
```

**Modify for your needs:**
```bash
# Copy template
cp use-cases/templates-chart/templates/cpol-val-disallow-root-user.yaml custom-policy.yaml

# Edit
vi custom-policy.yaml

# Deploy
kubectl apply -f custom-policy.yaml
```

---

## 🔧 Installation

### Install Kyverno

```bash
# Add Helm repo
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

# Install (cluster-wide)
helm install kyverno kyverno/kyverno \
  -n kyverno-system \
  --create-namespace

# Or use our Helm chart
helm install kyverno kyverno-installation \
  -n kyverno-system \
  --create-namespace
```

### Verify Installation

```bash
kubectl get pods -n kyverno-system
kubectl get crd | grep kyverno
```

---

## ✅ Next Steps

1. **Read Documentation**
   ```bash
   open use-cases/INDEX.md          # Start here
   open use-cases/WORKSHOP_USE_CASES.md
   ```

2. **Set Up Lab Environment**
   ```bash
   # Follow prerequisites in WORKSHOP_LAB.md
   # Install Kyverno
   # Prepare test namespaces
   ```

3. **Follow Hands-On Labs**
   ```bash
   # Start with Lab 1: Validate Policies
   # Then: Lab 2: Mutate Policies
   # Finally: Lab 3: Generate Policies
   # Plus: Lab 4-5: Management & Troubleshooting
   ```

4. **Implement in Your Cluster**
   - Copy policy templates
   - Adapt for your environment
   - Start with audit mode
   - Gradually move to enforce
   - Monitor and iterate

---

## 📞 Support & Resources

**In This Repository**
- Full documentation in [use-cases/](use-cases/)
- All policy templates ready to use
- Lab exercises for practice
- Quick reference guide

**External Resources**
- [Kyverno Official Docs](https://kyverno.io/docs/)
- [Kyverno Policy Library](https://kyverno.io/policies/)
- [Kyverno GitHub](https://github.com/kyverno/kyverno)
- [Kubernetes Slack #kyverno](https://kubernetes.slack.com/)

---

## 🎉 Ready to Start?

**Begin here:** → [use-cases/INDEX.md](use-cases/INDEX.md)
