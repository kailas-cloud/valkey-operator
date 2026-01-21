# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

A Kubernetes operator for provisioning [Valkey](https://valkey.io) (Redis-compatible) clusters. Built with Kubebuilder v4 and controller-runtime.

## Build & Development Commands

```bash
# Build binaries (manager + sidecar)
make build

# Run tests (unit/integration with envtest)
make test

# Run e2e tests (requires running cluster)
make test-e2e

# Run linter
make lint

# Fix lint issues
make lint-fix

# Generate CRDs and RBAC manifests
make manifests

# Generate DeepCopy methods after modifying types
make generate

# Build all container images
make docker-build

# Quick local development setup (minikube + operator + sample CR)
make quickstart

# With TLS and Prometheus enabled
make quickstart TLS=1 PROMETHEUS=1

# Run controller locally against cluster
make run

# Install CRDs into cluster
make install
```

### Local Development Workflow

```bash
# Start minikube cluster
make minikube

# Proxy to local registry (in separate terminal)
make registry-proxy

# Build and deploy locally
export REGISTRY=localhost:5000 VERSION=dev
make docker-build docker-push build-installer
kubectl apply -f dist/install.yaml
```

## Architecture

### CRD: `Valkey` (kailas.cloud/v1)
Defined in `api/v1/valkey_types.go`. Key spec fields:
- `shards` (nodes): Number of primary nodes
- `replicas`: Replicas per shard
- `tls`: Enable TLS with cert-manager
- `externalAccess`: LoadBalancer or Proxy mode for external connectivity
- `storage`: PersistentVolumeClaim spec

### Controller
`internal/controller/valkey_controller.go` - Main reconciliation loop that manages:
- StatefulSet for Valkey pods
- Services (headless, ClusterIP, LoadBalancer)
- ConfigMaps with Valkey configuration
- Secrets for authentication
- Certificates via cert-manager
- ServiceMonitor for Prometheus
- PodDisruptionBudget
- Optional Envoy proxy for external access

`internal/controller/cluster.go` - Types for modeling Valkey cluster topology (shards, nodes, primaries/replicas)

### Sidecar
`cmd/sidecar/` - Runs alongside Valkey in pods for:
- Prometheus metrics export
- Cluster bootstrapping

### Image Configuration
Default images are set at build time via ldflags in Makefile. Override at runtime via:
- `ValkeySpec.image`
- `ValkeySpec.exporterImage`
- Global config in `cfg/config.go`

## Testing

Uses Ginkgo/Gomega framework:
- Unit/integration tests: `internal/controller/*_test.go` (use envtest)
- E2E tests: `test/e2e/` (require real cluster)

Run single test:
```bash
go test ./internal/controller/... -run TestSpecificName -v
```

Integration tests are tagged with `//go:build integration`.

## Key Dependencies

- `sigs.k8s.io/controller-runtime` - Kubernetes controller framework
- `github.com/valkey-io/valkey-go` - Valkey client for cluster management
- `github.com/cert-manager/cert-manager` - TLS certificate management
- `github.com/prometheus-operator/prometheus-operator` - Metrics/monitoring

## Code Generation

After modifying `api/v1/valkey_types.go`:
```bash
make generate   # DeepCopy methods
make manifests  # CRDs, RBAC
```

## Container Images

Three images are built:
- `valkey-operator` (controller) - `Dockerfile.controller`
- `valkey-sidecar` - `Dockerfile.sidecar`
- `valkey` (server) - `Dockerfile.valkey`
