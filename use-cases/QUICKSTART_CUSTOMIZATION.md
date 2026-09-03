# 🚀 Policy Customization - Quick Start Guide

## What Is This?

All Kyverno policies in this workshop are now **100% customizable via Helm values**.

No more editing YAML files! Just customize values and deploy.

---

## ⚡ 5-Minute Quick Start

### Step 1: Copy the template
```bash
cd use-cases/templates-chart

# Copy the custom values template
cp values-custom.yaml values-myorg.yaml
```

### Step 2: Choose an example and uncomment
```bash
# Edit and uncomment ONE of the 15 examples
vi values-myorg.yaml

# Examples include:
# - EXAMPLE 1: Development environment (permissive)
# - EXAMPLE 2: Production environment (strict)
# - EXAMPLE 3: Custom labels for organization
# - EXAMPLE 4: Custom resource defaults per env
# - EXAMPLE 5: Different Istio versions
# - ... and 5 more
```

### Step 3: Deploy
```bash
# Deploy with your custom values
helm upgrade kyverno . -f values-myorg.yaml
```

### Step 4: Verify
```bash
# Check values were applied
helm get values kyverno

# Check policy is deployed
kubectl get clusterpolicy

# Test with a sample pod
kubectl run test --image=nginx -n default
```

Done! 🎉

---

## 📚 What Can You Customize?

✅ Enable/disable individual policies  
✅ Change failure modes (audit → enforce)  
✅ Add custom labels to pods  
✅ Set resource defaults (CPU, memory)  
✅ Change Istio version for injection  
✅ Customize security settings  
✅ Override Prometheus scrape config  
✅ Set different values per environment  
✅ Exclude specific namespaces/containers  
✅ And much more!

---

## 📖 Learn More

**New to customization?**  
→ Read: [CUSTOMIZATION_GUIDE.md](../CUSTOMIZATION_GUIDE.md)

**Want detailed examples?**  
→ See: [values-custom.yaml](values-custom.yaml) (15 examples)

**Need reference docs?**  
→ Check: [../INDEX.md](../INDEX.md)

---

## 🎯 Common Use Cases

### Case 1: Add Your Company Labels
```yaml
# values-myorg.yaml
policies:
  mutate:
    addDefaultLabels:
      enabled: true
      labels:
        managed-by: "kyverno"
        team: "your-team-name"
      additionalLabels:
        cost-center: "your-cost-center"
        owner: "your-email@company.com"
```

Deploy:
```bash
helm upgrade kyverno . -f values-myorg.yaml
```

### Case 2: Production-Grade Resources
```yaml
# values-prod.yaml
policies:
  mutate:
    addDefaultResources:
      enabled: true
      failureAction: enforce  # Block if can't apply
      resources:
        requests:
          cpu: "500m"      # Half a CPU
          memory: "512Mi"  # 512MB
        limits:
          cpu: "2000m"     # 2 CPUs max
          memory: "2Gi"    # 2GB max
```

Deploy:
```bash
helm upgrade kyverno . -f values-prod.yaml
```

### Case 3: Strict Security Only
```yaml
# values-security.yaml
policies:
  mutate:
    addSecurityContext:
      enabled: true
      failureAction: enforce
    addDefaultLabels:
      enabled: false
    addDefaultResources:
      enabled: false
```

Deploy:
```bash
helm upgrade kyverno . -f values-security.yaml
```

---

## 🔍 How It Works

**Before customization** (hard-coded):
```yaml
# Old cpol-mut-add-default-labels.yaml
metadata:
  labels:
    managed-by: kyverno          # Fixed!
    app: "{{ request.namespace }}"
```

**After customization** (Helm templating):
```yaml
# New cpol-mut-add-default-labels.yaml
metadata:
  labels:
    {{- range $key, $value := .Values.policies.mutate.addDefaultLabels.labels }}
    {{ $key }}: "{{ tpl $value . }}"  # From values!
    {{- end }}
```

**Your values.yaml**:
```yaml
policies:
  mutate:
    addDefaultLabels:
      labels:
        managed-by: "kyverno"
        team: "my-team"
        cost-center: "engineering"
```

---

## 📁 Files You'll Use

```
use-cases/templates-chart/
├── values.yaml                    # ← Main config (comprehensive)
├── values-custom.yaml             # ← Template with 15 examples
│
└── templates/
    ├── cpol-mut-add-default-labels.yaml       # ← Templated
    ├── cpol-mut-add-security-context.yaml     # ← Templated
    ├── cpol-mut-inject-istio.yaml            # ← Templated
    ├── cpol-mut-add-default-resources.yaml    # ← Templated
    ├── cpol-mut-add-monitoring.yaml          # ← Templated
    └── cpol-mut-set-memory-requests.yaml     # ← Templated
```

---

## ❓ FAQ

### Q: Do I need to edit the YAML policy files?
**A:** No! Just customize values.yaml or values-custom.yaml

### Q: Can I have different settings for dev vs prod?
**A:** Yes! Create values-dev.yaml and values-prod.yaml, then:
```bash
# Deploy to dev
helm upgrade kyverno . -f values-dev.yaml

# Deploy to prod
helm upgrade kyverno . -f values-prod.yaml
```

### Q: What if I make a mistake?
**A:** Easy! Just revert:
```bash
# Deploy with defaults again
helm upgrade kyverno .
```

### Q: Can I customize via command line?
**A:** Yes!
```bash
helm upgrade kyverno . \
  --set policies.mutate.addDefaultLabels.enabled=false \
  --set policies.mutate.addDefaultResources.resources.requests.cpu=100m
```

### Q: How do I see what values are applied?
**A:** Run:
```bash
helm get values kyverno
```

---

## 🆘 Troubleshooting

### Policy didn't apply?
```bash
# Check if policy is created
kubectl get clusterpolicy

# Check policy details
kubectl describe clusterpolicy add-default-labels

# Check Kyverno logs
kubectl logs -n kyverno-system -l app=kyverno
```

### Wrong values being used?
```bash
# See current values
helm get values kyverno

# See full manifest (with rendered values)
helm get manifest kyverno | grep -A 20 "name: add-default-labels"
```

### Need to reset?
```bash
# Deploy with default values
helm upgrade kyverno .
```

---

## 🚦 Recommended Flow

1. **Learn** (20 min)
   - Read [CUSTOMIZATION_GUIDE.md](../CUSTOMIZATION_GUIDE.md)

2. **Start Simple** (5 min)
   - Copy values-custom.yaml
   - Uncomment one example
   - Deploy

3. **Experiment** (30 min)
   - Try different customizations
   - Test in dev environment
   - Monitor policy violations

4. **Finalize** (15 min)
   - Create values-prod.yaml with final settings
   - Document your customizations
   - Deploy to production

5. **Monitor** (Ongoing)
   - Check policy violations: `kubectl get policyreport -A`
   - Adjust settings as needed
   - Keep values in Git for version control

---

## 💡 Pro Tips

1. **Version control your values**
   ```bash
   git add values-myorg.yaml
   git commit -m "Customized policies for our org"
   ```

2. **Start with audit mode**
   ```yaml
   failureAction: audit  # Log violations, don't block
   ```
   Then switch to enforce after testing:
   ```yaml
   failureAction: enforce  # Block policy violations
   ```

3. **Test before deploying to prod**
   ```bash
   # Deploy to dev first
   helm upgrade kyverno . -f values-dev.yaml
   
   # Test with sample workloads
   kubectl run test --image=nginx -n test-ns
   
   # Check violations
   kubectl get policyreport -n test-ns
   ```

4. **Keep defaults commented**
   ```yaml
   # Original: managed-by: "kyverno"
   managed-by: "our-org"  # Our customization
   ```

5. **Use namespaceSelector to apply selectively**
   ```yaml
   namespaceSelector:
     matchLabels:
       enforce-policy: "true"
   ```
   Then only label namespaces where you want the policy:
   ```bash
   kubectl label ns production enforce-policy=true
   ```

---

## 🎓 Next Steps

1. ✅ You've read this quick start
2. 📖 Read [CUSTOMIZATION_GUIDE.md](../CUSTOMIZATION_GUIDE.md) for details
3. 📋 Copy and customize values-custom.yaml
4. 🚀 Deploy: `helm upgrade kyverno . -f values-myorg.yaml`
5. ✔️ Verify: `kubectl get clusterpolicy`
6. 🧪 Test with sample workloads
7. 📊 Monitor policy violations
8. 🔧 Iterate and improve

---

**Ready to customize?** Let's go! 🚀

Start with:
```bash
cd use-cases/templates-chart
cp values-custom.yaml values-myorg.yaml
vi values-myorg.yaml  # Uncomment an example
helm upgrade kyverno . -f values-myorg.yaml
```

Then read [CUSTOMIZATION_GUIDE.md](../CUSTOMIZATION_GUIDE.md) for all options!
