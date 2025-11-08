# Kafka Connect with Debezium

Kafka Connect cluster with Debezium connectors using Strimzi operator - deployed via ArgoCD.

## Overview

This repository contains Kubernetes manifests for deploying Kafka Connect using Strimzi's KafkaConnect custom resource. The cluster uses a custom image with Debezium connectors pre-installed for CDC (Change Data Capture).

## Features

- ✅ **Debezium Connectors**: Pre-installed in custom image
- ✅ **Strimzi Integration**: Managed by Strimzi operator
- ✅ **Connector Resources**: Dynamic connector management via CRDs
- ✅ **High Availability**: 2 replicas for redundancy
- ✅ **Persistent Configuration**: Topics for configs, offsets, and status
- ✅ **GitOps Ready**: Managed via ArgoCD

## Prerequisites

- Kubernetes cluster (v1.19+)
- Strimzi Kafka Operator installed (v0.48.0+)
- Kafka cluster running (kraft-kafka-cluster)
- ArgoCD (for GitOps deployment)
- postgres-credentials secret (if using PostgreSQL connectors)

## Repository Structure

```
kafka-connect/
├── namespace.yaml          # kraft-kafka-connect namespace
├── kafka-connect.yaml      # KafkaConnect CR with Debezium
└── README.md               # This file
```

## Configuration

### Kafka Connect Specification

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

### Resources

```yaml
requests:
  memory: 512Mi
  cpu: 300m
limits:
  memory: 2Gi
  cpu: 1000m
```

### Internal Topics

- **connect-cluster-configs**: Connector configurations
- **connect-cluster-offsets**: Connector offsets
- **connect-cluster-status**: Connector status

**Replication Factor**: 1 (suitable for local/dev)

## Installation

### Using ArgoCD (Recommended)

This cluster is designed to be deployed via ArgoCD:

```bash
# Deployed automatically via app-of-apps pattern
kubectl apply -f https://raw.githubusercontent.com/kadirsahan/argocd-apps/main/app-of-apps.yaml
```

### Manual Installation

```bash
# Clone the repository
git clone https://github.com/kadirsahan/kafka-connect.git
cd kafka-connect

# Create namespace
kubectl apply -f namespace.yaml

# Create postgres credentials secret (if needed)
kubectl create secret generic postgres-credentials \
  --from-literal=password=yourpassword \
  -n kraft-kafka-connect

# Deploy Kafka Connect (requires Strimzi operator)
kubectl apply -f kafka-connect.yaml
```

## Verify Installation

```bash
# Check namespace
kubectl get ns kraft-kafka-connect

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

## Example Connectors

### Debezium Oracle CDC

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  name: oracle-cdc
  namespace: kraft-kafka-connect
  labels:
    strimzi.io/cluster: debezium-oracle-connect
spec:
  class: io.debezium.connector.oracle.OracleConnector
  tasksMax: 1
  config:
    database.hostname: oracle-db.default.svc.cluster.local
    database.port: "1521"
    database.user: debezium
    database.password: ${file:/mnt/postgres-credentials/password}
    database.dbname: ORCLCDB
    database.server.name: oracle-server
    table.include.list: ORCLPDB1.DEBEZIUM.CUSTOMERS
    database.history.kafka.bootstrap.servers: kraft-kafka-cluster-kafka-bootstrap.kraft-kafka.svc.cluster.local:9092
    database.history.kafka.topic: schema-changes.oracle
```

### Debezium JDBC Sink

```yaml
apiVersion: kafka.strimzi.io/v1alpha1
kind: KafkaConnector
metadata:
  name: jdbc-sink
  namespace: kraft-kafka-connect
  labels:
    strimzi.io/cluster: debezium-oracle-connect
spec:
  class: io.debezium.connector.jdbc.JdbcSinkConnector
  tasksMax: 1
  config:
    topics: oracle-server.ORCLPDB1.DEBEZIUM.CUSTOMERS
    connection.url: jdbc:postgresql://postgres-db:5432/targetdb
    connection.username: postgres
    connection.password: ${file:/mnt/postgres-credentials/password}
    insert.mode: upsert
    delete.enabled: true
    primary.key.mode: record_key
    schema.evolution: basic
    database.time_zone: UTC
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
kubectl patch kafkaconnect debezium-oracle-connect -n kraft-kafka-connect \
  --type='merge' -p '{"spec":{"replicas":3}}'
```

## Secrets Management

The configuration uses Kubernetes secrets for sensitive data:

```bash
# Create postgres credentials
kubectl create secret generic postgres-credentials \
  --from-literal=password=mypassword \
  -n kraft-kafka-connect

# Secret is mounted at /mnt/postgres-credentials
# Reference in connector config: ${file:/mnt/postgres-credentials/password}
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

## Upgrading

### Update Kafka Connect Image

```bash
kubectl patch kafkaconnect debezium-oracle-connect -n kraft-kafka-connect \
  --type='merge' -p '{"spec":{"image":"kadirsahan/kafka-connect-strimzi-build:1.5"}}'
```

### Rolling Restart

```bash
kubectl rollout restart statefulset/debezium-oracle-connect-connect -n kraft-kafka-connect
```

## Uninstalling

```bash
# Delete all connectors first
kubectl delete kafkaconnector --all -n kraft-kafka-connect

# Delete Kafka Connect
kubectl delete kafkaconnect debezium-oracle-connect -n kraft-kafka-connect

# Delete namespace
kubectl delete namespace kraft-kafka-connect
```

**Warning**: This will delete all connector configurations and offsets.

## Related Repositories

- [strimzi-operator](https://github.com/kadirsahan/strimzi-operator) - Strimzi operator installation
- [kraft-kafka-server](https://github.com/kadirsahan/kraft-kafka-server) - Kafka cluster
- [argocd-gitops](https://github.com/kadirsahan/argocd-gitops) - ArgoCD installation
- [argocd-apps](https://github.com/kadirsahan/argocd-apps) - App-of-Apps GitOps repository

## Documentation

- [Strimzi Kafka Connect](https://strimzi.io/docs/operators/latest/deploying.html#kafka-connect-str)
- [Debezium Documentation](https://debezium.io/documentation/)
- [Debezium Connectors](https://debezium.io/documentation/reference/stable/connectors/index.html)

## License

MIT

## Author

kadirsahan
