# Kafka Connect with Debezium - Helm Chart

Kafka Connect cluster with Debezium connectors using Strimzi operator - deployed via ArgoCD or Helm.

## Overview

This Helm chart deploys a highly configurable Kafka Connect cluster using Strimzi's KafkaConnect custom resource. The cluster uses a custom image with Debezium connectors pre-installed for CDC (Change Data Capture).

## Features

- ✅ **Helm Templated**: Fully parameterized for multiple deployments
- ✅ **Debezium Connectors**: Pre-installed in custom image
- ✅ **Strimzi Integration**: Managed by Strimzi operator
- ✅ **Connector Resources**: Dynamic connector management via CRDs
- ✅ **Multiple Environments**: Dev and prod value files included
- ✅ **Configurable Resources**: CPU, memory, replicas customizable
- ✅ **Secret Management**: PostgreSQL credentials via Kubernetes secrets
- ✅ **GitOps Ready**: Managed via ArgoCD

## Prerequisites

- Kubernetes cluster (v1.19+)
- Strimzi Kafka Operator installed (v0.48.0+)
- Kafka cluster running (kraft-kafka-cluster)
- Helm 3.x (for manual installation)
- ArgoCD (for GitOps deployment)

## Repository Structure

```
kafka-connect/
├── Chart.yaml              # Helm chart metadata
├── values.yaml             # Default values
├── values-dev.yaml         # Development environment values
├── values-prod.yaml        # Production environment values
├── templates/
│   ├── namespace.yaml      # Namespace template
│   ├── kafka-connect.yaml  # KafkaConnect CR template
│   └── postgres-secret.yaml # PostgreSQL secret template
└── README.md               # This file
```

## Configuration

### Default Values

| Component | Value |
|-----------|-------|
| Name | debezium-oracle-connect |
| Namespace | kraft-kafka-connect |
| Replicas | 2 |
| Image | kadirsahan/kafka-connect-strimzi-build:1.4 |
| Kafka Cluster | kraft-kafka-cluster-kafka-bootstrap.kraft-kafka.svc.cluster.local:9092 |

### Image Contents

**Custom Image**: `kadirsahan/kafka-connect-strimzi-build:1.4`
- Debezium 3.3.0.Final
- Debezium JDBC Sink
- Kafka 4.0 compatible

### Key Parameters

```yaml
# Cluster identity
nameOverride: "debezium-oracle-connect"
namespace: "kraft-kafka-connect"

# Kafka Connect
kafkaConnect.replicas: 2
kafkaConnect.image: "kadirsahan/kafka-connect-strimzi-build:1.4"
kafkaConnect.bootstrapServers: "kraft-kafka-cluster-kafka-bootstrap.kraft-kafka.svc.cluster.local:9092"

# Resources
kafkaConnect.resources.requests.memory: "512Mi"
kafkaConnect.resources.requests.cpu: "300m"
kafkaConnect.resources.limits.memory: "2Gi"
kafkaConnect.resources.limits.cpu: "1000m"

# Configuration
kafkaConnect.config.configStorageReplicationFactor: 1
kafkaConnect.config.offsetStorageReplicationFactor: 1
kafkaConnect.config.statusStorageReplicationFactor: 1

# PostgreSQL secret
postgresSecret.enabled: true
postgresSecret.name: "postgres-credentials"
postgresSecret.password: "base64-encoded-password"
```

## Installation

### Option 1: Using ArgoCD (Recommended)

This chart is designed to be deployed via ArgoCD:

```bash
# Deployed automatically via app-of-apps pattern
kubectl apply -f https://raw.githubusercontent.com/kadirsahan/argocd-apps/main/app-of-apps.yaml
```

The ArgoCD Application manifest uses default values:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kafka-connect
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/kadirsahan/kafka-connect.git
    targetRevision: main
    path: .
    helm:
      valueFiles:
        - values.yaml  # Use values-dev.yaml or values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: kraft-kafka-connect
```

### Option 2: Using Helm Directly

```bash
# Clone the repository
git clone https://github.com/kadirsahan/kafka-connect.git
cd kafka-connect

# Install with default values
helm install kafka-connect-prod . -n kraft-kafka-connect --create-namespace

# Install with development values
helm install kafka-connect-dev . -f values-dev.yaml -n kraft-kafka-connect-dev --create-namespace

# Install with custom values
helm install my-kafka-connect . -f my-values.yaml -n my-namespace --create-namespace
```

### Option 3: Manual kubectl

```bash
# Render templates and apply
helm template kafka-connect-prod . | kubectl apply -f -

# Or with custom values
helm template kafka-connect-dev . -f values-dev.yaml | kubectl apply -f -
```

## Environment-Specific Deployments

### Development Environment

Uses `values-dev.yaml`:
- 1 replica (minimal resources)
- Lower memory (256Mi-1Gi)
- Lower CPU (200m-500m)
- Replication factor: 1

```bash
helm install kafka-connect-dev . -f values-dev.yaml -n kraft-kafka-connect-dev --create-namespace
```

### Production Environment

Uses `values-prod.yaml`:
- 3 replicas (high availability)
- Higher memory (1Gi-4Gi)
- Higher CPU (500m-2000m)
- Replication factor: 3

```bash
helm install kafka-connect-prod . -f values-prod.yaml -n kraft-kafka-connect --create-namespace
```

## Deploying Multiple Kafka Connect Clusters

Deploy multiple independent Kafka Connect clusters with different configurations:

```bash
# Cluster 1: Development
helm install connect1 . -f values-dev.yaml \
  --set nameOverride=connect1 \
  --set namespace=connect1 \
  --create-namespace

# Cluster 2: Staging
helm install connect2 . \
  --set nameOverride=connect2 \
  --set namespace=connect2 \
  --set kafkaConnect.replicas=2 \
  --create-namespace

# Cluster 3: Production
helm install connect3 . -f values-prod.yaml \
  --set nameOverride=connect3 \
  --set namespace=connect3 \
  --create-namespace
```

## Verify Installation

```bash
# Check Helm release
helm list -n kraft-kafka-connect

# Check KafkaConnect resource
kubectl get kafkaconnect -n kraft-kafka-connect

# Check pods
kubectl get pods -n kraft-kafka-connect

# Watch connector creation
kubectl get kafkaconnect debezium-oracle-connect -n kraft-kafka-connect -w
```

Expected output when ready:
```
NAME                      DESIRED REPLICAS   READY
debezium-oracle-connect   2                  True
```

## Managing Connectors

### Using Strimzi Connector CRD

With `strimzi.io/use-connector-resources: "true"`, you can manage connectors as Kubernetes resources:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  name: my-source-connector
  namespace: kraft-kafka-connect
  labels:
    strimzi.io/cluster: debezium-oracle-connect
spec:
  class: io.debezium.connector.oracle.OracleConnector
  tasksMax: 1
  config:
    database.hostname: oracle-db
    database.port: 1521
    database.user: debezium
    database.password: ${file:/mnt/postgres-credentials/password}
    database.dbname: mydb
    database.server.name: myserver
    table.include.list: public.my_table
```

### Using REST API

```bash
# Port-forward to Kafka Connect
kubectl port-forward -n kraft-kafka-connect svc/debezium-oracle-connect-connect-api 8083:8083

# List connectors
curl http://localhost:8083/connectors

# Get connector status
curl http://localhost:8083/connectors/my-connector/status

# List connector plugins
curl http://localhost:8083/connector-plugins
```

## Customization Examples

### Change Number of Replicas

```bash
helm upgrade kafka-connect-prod . \
  --set kafkaConnect.replicas=3 \
  -n kraft-kafka-connect
```

### Change Resource Limits

```bash
helm upgrade kafka-connect-prod . \
  --set kafkaConnect.resources.limits.memory=4Gi \
  --set kafkaConnect.resources.limits.cpu=2000m \
  -n kraft-kafka-connect
```

### Use Different Kafka Cluster

```bash
helm upgrade kafka-connect-prod . \
  --set kafkaConnect.bootstrapServers=my-kafka-cluster:9092 \
  -n kraft-kafka-connect
```

### Update Custom Image

```bash
helm upgrade kafka-connect-prod . \
  --set kafkaConnect.image=kadirsahan/kafka-connect-strimzi-build:1.5 \
  -n kraft-kafka-connect
```

### Disable PostgreSQL Secret

```bash
helm upgrade kafka-connect-prod . \
  --set postgresSecret.enabled=false \
  -n kraft-kafka-connect
```

## Secrets Management

### PostgreSQL Credentials

The chart includes a template for PostgreSQL credentials:

```bash
# Encode password
echo -n 'mypassword' | base64

# Update values.yaml or pass via --set
helm install kafka-connect-prod . \
  --set postgresSecret.password=$(echo -n 'mypassword' | base64) \
  -n kraft-kafka-connect --create-namespace
```

Secret is mounted at `/mnt/postgres-credentials` and can be referenced in connector configs:

```yaml
database.password: ${file:/mnt/postgres-credentials/password}
```

### Using Existing Secrets

Disable the built-in secret and use your own:

```bash
# Create your own secret
kubectl create secret generic my-credentials \
  --from-literal=password=mypassword \
  -n kraft-kafka-connect

# Install without built-in secret
helm install kafka-connect-prod . \
  --set postgresSecret.enabled=false \
  -n kraft-kafka-connect --create-namespace
```

## Upgrading

### Upgrade Helm Release

```bash
# Upgrade with new values
helm upgrade kafka-connect-prod . -f values-prod.yaml -n kraft-kafka-connect

# Upgrade specific values
helm upgrade kafka-connect-prod . --set kafkaConnect.replicas=3 -n kraft-kafka-connect
```

### Update Image Version

```bash
helm upgrade kafka-connect-prod . \
  --set kafkaConnect.image=kadirsahan/kafka-connect-strimzi-build:1.5 \
  -n kraft-kafka-connect
```

## Monitoring

```bash
# Check KafkaConnect status
kubectl get kafkaconnect debezium-oracle-connect -n kraft-kafka-connect -o yaml

# Check pods
kubectl get pods -n kraft-kafka-connect

# View logs
kubectl logs -n kraft-kafka-connect debezium-oracle-connect-connect-0

# Check connectors (using CRD)
kubectl get kafkaconnector -n kraft-kafka-connect

# Check connector status
kubectl get kafkaconnector my-connector -n kraft-kafka-connect -o jsonpath='{.status}'
```

## Scaling

```bash
# Scale Kafka Connect workers
helm upgrade kafka-connect-prod . \
  --set kafkaConnect.replicas=5 \
  -n kraft-kafka-connect
```

## Troubleshooting

### Pods Not Starting

```bash
# Check pod events
kubectl describe pod <pod-name> -n kraft-kafka-connect

# Check operator logs
kubectl logs -n kraft-strimzi-operator deployment/strimzi-cluster-operator
```

### Connector Creation Fails

```bash
# Check KafkaConnector status
kubectl describe kafkaconnector my-connector -n kraft-kafka-connect

# Check connector logs via REST API
kubectl port-forward -n kraft-kafka-connect svc/debezium-oracle-connect-connect-api 8083:8083
curl http://localhost:8083/connectors/my-connector/status

# Check Kafka Connect logs
kubectl logs -n kraft-kafka-connect debezium-oracle-connect-connect-0 | grep ERROR
```

### Secret Not Found

```bash
# Verify secret exists
kubectl get secret postgres-credentials -n kraft-kafka-connect

# Check secret mount
kubectl describe pod debezium-oracle-connect-connect-0 -n kraft-kafka-connect | grep -A 5 Volumes
```

## Uninstalling

```bash
# Using Helm
helm uninstall kafka-connect-prod -n kraft-kafka-connect

# Delete namespace
kubectl delete namespace kraft-kafka-connect
```

**Warning**: This will delete all connector configurations and offsets.

## Values File Reference

See [values.yaml](values.yaml) for all available configuration options.

Key sections:
- `nameOverride`: Kafka Connect cluster name
- `namespace`: Target namespace
- `kafkaConnect.replicas`: Number of worker replicas
- `kafkaConnect.image`: Custom image with Debezium connectors
- `kafkaConnect.bootstrapServers`: Kafka cluster connection
- `kafkaConnect.config.*`: Kafka Connect configuration
- `kafkaConnect.resources.*`: Resource limits and requests
- `postgresSecret.*`: PostgreSQL credentials secret

## Related Repositories

- [strimzi-operator](https://github.com/kadirsahan/strimzi-operator) - Strimzi operator installation
- [kraft-kafka-server](https://github.com/kadirsahan/kraft-kafka-server) - Kafka cluster
- [argocd-gitops](https://github.com/kadirsahan/argocd-gitops) - ArgoCD installation
- [argocd-apps](https://github.com/kadirsahan/argocd-apps) - App-of-Apps GitOps repository

## Documentation

- [Strimzi Kafka Connect](https://strimzi.io/docs/operators/latest/deploying.html#kafka-connect-str)
- [Debezium Documentation](https://debezium.io/documentation/)
- [Debezium Connectors](https://debezium.io/documentation/reference/stable/connectors/index.html)
- [Helm Documentation](https://helm.sh/docs/)

## License

MIT

## Author

kadirsahan
