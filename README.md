# GitOps Fleet - Store Stack Deployment

This repository manages the deployment of the Store Stack application across multiple edge devices using **Rancher Fleet** with a label-based, version-aware approach.

## 🏗️ Repository Structure

```
gitops-fleet-3/
├── clusters/                    # Cluster definitions with labels
│   ├── edge-device-001.yaml    # Label: store-version=v1.4
│   └── edge-device-002.yaml    # Label: store-version=v1.5
│
└── apps/
    └── store-stack/             # SINGLE app folder for all versions
        ├── fleet.yaml           # Fleet logic with targetCustomizations
        └── versions/            # Version-specific Helm values
            ├── v1.4.0.yaml     # Values for v1.4 (image tags, resources)
            └── v1.5.0.yaml     # Values for v1.5 (image tags, resources)
```

## 🎯 Key Concepts

### 1. **Label-Based Deployment**
Clusters are labeled with `store-version` to determine which version to deploy:
- `store-version: "v1.4"` → Deploys chart version `1.4.0`
- `store-version: "v1.5"` → Deploys chart version `1.5.0`

### 2. **Dynamic Configuration Injection**
Cluster-specific configuration is injected via labels:
- `lb-ip` → Sets `nginx-service.service.loadBalancerIP`
- `replicas` → Sets `nginx-service.replicaCount`

### 3. **Single Source of Truth**
One `fleet.yaml` manages all clusters and versions using `targetCustomizations`.

## 📝 How It Works

### Step 1: Cluster Registration
Each cluster is registered with specific labels:

```yaml
# clusters/edge-device-001.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: Cluster
metadata:
  name: edge-device-001
  labels:
    store-version: "v1.4"      # Version selector
    lb-ip: "192.168.1.10"      # Cluster-specific IP
    replicas: "2"              # Cluster-specific replica count
    environment: production
    location: store-east
```

### Step 2: Fleet Configuration
`fleet.yaml` uses `targetCustomizations` to match clusters by labels:

```yaml
targetCustomizations:
  - name: train-v1.4
    clusterSelector:
      matchLabels:
        store-version: "v1.4"
    helm:
      version: 1.4.0
      valuesFiles:
        - versions/v1.4.0.yaml
      values:
        nginx-service:
          replicaCount: "${ .ClusterLabels.replicas }"
          service:
            loadBalancerIP: "${ .ClusterLabels.lb-ip }"
```

### Step 3: Version-Specific Values
Each version has a dedicated values file:

```yaml
# versions/v1.4.0.yaml
nginx-service:
  image:
    tag: "1.27.0"
  resources:
    limits:
      cpu: 100m
      memory: 128Mi
```

## 🚀 Usage

### Adding a New Cluster

1. **Create cluster file** in `clusters/`:
```yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: Cluster
metadata:
  name: edge-device-003
  labels:
    store-version: "v1.5"
    lb-ip: "192.168.1.30"
    replicas: "3"
spec:
  kubeConfigSecret: edge-device-003-kubeconfig
```

2. **Commit and push** - Fleet automatically deploys based on labels!

### Adding a New Version (v1.6)

1. **Create version values file** `versions/v1.6.0.yaml`:
```yaml
nginx-service:
  image:
    tag: "1.28.0"
# ... other config
```

2. **Add targetCustomization** to `fleet.yaml`:
```yaml
targetCustomizations:
  - name: train-v1.6
    clusterSelector:
      matchLabels:
        store-version: "v1.6"
    helm:
      version: 1.6.0
      valuesFiles:
        - versions/v1.6.0.yaml
      values:
        nginx-service:
          replicaCount: "${ .ClusterLabels.replicas }"
          service:
            loadBalancerIP: "${ .ClusterLabels.lb-ip }"
```

3. **No folder duplication needed!** ✅

### Upgrading a Cluster

Simply change the cluster label:
```bash
kubectl label cluster edge-device-001 store-version=v1.5 --overwrite
```

Fleet automatically redeploys with the new version.

## 🔧 Advantages Over Previous Approach

| Previous (`gitops-fleet-2`) | New (`gitops-fleet-3`) |
|------------------------------|------------------------|
| Separate folder per version (`store-stack-v1.4/`, `store-stack-v1.5/`) | Single `store-stack/` folder |
| Duplicate `fleet.yaml` files | One `fleet.yaml` with targetCustomizations |
| Cluster config in `fleet.yaml` | Cluster config in cluster labels |
| Manual `Chart.yaml` per version | Direct chart reference by version |
| Harder to scale (500 clusters × 2 versions = complexity) | Scales linearly (labels handle everything) |

## 📊 Scaling Example

### 500 Clusters, 2 Versions:
- **Old approach**: 2 folders, 2 `fleet.yaml` files, cluster IPs duplicated
- **New approach**: 1 folder, 1 `fleet.yaml`, 500 cluster files with labels

### Adding Version 1.6:
- **Old approach**: Copy folder, update `Chart.yaml`, duplicate `fleet.yaml`
- **New approach**: Add `v1.6.0.yaml` + 10 lines in `fleet.yaml`

## 🔍 Label Reference

| Label | Purpose | Example |
|-------|---------|---------|
| `store-version` | **Required**. Determines chart version | `v1.4`, `v1.5` |
| `lb-ip` | LoadBalancer IP for nginx service | `192.168.1.10` |
| `replicas` | Number of nginx replicas | `2`, `3` |
| `environment` | Cluster environment (metadata) | `production`, `staging` |
| `location` | Physical location (metadata) | `store-east` |
| `region` | Cloud region (metadata) | `us-east-1` |

## 📦 Chart Repository

Charts are served from: `https://saurabh-newera.github.io/helm-charts`

Available versions:
- `store-stack-1.4.0.tgz`
- `store-stack-1.5.0.tgz`

## 🐛 Troubleshooting

### Fleet not deploying to cluster
1. Check cluster labels: `kubectl get cluster edge-device-001 -o yaml`
2. Verify `store-version` label matches a `targetCustomization`
3. Check Fleet status: `kubectl get bundledeployments -A`

### Wrong version deployed
1. Verify cluster label is correct
2. Check `fleet.yaml` targetCustomization selector
3. Ensure chart version exists in Helm repo

### Label interpolation not working
Fleet uses `${ .ClusterLabels.key }` syntax. Ensure:
- Label exists on cluster resource
- Syntax is exactly as shown (case-sensitive)
- Label values are strings (even numbers like `"2"`)

## 🎓 Learn More

- [Fleet Documentation](https://fleet.rancher.io/)
- [Fleet targetCustomizations](https://fleet.rancher.io/ref-fleet-yaml#targetcustomizations)
- [Helm Charts Repository](https://github.com/saurabh-newera/helm-charts)

## 📄 License

MIT
