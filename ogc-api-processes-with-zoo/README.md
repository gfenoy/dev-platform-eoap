# ZOO-Project OGC API Processes Deployment

This directory contains the deployment configuration for ZOO-Project with OGC API Processes using Skaffold and Helm.

## Prerequisites

Before deploying, ensure you have the following tools installed:
- **kubectl** - Kubernetes command-line tool
- **helm** (v3+) - Kubernetes package manager
- **skaffold** - Kubernetes development tool

Add the required Helm repositories:

```bash
helm repo add zoo-project https://zoo-project.github.io/charts/
helm repo add localstack https://helm.localstack.cloud
helm repo update
```

### LocalStack authentication token

Recent LocalStack versions **require an auth token to start the container**, even for
community features such as S3. Without it the LocalStack pod exits with
`exit code 55 — License activation failed`.

1. Create a free account and copy your token from https://app.localstack.cloud
   (the free *Hobby* plan is enough; *Students* / *OSS* options also exist).
2. Export the token in the shell **before** running Skaffold — it is injected into the
   LocalStack release via `setValueTemplates`, so it is never committed to the repo:

```bash
export LOCALSTACK_AUTH_TOKEN="ls-xxxxxxxx"
```

## Deployment Profiles

### Standard Installation

Deploy ZOO-Project with Calrissian workflow engine:

```bash
skaffold dev
```

This profile includes:
- ZOO-Project DRU (v0.8.2)
- Calrissian CWL runner
- LocalStack S3 for storage
- Code-server development environment
- RabbitMQ message queue
- PostgreSQL database
- Redis cache

**Access points:**
- Code-server: http://localhost:8000
- ZOO-Project API: http://localhost:8080
- WebSocket: http://localhost:8888

### KEDA Autoscaling Profile

Deploy with Kubernetes Event-Driven Autoscaling (KEDA) and Kyverno policy enforcement:

```bash
skaffold dev -p keda
```

Additional features:
- **KEDA autoscaling** based on PostgreSQL and RabbitMQ metrics
- **Kyverno** policy engine for pod protection
- **Eviction controller** to protect active workers from termination
- Automatic scaling of ZOO-FPM workers based on queue depth

This profile is ideal for production environments requiring dynamic scaling.

### Argo Workflows Profile

Deploy with Argo Workflows for advanced workflow orchestration:

```bash
# Create S3 credentials secret first
kubectl create secret generic s3-service -n eoap-zoo-project \
  --from-literal=rootUser=test \
  --from-literal=rootPassword=test \
  --dry-run=client -o yaml | kubectl apply -f -

# Deploy with Argo profile
skaffold dev -p argo
```

Additional features:
- **Argo Workflows** (v3.7.1) for workflow orchestration
- Workflow artifact storage in LocalStack S3
- Namespaced deployment with instance isolation
- Workflow TTL and pod garbage collection
- Argo Workflows UI for workflow visualization

**Additional access points:**
- Argo Workflows UI: http://localhost:2746
- LocalStack S3: http://localhost:9000

### macOS / ARM Processor Support

For Apple Silicon or other ARM-based systems:

```bash
skaffold dev -p macos
```

This profile configures `hostpath` storage class compatible with Docker Desktop on macOS.

Keep **Rosetta enabled** in Docker Desktop (Settings → General → *Apple Virtualization
framework* + *Use Rosetta for x86_64/amd64 emulation*). The rest of the stack ships as
amd64 and relies on Rosetta.

#### Temporary: build the code-server image natively in arm64

> ⚠️ **Temporary workaround.** The `eoap-coder` image is currently published for amd64
> only. Under Rosetta emulation the Jupyter Python kernel (`ipykernel`) **deadlocks on
> startup**, which makes notebooks unusable (the terminal still works fine). Running the
> code-server pod as a **native arm64** image avoids the emulation entirely.
>
> Once the multi-arch image is published upstream (PR pending on
> [`eoap/dev-platform-eoap`](https://github.com/eoap/dev-platform-eoap)), this whole
> section can be ignored: point `coder.coderImage` back to
> `ghcr.io/eoap/dev-platform-eoap/eoap-coder:0.1.0` and Kubernetes will pull the arm64
> variant natively.

Build the arm64 image locally (the `Dockerfile.coder` is already multi-arch aware) and
load it into Docker Desktop's image store:

```bash
cd ../container/coder
docker buildx build --builder desktop-linux --platform linux/arm64 --load \
  -t eoap-coder:0.1.0-arm64 -f Dockerfile.coder .
cd -
```

Update the `skaffold.yaml` to reference `eoap-coder:0.1.0-arm64` in place of
the default `ghcr.io/eoap/dev-platform-eoap/eoap-coder:0.1.0`.

Then deploy as usual (don't forget the LocalStack token above):

```bash
export LOCALSTACK_AUTH_TOKEN="ls-xxxxxxxx"
skaffold dev -p macos --platform linux/amd64 --enable-platform-node-affinity=true
```

> If notebooks show Python import errors after switching architecture, the persistent
> workspace still holds a venv built for the previous arch. Recreate it:
> `kubectl -n eoap-zoo-project delete pvc code-server-pvc`, then redeploy so `init.sh`
> rebuilds `/workspace/.venv` in arm64.


## Cleanup

When switching between profiles or redeploying, use the cleanup script to ensure all resources are properly removed:

```bash
./cleanup.sh
```

The cleanup script will:
- Stop running Skaffold processes
- Remove Helm releases (ZOO-Project, Kyverno, LocalStack)
- Clean up KEDA and Argo Workflows resources
- Remove Custom Resource Definitions (CRDs)
- Force removal of stuck namespaces and persistent volumes
- Validate complete cleanup

**Note:** This script is particularly important when switching between KEDA and Argo profiles to avoid resource conflicts.

## Combining Profiles

Profiles can be combined for specific deployment scenarios:

```bash
# KEDA + macOS
skaffold dev -p keda,macos

# Argo + macOS
skaffold dev -p argo,macos
```

## Troubleshooting

### Namespace stuck in Terminating state
Run the cleanup script which handles finalizer removal:
```bash
./cleanup.sh
```

### Port conflicts
Ensure no other services are using the default ports (8000, 8080, 8888, 2746, 9000).

### Persistent Volume issues
The cleanup script removes all PVs. If issues persist, manually check:
```bash
kubectl get pv
kubectl delete pv <pv-name> --grace-period=0 --force
```

## Configuration Files

- **skaffold.yaml** - Main deployment configuration with all profiles
- **values.yaml** - Default Helm values for standard/KEDA deployments
- **values_argo.yaml** - Helm values for Argo Workflows deployment
- **cleanup.sh** - Resource cleanup script

## More Information

For detailed information about ZOO-Project, visit:
- [ZOO-Project Documentation](https://zoo-project.github.io/docs/)
- [ZOO-Project Helm Charts](https://github.com/ZOO-Project/charts)

