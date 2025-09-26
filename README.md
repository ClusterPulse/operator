# ClusterPulse Operator Documentation

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Components](#components)
- [Custom Resource Definitions](#custom-resource-definitions)
- [Development Setup](#development-setup)
- [Building & Testing](#building--testing)
- [Deployment](#deployment)
- [API Reference](#api-reference)

## Overview

ClusterPulse is a Helm-based Operator built with the Operator SDK that provides comprehensive multi-cluster OpenShift/Kubernetes monitoring with fine-grained Role-Based Access Control (RBAC). It's designed to be deployed via the Operator Lifecycle Manager (OLM) and manages the entire ClusterPulse monitoring platform lifecycle.

### Key Features

- **Multi-Cluster Monitoring**: Monitor multiple OpenShift/Kubernetes clusters from a single pane of glass
- **Fine-Grained RBAC**: Policy-based access control with namespace, node, and resource filtering
- **Container Registry Monitoring**: Track registry health and availability
- **Operator Management**: Visibility into installed operators across clusters
- **Real-Time Metrics**: Live cluster health, resource utilization, and status updates
- **OAuth Integration**: Seamless OpenShift OAuth proxy integration for authentication

## Architecture

### System Overview

```
┌───────────────────────────────────────────────────────────────────┐
│                     OpenShift/Kubernetes Cluster                  │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐    │
│  │                  ClusterPulse Operator                    │    │
│  │                    (Helm Operator)                        │    │
│  └────────────────────┬──────────────────────────────────────┘    │
│                       │ Manages                                   │
│                       ▼                                           │
│  ┌───────────────────────────────────────────────────────────┐    │
│  │                  ClusterPulse CR                          │    │
│  └────────────────────┬──────────────────────────────────────┘    │
│                       │ Deploys                                   │
│         ┌─────────────┼─────────────┬──────────────┬────────┐     │
│         ▼             ▼             ▼              ▼        ▼     │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────┐ ┌─────┐  │
│  │Frontend UI │ │  API       │ │  Cluster   │ │Policy  │ │Redis│  │
│  │  + OAuth   │ │  Backend   │ │ Controller │ │Engine  │ │     │  │
│  └────────────┘ └────────────┘ └────────────┘ └────────┘ └─────┘  │
│         │             │             │              │        │     │
│         └─────────────┴─────────────┼──────────────┴────────┘     │
│                                     │                             │
│  ┌──────────────────────────────────▼──────────────────────────┐  │
│  │         Custom Resources (CRDs)                             │  │
│  │  • ClusterConnection  • RegistryConnection                  │  │
│  │  • MonitorAccessPolicy                                      │  │
│  └─────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
                                │
                     Monitors   ▼
            ┌─────────────────────────────────┐
            │    Target Clusters/Registries   │
            └─────────────────────────────────┘
```

### Component Interaction Flow

1. **Operator Deployment**: OLM installs the ClusterPulse operator
2. **CR Creation**: User creates a `ClusterPulse` custom resource
3. **Helm Rendering**: Operator renders Helm charts based on CR spec
4. **Component Deployment**: Deploys UI, API, controllers, and Redis
5. **CRD Management**: Users create ClusterConnection/RegistryConnection resources
6. **Data Collection**: Controllers connect to target clusters and collect metrics
7. **Policy Enforcement**: Policy engine evaluates MonitorAccessPolicy CRs
8. **Data Storage**: Metrics stored in Redis
9. **API Access**: Frontend queries API with OAuth authentication
10. **RBAC Filtering**: API filters data based on user permissions

## Components

### ClusterPulse Operator

The main operator that manages the ClusterPulse platform deployment.

**Key Files**:
- `watches.yaml`: Defines watched CRDs and Helm chart mappings
- `Dockerfile`: Container image definition
- `helm-charts/clusterpulse/`: Main Helm chart

**Responsibilities**:
- Watch for `ClusterPulse` custom resources
- Render and deploy Helm charts
- Manage component lifecycle
- Handle upgrades and rollbacks

### Frontend UI

React-based dashboard for cluster monitoring.

**Features**:
- Real-time cluster monitoring dashboard
- Node and operator management views
- Dark/light theme support
- OAuth proxy integration
- PatternFly design system

**Configuration** (via Helm values):
```yaml
frontend:
  replicas: 3
  image: "youruser/clusterpulse-ui:latest"
  oauth:
    enabled: true
    provider: openshift
  route:
    enabled: true
    tls:
      termination: reencrypt
```

### API Backend

FastAPI service providing REST endpoints with RBAC enforcement.

**Configuration**:
```yaml
api:
  replicas: 3
  rbac:
    enabled: true
    defaultDeny: true
    cacheTTL: 300
```

### Cluster Controller

Kubernetes operator for managing cluster and registry connections.

**Features**:
- Reconciles ClusterConnection CRs
- Reconciles RegistryConnection CRs
- Collects cluster metrics and health
- Discovers installed operators
- Stores data in Redis

**Configuration**:
```yaml
clusterEngine:
  replicas: 1
  timing:
    reconciliationInterval: "30"
    nodeMetricsInterval: "15"
    operatorScanInterval: "300"
```

### Policy Controller

Python-based controller for RBAC policy management.

**Features**:
- Reconciles MonitorAccessPolicy CRs
- Compiles policies for fast evaluation
- Manages Redis policy storage
- Handles cache invalidation

**Configuration**:
```yaml
policyEngine:
  replicas: 1
  cache:
    policyCacheTTL: 300
    groupCacheTTL: 300
```

## Custom Resource Definitions

### ClusterPulse

The main CR that defines the ClusterPulse platform deployment.

```yaml
apiVersion: charts.clusterpulse.io/v1alpha1
kind: ClusterPulse
metadata:
  name: clusterpulse-prod
spec:
  redis:
    enabled: true
  
  api:
    replicas: 3
    rbac:
      enabled: true
      defaultDeny: true
  
  frontend:
    replicas: 3
    oauth:
      enabled: true
  
  clusterEngine:
    replicas: 1
    timing:
      reconciliationInterval: "30"
  
  policyEngine:
    enabled: true
    replicas: 1
```

### ClusterConnection

Defines a connection to a monitored Kubernetes/OpenShift cluster.

```yaml
apiVersion: clusterpulse.io/v1alpha1
kind: ClusterConnection
metadata:
  name: production-cluster
spec:
  displayName: "Production Cluster"
  endpoint: "https://api.cluster.example.com:6443"
  credentialsRef:
    name: cluster-credentials
  monitoring:
    interval: 30
    timeout: 10
```

### RegistryConnection

Defines a connection to a container registry.

```yaml
apiVersion: clusterpulse.io/v1alpha1
kind: RegistryConnection
metadata:
  name: dockerhub
spec:
  displayName: "Docker Hub"
  endpoint: "https://registry-1.docker.io"
  skipTLSVerify: false
  monitoring:
    interval: 60
```

### MonitorAccessPolicy

Defines fine-grained access control policies.

```yaml
apiVersion: clusterpulse.io/v1alpha1
kind: MonitorAccessPolicy
metadata:
  name: developer-policy
spec:
  identity:
    priority: 100
    subjects:
      users: ["alice@example.com"]
      groups: ["developers"]
  access:
    effect: Allow
    enabled: true
  scope:
    clusters:
      default: none
      rules:
        - selector:
            matchPattern: "dev-.*"
          permissions:
            view: true
            viewMetrics: true
```

## Development Setup

### Prerequisites

- Go 1.21+
- Docker/Podman
- kubectl/oc CLI
- make
- Access to an OpenShift/Kubernetes cluster
- operator-sdk v1.41.1 (optional, will be downloaded)

### Local Development

1. **Clone the repository**:
```bash
git clone https://github.com/ClusterPulse/operator.git
cd operator
```

2. **Install CRDs**:
```bash
make install
```

3. **Run the operator locally**:
```bash
# Set namespace to watch
export WATCH_NAMESPACE=clusterpulse

# Run operator
make run
```

4. **Deploy a sample CR**:
```bash
kubectl apply -f config/samples/charts_v1alpha1_clusterpulse.yaml
```

### Development Workflow

1. **Modify Helm charts**: Edit files in `helm-charts/clusterpulse/`
2. **Update CRDs**: Modify files in `config/crd/bases/`
3. **Test locally**: Use `make run` to test changes
4. **Build image**: `make docker-build IMG=your-registry/clusterpulse:dev`
5. **Push image**: `make docker-push IMG=your-registry/clusterpulse:dev`
6. **Deploy to cluster**: `make deploy IMG=your-registry/clusterpulse:dev`

## Building & Testing

### Building the Operator

```bash
# Build operator image
make docker-build IMG=docker.io/youruser/clusterpulse-operator:latest

# Build for multiple architectures
make docker-buildx IMG=docker.io/youruser/clusterpulse-operator:latest

# Push to registry
make docker-push IMG=docker.io/youruser/clusterpulse-operator:latest
```

### Building the Bundle

```bash
# Generate bundle manifests
make bundle VERSION=0.1.0

# Build bundle image
make bundle-build BUNDLE_IMG=docker.io/youruser/clusterpulse-bundle:v0.1.0

# Push bundle image
make bundle-push BUNDLE_IMG=docker.io/youruser/clusterpulse-bundle:v0.1.0
```

### Building the Catalog

```bash
# Build catalog image for OLM
make catalog-build CATALOG_IMG=docker.io/youruser/clusterpulse-catalog:v0.1.0

# Push catalog image
make catalog-push CATALOG_IMG=docker.io/youruser/clusterpulse-catalog:v0.1.0
```

### Running Tests

```bash
# Run operator scorecard tests
operator-sdk scorecard bundle

# Validate bundle
operator-sdk bundle validate ./bundle

# Test Helm chart rendering
helm template test helm-charts/clusterpulse --debug
```

## Deployment

### Via OLM

1. **Create CatalogSource**:
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: clusterpulse-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: docker.io/youruser/clusterpulse-catalog:v0.1.0
  displayName: ClusterPulse Catalog
```

2. **Create Subscription**:
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: clusterpulse
  namespace: clusterpulse-system
spec:
  channel: stable
  name: clusterpulse
  source: clusterpulse-catalog
  sourceNamespace: openshift-marketplace
```

3. **Create ClusterPulse instance**:
```bash
kubectl apply -f config/samples/charts_v1alpha1_clusterpulse.yaml
```

### Manual Deployment

```bash
# Deploy operator and CRDs
make deploy IMG=docker.io/youruser/clusterpulse-operator:latest

# Create namespace
kubectl create namespace clusterpulse

# Deploy sample configuration
kubectl apply -f config/samples/
```

## Contributing Guidelines

### Code Organization

- **Operator logic**: `operator/` directory
- **Helm templates**: `operator/helm-charts/clusterpulse/templates/`
- **CRD definitions**: `operator/config/crd/bases/`
- **RBAC configs**: `operator/config/rbac/`
- **Samples**: `operator/config/samples/`

### Adding a New Component

1. **Create Helm templates** in `helm-charts/clusterpulse/templates/<component>/`
2. **Add configuration** to CRD schema in `config/crd/bases/charts.clusterpulse.io_clusterpulses.yaml`
3. **Update values.yaml** with default configuration
4. **Add RBAC rules** if needed in `config/rbac/role.yaml`
5. **Update documentation** and samples

### Modifying CRDs

1. **Edit CRD YAML** in `config/crd/bases/`
2. **Regenerate bundle**: `make bundle`
3. **Update samples** in `config/samples/`
4. **Test changes**: `make install && make run`

### Testing Checklist

- [ ] CRDs install successfully
- [ ] Operator starts without errors
- [ ] Helm charts render correctly
- [ ] Components deploy as expected
- [ ] RBAC permissions are correct
- [ ] Services are accessible
- [ ] Redis connectivity works
- [ ] OAuth proxy functions (OpenShift)
- [ ] Frontend loads properly
- [ ] API endpoints respond
- [ ] Cluster connections work
- [ ] Policy enforcement functions

## API Reference

### Helm Values Reference

Key configuration options in the Helm chart:

```yaml
# Redis configuration
redis:
  enabled: true
  auth:
    enabled: true
    existingSecret: ""

# API configuration  
api:
  enabled: true
  replicas: 3
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "512Mi"
      cpu: "500m"

# Frontend configuration
frontend:
  enabled: true
  replicas: 3
  oauth:
    enabled: true
    provider: openshift
    
# Cluster controller
clusterEngine:
  replicas: 1
  
# Policy engine
policyEngine:
  enabled: true
  replicas: 1
```
