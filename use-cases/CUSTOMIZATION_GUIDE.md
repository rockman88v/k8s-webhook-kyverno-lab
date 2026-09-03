# Kyverno Policy Customization Guide

## Overview

All Kyverno policies in this workshop kit are **fully customizable via Helm values**. You can:
- ✅ Enable/disable individual policies
- ✅ Customize policy behavior and settings
- ✅ Change failure modes (audit vs enforce)
- ✅ Modify namespace selectors and exclusions
- ✅ Adjust resource kinds checked/mutated (Pod, Deployment, StatefulSet, etc.)
- ✅ Adjust resource values, labels, required labels, approved registries, etc.

> **Status**: All 5 **Validate** policies and 6 **Mutate** policies are fully parametrized (no hardcoded values). Generate policies already expose their key settings via values.yaml.

## How to Customize

### Method 1: Edit values.yaml
```bash
# Edit the main values file
vi use-cases/templates-chart/values.yaml

# Then redeploy
helm upgrade kyverno ./use-cases/templates-chart
```

### Method 2: Use values-custom.yaml (Recommended for Prod)
```bash
# Copy custom values template
cp use-cases/templates-chart/values-custom.yaml values-prod.yaml

# Edit your custom values
vi values-prod.yaml

# Deploy with custom values
helm upgrade kyverno ./use-cases/templates-chart -f values-prod.yaml
```

### Method 3: Pass values from command line
```bash
helm upgrade kyverno ./use-cases/templates-chart \
  --set policies.mutate.addDefaultLabels.enabled=false \
  --set policies.mutate.addDefaultResources.resources.requests.cpu=50m
```

---

## Policy-by-Policy Customization

### VALIDATE POLICIES

All 5 validate policies now support customizing **resource kinds**, **namespace selector**, and **rule-specific parameters** — nothing is hardcoded anymore.

#### 1. Disallow Root User (`disallowRoot`)

**Purpose**: Blocks containers running as root

**Location in values.yaml**:
```yaml
policies:
  validate:
    disallowRoot:
```

**Customizable Options**:

```yaml
disallowRoot:
  enabled: true
  failureAction: audit          # audit | enforce
  
  # Resource kinds this rule matches (add/remove freely)
  resourceKinds:
    - Pod
    - Deployment
  
  # Leave matchLabels: {} for cluster-wide (no namespace restriction)
  namespaceSelector:
    matchLabels:
      enforce-security: "true"
```

**Example: Apply cluster-wide, no namespace restriction**
```yaml
disallowRoot:
  enabled: true
  resourceKinds:
    - Pod
  namespaceSelector:
    matchLabels: {}   # empty = cluster-wide
```

---

#### 2. Require Image Digest (`requireImageDigest`)

**Purpose**: Forces images to be pinned by digest instead of mutable tags

**Customizable Options**:

```yaml
requireImageDigest:
  enabled: true
  failureAction: audit
  resourceKinds:
    - Pod
    - Deployment
    - StatefulSet
    - DaemonSet
  namespaceSelector:
    matchLabels:
      require-image-digest: "true"
```

**Example: Only enforce for Deployments/StatefulSets**
```yaml
requireImageDigest:
  enabled: true
  resourceKinds:
    - Deployment
    - StatefulSet
  namespaceSelector:
    matchLabels:
      require-image-digest: "true"
```

---

#### 3. Enforce Resources (`enforceResources`)

**Purpose**: Requires CPU/memory requests and limits on every container

**Customizable Options**:

```yaml
enforceResources:
  enabled: true
  failureAction: audit
  resourceKinds:
    - Pod
    - Deployment
    - StatefulSet
    - DaemonSet
  namespaceSelector:
    matchLabels:
      require-resources: "true"
```

**Example: Apply only to Pod and Job**
```yaml
enforceResources:
  enabled: true
  resourceKinds:
    - Pod
    - Job
  namespaceSelector:
    matchLabels:
      require-resources: "true"
```

---

#### 4. Restrict Registries (`restrictRegistries`)

**Purpose**: Blocks images from unapproved registries

**Customizable Options**:

```yaml
restrictRegistries:
  enabled: true
  failureAction: enforce
  resourceKinds:
    - Pod
    - Deployment
    - StatefulSet
    - DaemonSet
    - Job
    - CronJob
  approvedRegistries:
    - "gcr.io"
    - "docker.io"
    - "quay.io"
  namespaceSelector:
    matchLabels: {}   # empty = cluster-wide
```

**Example: Only allow your private registry**
```yaml
restrictRegistries:
  enabled: true
  approvedRegistries:
    - "myregistry.company.com"
  resourceKinds:
    - Pod
    - Deployment
```

---

#### 5. Require Labels (`requireLabels`)

**Purpose**: Enforces mandatory labels on resources

**Customizable Options**:

```yaml
requireLabels:
  enabled: true
  failureAction: audit
  resourceKinds:
    - Pod
  requiredLabels:
    - app
    - version
    - managed-by
  namespaceSelector:
    matchLabels: {}   # empty = cluster-wide
```

**Example: Add your own required labels**
```yaml
requireLabels:
  enabled: true
  resourceKinds:
    - Pod
    - Deployment
  requiredLabels:
    - app
    - team
    - cost-center
  namespaceSelector:
    matchLabels:
      require-labels: "true"
```

---

### MUTATE POLICIES

#### 1. Add Default Labels (`addDefaultLabels`)

**Purpose**: Auto-adds labels to all pods for tracking and management

**Location in values.yaml**:
```yaml
policies:
  mutate:
    addDefaultLabels:
```

**Customizable Options**:

```yaml
addDefaultLabels:
  # Enable or disable this policy
  enabled: true
  
  # When mutation fails, log (audit) or block (enforce)
  failureAction: audit  # options: audit | enforce
  
  # Only apply to namespaces with these labels
  namespaceSelector:
    matchLabels:
      auto-label: "true"
  
  # Labels to add to every pod
  labels:
    managed-by: "kyverno"                    # Static label
    environment: "{{ request.namespace }}"   # Dynamic: pod's namespace
    created-by: "{{ serviceAccountName }}"   # Dynamic: service account
  
  # Additional custom labels (add your own here)
  additionalLabels:
    team: "platform"
    cost-center: "engineering"
```

**Example 1: Add team-specific labels**
```yaml
addDefaultLabels:
  enabled: true
  labels:
    managed-by: "kyverno"
    team: "backend-team"
    version: "v1"
  additionalLabels:
    owner: "john-doe@company.com"
```

**Example 2: Apply to specific namespaces only**
```yaml
addDefaultLabels:
  enabled: true
  namespaceSelector:
    matchLabels:
      auto-label: "true"  # Only namespaces with this label
```

**To enable**: `kubectl label namespace myns auto-label=true`

---

#### 2. Add Security Context (`addSecurityContext`)

**Purpose**: Auto-injects restrictive security settings to harden containers

**Location in values.yaml**:
```yaml
policies:
  mutate:
    addSecurityContext:
```

**Customizable Options**:

```yaml
addSecurityContext:
  enabled: true
  failureAction: audit
  
  # Apply to labeled namespaces
  namespaceSelector:
    matchLabels:
      auto-sec-context: "true"
  
  # Security constraints to apply
  securityContext:
    runAsNonRoot: true        # Don't run as root
    runAsUser: 1000           # Run as UID 1000
    readOnlyRootFilesystem: true  # Read-only root
    allowPrivilegeEscalation: false  # No privilege escalation
    capabilities:
      drop:
        - ALL                 # Drop all capabilities
  
  # Containers to exclude (e.g., if sidecar needs special privileges)
  excludeContainers:
    - istio-proxy
    - special-sidecar
```

**Example 1: Strict security**
```yaml
addSecurityContext:
  enabled: true
  failureAction: enforce  # Block if can't apply
  securityContext:
    runAsNonRoot: true
    runAsUser: 65534  # Nobody user
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
    capabilities:
      drop:
        - ALL
```

**Example 2: Relax for specific containers**
```yaml
addSecurityContext:
  enabled: true
  excludeContainers:
    - postgres   # Database needs write access
    - cache      # Cache needs special capabilities
```

---

#### 3. Inject Istio Sidecar (`injectIstioSidecar`)

**Purpose**: Auto-injects service mesh proxy for traffic management

**Location in values.yaml**:
```yaml
policies:
  mutate:
    injectIstioSidecar:
```

**Customizable Options**:

```yaml
injectIstioSidecar:
  enabled: true
  failureAction: audit
  
  # Apply to labeled namespaces
  namespaceSelector:
    matchLabels:
      istio-injection: "enabled"
  
  # Istio proxy version to inject
  image:
    repository: "istio/proxyv2"
    tag: "1.17.0"  # Customize version
  
  # System namespaces to exclude (never inject)
  excludeNamespaces:
    - kyverno
    - kube-system
    - istio-system
  
  # Pods with these labels skip injection
  excludePodSelector:
    matchLabels:
      skip-istio: "true"
```

**Example 1: Different Istio versions per environment**
```yaml
# In values-prod.yaml
injectIstioSidecar:
  enabled: true
  image:
    tag: "1.18.0"  # Production version

# In values-dev.yaml
injectIstioSidecar:
  enabled: true
  image:
    tag: "1.17.0"  # Dev/test version
```

**Example 2: Exclude specific workloads**
```yaml
injectIstioSidecar:
  enabled: true
  excludeNamespaces:
    - kyverno
    - kube-system
    - istio-system
    - monitoring  # Don't inject in monitoring ns
```

---

#### 4. Add Default Resources (`addDefaultResources`)

**Purpose**: Auto-adds CPU and memory requests/limits for predictable scheduling

**Location in values.yaml**:
```yaml
policies:
  mutate:
    addDefaultResources:
```

**Customizable Options**:

```yaml
addDefaultResources:
  enabled: true
  failureAction: audit
  
  namespaceSelector:
    matchLabels:
      auto-add-requests: "true"
  
  # Default resources for all containers
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
  
  # Override defaults per environment
  namespaceDefaults:
    dev:
      requests:
        cpu: "50m"
        memory: "64Mi"
      limits:
        cpu: "200m"
        memory: "256Mi"
    
    prod:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "1000m"
        memory: "1Gi"
```

**Example 1: Conservative defaults**
```yaml
addDefaultResources:
  enabled: true
  resources:
    requests:
      cpu: "50m"      # Very low
      memory: "64Mi"
    limits:
      cpu: "100m"
      memory: "128Mi"
```

**Example 2: Production-grade defaults**
```yaml
addDefaultResources:
  enabled: true
  resources:
    requests:
      cpu: "250m"     # Higher for stability
      memory: "256Mi"
    limits:
      cpu: "1000m"
      memory: "1Gi"
```

---

#### 5. Add Monitoring Annotations (`addMonitoringAnnotations`)

**Purpose**: Auto-adds Prometheus scrape annotations for metrics collection

**Location in values.yaml**:
```yaml
policies:
  mutate:
    addMonitoringAnnotations:
```

**Customizable Options**:

```yaml
addMonitoringAnnotations:
  enabled: true
  failureAction: audit
  
  namespaceSelector:
    matchLabels:
      prometheus-monitoring: "enabled"
  
  # Prometheus scrape annotations
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"         # Metrics port
    prometheus.io/path: "/metrics"     # Metrics path
    prometheus.io/scheme: "http"       # http or https
  
  # Override ports for specific apps
  customPorts:
    myapp: "9090"
    another-app: "5000"
```

**Example 1: HTTPS metrics**
```yaml
addMonitoringAnnotations:
  enabled: true
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8443"
    prometheus.io/path: "/metrics"
    prometheus.io/scheme: "https"
    prometheus.io/tls-verify: "false"
```

**Example 2: App-specific ports**
```yaml
addMonitoringAnnotations:
  enabled: true
  customPorts:
    prometheus: "9090"
    grafana: "3000"
    alertmanager: "9093"
```

---

### GENERATE POLICIES

#### 1. Generate NetworkPolicy (`generateNetworkPolicy`)

**Purpose**: Auto-creates network isolation policy for new namespaces

**Location in values.yaml**:
```yaml
policies:
  generate:
    generateNetworkPolicy:
```

**Customizable Options**:

```yaml
generateNetworkPolicy:
  enabled: true
  resourceKinds:
    - Namespace
  
  # Only create NetworkPolicy for namespaces with this label
  namespaceSelector:
    matchLabels:
      network-policy: "enabled"
  
  # Name of the created NetworkPolicy
  policyName: "default-deny-ingress"

  # Pods covered by the generated NetworkPolicy ({} means every Pod)
  podSelector: {}

  policyTypes:
    - Ingress
  
  # Allowed ingress ports (optional whitelist)
  allowedPorts:
    - port: 8080
      protocol: TCP
    - port: 443
      protocol: TCP
```

**Example 1: Create for all namespaces**
```yaml
generateNetworkPolicy:
  enabled: true
  namespaceSelector:
    matchLabels: {}
  policyName: "default-deny-all"
```

**Example 2: With whitelist rules**
```yaml
generateNetworkPolicy:
  enabled: true
  allowedPorts:
    - port: 80
      protocol: TCP
    - port: 443
      protocol: TCP
    - port: 5432
      protocol: TCP  # PostgreSQL
```

---

#### 2. Clone Registry Secrets (`cloneRegistrySecrets`)

**Purpose**: Auto-distributes docker credentials to new namespaces

**Location in values.yaml**:
```yaml
policies:
  generate:
    cloneRegistrySecrets:
```

**Customizable Options**:

```yaml
cloneRegistrySecrets:
  enabled: true
  resourceKinds:
    - Namespace

  # Only clone to matching newly created namespaces
  namespaceSelector:
    matchLabels:
      clone-secrets: "true"
  
  # Source namespace (where original secret exists)
  sourceNamespace: "default"
  
  # Name of secret to clone
  secretName: "docker-registry-credentials"
  
  # Name in destination namespace (can be different)
  targetSecretName: "docker-registry-credentials"
```

**Example 1: Clone from different source**
```yaml
cloneRegistrySecrets:
  enabled: true
  sourceNamespace: "docker-config"
  secretName: "docker-auth"
  targetSecretName: "docker-pull-secret"
```

---

#### 3. Generate ResourceQuota (`generateResourceQuota`)

**Purpose**: Auto-limits total resources per namespace

**Location in values.yaml**:
```yaml
policies:
  generate:
    generateResourceQuota:
```

**Customizable Options**:

```yaml
generateResourceQuota:
  enabled: true
  resourceKinds:
    - Namespace
  
  # One generation rule is rendered per quota item
  quotas:
    dev:
      ruleName: "create-dev-quota"
      namespaceSelector:
        matchLabels:
          env: "development"
      quotaName: "dev-quota"
      hard:
        requests.cpu: "5"
        requests.memory: "10Gi"
        limits.cpu: "10"
        limits.memory: "20Gi"
        pods: "100"
    
    prod:
      ruleName: "create-prod-quota"
      namespaceSelector:
        matchLabels:
          env: "production"
      quotaName: "prod-quota"
      hard:
        requests.cpu: "100"
        requests.memory: "200Gi"
        limits.cpu: "200"
        limits.memory: "400Gi"
        pods: "500"
```

**Example 1: Strict quotas for dev**
```yaml
generateResourceQuota:
  enabled: true
  quotas:
    dev:
      ruleName: "create-dev-strict-quota"
      namespaceSelector:
        matchLabels:
          env: "development"
      quotaName: "dev-strict"
      hard:
        requests.cpu: "2"
        requests.memory: "4Gi"
        pods: "20"
```

---

#### 4. Generate LimitRange (`generateLimitRange`)

**Purpose**: Auto-sets default resource limits for containers

**Location in values.yaml**:
```yaml
policies:
  generate:
    generateLimitRange:
```

**Customizable Options**:

```yaml
generateLimitRange:
  enabled: true
  resourceKinds:
    - Namespace
  namespaceSelector:
    matchLabels: {}  # Empty = every newly created namespace
  limitRangeName: "default-limits"
  
  # Container defaults
  containerLimits:
    max:
      cpu: "2"
      memory: "2Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    ratio:
      limits/requests: "4"
  podLimits:
    max:
      cpu: "4"
      memory: "4Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
```

---

#### 5. Clone Secret on Namespace Creation (`cloneSecretOnNamespaceCreation`)

**Purpose**: Clones a named Secret from a source namespace into matching new namespaces.

```yaml
cloneSecretOnNamespaceCreation:
  enabled: true
  resourceKinds:
    - Namespace
  namespaceSelector:
    matchLabels: {}  # Empty = every newly created namespace
  sourceNamespace: "default"
  sourceSecretName: "registry-credentials"
  targetSecretName: "registry-credentials"
```

---

#### 6. Default NetworkPolicy (`defaultNetworkPolicy`)

**Purpose**: Creates an ingress-deny NetworkPolicy for namespaces selected by expressions.

```yaml
defaultNetworkPolicy:
  enabled: true
  resourceKinds:
    - Namespace
  namespaceSelector:
    matchExpressions:
      - key: "network-policy"
        operator: In
        values:
          - "default"
  policyName: "default-deny-ingress"
  podSelector: {}
  policyTypes:
    - Ingress
```

---

## Global Configuration

All policies respect these global settings:

```yaml
global:
  # Default failure action (can override per-policy)
  defaultFailureAction: "audit"  # audit | enforce
  
  # System namespaces to always exclude
  excludeNamespaces:
    - kube-system
    - kube-public
    - kyverno
    - default
  
  # Apply policies to existing resources
  background: true
  
  # Validation failure action
  validationFailureAction: "audit"
```

---

## Common Customization Scenarios

### Scenario 1: Development Environment

**Goal**: Permissive settings, easy testing

```yaml
# values-dev.yaml
policies:
  validate:
    enabled: false  # Skip strict validation in dev

  mutate:
    addDefaultLabels:
      enabled: true
      failureAction: audit
    addSecurityContext:
      enabled: false  # Too strict for local dev
    addDefaultResources:
      enabled: true
      resources:
        requests:
          cpu: "10m"
          memory: "32Mi"
        limits:
          cpu: "100m"
          memory: "128Mi"
  
  generate:
    generateNetworkPolicy:
      enabled: false  # No network restrictions in dev
```

### Scenario 2: Production Environment

**Goal**: Strict security and resource management

```yaml
# values-prod.yaml
policies:
  validate:
    enabled: true
    disallowRoot:
      enabled: true
      failureAction: enforce
      resourceKinds: [Pod]
      namespaceSelector:
        matchLabels: {}   # cluster-wide
    enforceResources:
      enabled: true
      failureAction: enforce
      resourceKinds: [Pod, Deployment, StatefulSet]
    restrictRegistries:
      enabled: true
      failureAction: enforce
      approvedRegistries:
        - "gcr.io"
        - "myregistry.company.com"

  mutate:
    addDefaultLabels:
      enabled: true
      failureAction: enforce  # Block if fails
    addSecurityContext:
      enabled: true
      failureAction: enforce  # Strict security
    addDefaultResources:
      enabled: true
      failureAction: enforce  # Ensure resources are set
      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
        limits:
          cpu: "1000m"
          memory: "1Gi"
  
  generate:
    generateNetworkPolicy:
      enabled: true
    generateResourceQuota:
      enabled: true
```

### Scenario 3: Selective Policies

**Goal**: Only enable specific policies

```yaml
# values-security-only.yaml
policies:
  validate:
    enabled: true
    disallowRoot:
      enabled: true
      failureAction: enforce
    requireImageDigest:
      enabled: false
    enforceResources:
      enabled: false
    restrictRegistries:
      enabled: true
      failureAction: enforce
    requireLabels:
      enabled: false

  mutate:
    addDefaultLabels:
      enabled: false
    addSecurityContext:
      enabled: true  # Only security
    addDefaultResources:
      enabled: false
    addMonitoringAnnotations:
      enabled: false
  
  generate:
    generateNetworkPolicy:
      enabled: true  # Only network policies
```

---

## Deployment Examples

### Deploy with default values
```bash
helm upgrade --install kyverno ./use-cases/templates-chart
```

### Deploy with custom values
```bash
helm upgrade --install kyverno ./use-cases/templates-chart \
  -f values-prod.yaml
```

### Override specific setting
```bash
helm upgrade kyverno ./use-cases/templates-chart \
  --set policies.mutate.addDefaultLabels.enabled=false
```

### Disable all policies temporarily
```bash
helm upgrade kyverno ./use-cases/templates-chart \
  --set policies.validate.enabled=false \
  --set policies.mutate.enabled=false \
  --set policies.generate.enabled=false
```

---

## Verifying Customizations

### Check deployed policy
```bash
# See the actual YAML with values rendered
helm get values kyverno

# Get the full manifest
helm get manifest kyverno

# Describe a specific policy
kubectl describe clusterpolicy add-default-labels
```

### Test a policy
```bash
# Create a test pod
kubectl run test --image=nginx -n test-ns

# Check if policy applied (for mutate)
kubectl get pod test -n test-ns -o yaml | grep labels

# Check policy report
kubectl get policyreport -n test-ns
```

---

## Tips & Best Practices

1. **Start with audit mode**: Enable policies with `failureAction: audit` first
2. **Test in dev**: Always test customizations in development before production
3. **Version control**: Keep custom values files in Git for tracking changes
4. **Document changes**: Comment custom values explaining why they differ from defaults
5. **Gradual rollout**: Enable policies gradually across namespaces
6. **Monitor violations**: Check policy reports regularly before switching to enforce
7. **Use namespaceSelector**: Apply policies to specific namespaces using labels
8. **Exclude system namespaces**: Always exclude kube-system, kyverno, etc.

---

## Troubleshooting

### Policy not applying
```bash
# Check policy is created
kubectl get clusterpolicy

# Verify namespace has correct labels
kubectl get ns --show-labels

# Check Kyverno logs
kubectl logs -n kyverno-system -l app=kyverno
```

### Wrong values being used
```bash
# View current values
helm get values kyverno

# Reset to defaults
helm upgrade kyverno ./use-cases/templates-chart

# Use custom values again
helm upgrade kyverno ./use-cases/templates-chart -f values-prod.yaml
```

---

## Next Steps

1. **Copy values-custom.yaml**: `cp templates-chart/values-custom.yaml values-myorg.yaml`
2. **Edit for your needs**: Customize policies, resources, namespaces
3. **Deploy**: `helm upgrade kyverno ./templates-chart -f values-myorg.yaml`
4. **Monitor**: `kubectl get policyreport -A`
5. **Iterate**: Adjust settings based on policy violations and feedback
