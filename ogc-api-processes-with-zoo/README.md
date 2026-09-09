# ZOO-Project OGC API Processes — deployment guide

This directory deploys a full **OGC API - Processes** stack (ZOO-Project DRU) on a local
Kubernetes cluster, together with a browser IDE, an S3-compatible object store and a CWL
runner. Deployment is driven by [Skaffold](https://skaffold.dev/) on top of Helm.

**Run every command in this document from this directory** (`ogc-api-processes-with-zoo/`).
There is no `skaffold.yaml` at the repository root, so running Skaffold from the parent
directory fails with `Skaffold config file skaffold.yaml not found`.

## What gets deployed

| Component | Release | Notes |
|---|---|---|
| ZOO-Project DRU | `zoo-project-dru` | Chart **0.10.4**, app version 0.2.48 |
| Code-server IDE | `eoap-zoo-project-coder` | Local chart, `../charts/coder` |
| LocalStack | `eoap-zoo-project-localstack` | S3 emulation, bucket `results` |
| PostgreSQL, RabbitMQ, Redis | templated by `zoo-project-dru` | job store, queue, cache |
| Calrissian | enabled via the coder chart | CWL execution on Kubernetes |

The code-server init script clones
[`gfenoy/ogc-api-processes-with-zoo`](https://github.com/gfenoy/ogc-api-processes-with-zoo)
into the workspace and builds a Python virtualenv at `/workspace/.venv`. See
[`files/init.sh`](files/init.sh).

## Prerequisites

Install and verify these first. Versions below are known to work.

```bash
kubectl version --client   # v1.36.1
helm version               # v4.2.4
skaffold version           # v2.24.0
docker version             # 29.7.2
jq --version               # required by cleanup.sh
```

On macOS everything is available from Homebrew:

```bash
brew install kubectl helm skaffold jq
```

> If Homebrew refuses to install because of an untrusted third-party tap, the failure is
> unrelated to this project. Either trust the tap or remove it, then retry.

### A running Kubernetes cluster

Skaffold needs a reachable cluster **before** you start. Either enable Kubernetes in
Docker Desktop (Settings → Kubernetes → *Enable Kubernetes*), or install and start
minikube. Verify:

```bash
kubectl config current-context
kubectl get nodes
```

If this errors with `dial tcp [::1]:8080: connect: connection refused`, no cluster is
running and every Skaffold command will fail.

### Helm repositories

```bash
helm repo add zoo-project https://zoo-project.github.io/charts/
helm repo add localstack https://helm.localstack.cloud
helm repo update
```

### LocalStack authentication token

Recent LocalStack images require an auth token to start, even for community features such
as S3. Without it the LocalStack pod exits with `exit code 55 — License activation failed`.

Get a token from your account at https://app.localstack.cloud and export it **in the
shell that runs Skaffold**:

```bash
export LOCALSTACK_AUTH_TOKEN="<PASTE-YOUR-OWN-TOKEN-HERE>"
```

`<PASTE-YOUR-OWN-TOKEN-HERE>` above is a placeholder, not a value that works. Replace the
whole thing, quotes included, with the token from your own LocalStack account. Copying the
placeholder verbatim gets you a container that starts and then dies with
`License activation failed`.

Skaffold injects it through `setValueTemplates`, so it is never written to the repository.
Treat the token as a credential: do not commit it, do not paste it into issues or chat
logs, and rotate it from the LocalStack console if it leaks.

## Pick the right profile: it comes down to the storage class

This is the single most common cause of failed deployments. The two supported local
clusters ship **different default storage classes**, and the charts must be told which one
to use.

| Cluster | Default storage class | Profile to use |
|---|---|---|
| minikube | `standard` | none (`skaffold dev`) |
| Docker Desktop | `hostpath` | **`macos`** (`skaffold dev -p macos`) |

Check yours with `kubectl get sc`.

The default profile sets no storage class at all, so the charts fall back to their own
default of `standard`. On Docker Desktop that class does not exist, the persistent volume
claims never bind, and pods stay stuck on
`0/1 nodes are available: pod has unbound immediate PersistentVolumeClaims`.

The `macos` profile exists precisely to patch every storage class to `hostpath`. Despite
its name it is about **Docker Desktop**, not about the CPU architecture.

## Quick start — Docker Desktop on Apple Silicon

This is the verified path on an arm64 Mac.

**1. Keep Rosetta enabled** in Docker Desktop (Settings → General → *Apple Virtualization
framework* + *Use Rosetta for x86_64/amd64 emulation*). The ZOO-Project images are amd64
only and run under emulation.

**2. Pre-pull the amd64 ZOO images.** The `macos` profile sets
`zoofpm.image.pullPolicy: Never` and `zookernel.image.pullPolicy: Never`, so Kubernetes
will **not** fetch these images itself. It reads them from the local Docker image store,
which Docker Desktop shares with its Kubernetes node. If they are absent the pods fail
with `ErrImageNeverPull`.

```bash
TAG=$(helm show values zoo-project/zoo-project-dru --version 0.10.4 \
      | grep -A4 '^zoofpm:' | grep 'tag:' | awk '{print $2}')

docker pull zooproject/zoo-project:$TAG --platform linux/amd64
docker pull zooproject/websocketd:67449315857b54bbc970f02c7aa4fd10a94721f0 --platform linux/amd64
```

The `--platform linux/amd64` flag is mandatory. Without it Docker pulls whatever the
manifest offers for arm64 and the ZOO pods will not start.

Deriving the tag from the **pinned** chart version keeps it consistent with what Helm
installs. Reading it from the `main` branch of the charts repository also works today but
will drift as soon as upstream moves ahead of 0.10.4.

**3. Deploy.**

```bash
export LOCALSTACK_AUTH_TOKEN="<PASTE-YOUR-OWN-TOKEN-HERE>"
skaffold dev -p macos
```

Deployment stabilises in roughly one to two minutes.

**Leave this terminal open.** The Skaffold process is what holds the port forwards open,
so use a second tab for any other command. Pressing `Ctrl+C` does not merely close the
forwards, it uninstalls the three Helm releases and deletes the whole stack. You will see
it happen as `release "zoo-project-dru" uninstalled` and two similar lines. Getting back
to a working environment then means running step 3 again from the start.

### What you repeat, and what you do only once

| Step | How often |
|---|---|
| Enabling Kubernetes in Docker Desktop | once, it comes back up with Docker |
| `brew install` | once |
| `helm repo add` | once |
| `docker pull` of the amd64 images | once, they stay in the local image store |
| `export LOCALSTACK_AUTH_TOKEN` | **every new shell**, `export` dies with the terminal |
| `cd` into this directory | every new shell |
| `skaffold dev -p macos` | every session |

### Expected warnings

These appear on a healthy deployment. Ignore them.

- `build target platforms "linux/amd64" do not match active kubernetes cluster node platforms "linux/arm64"` — expected, the stack is amd64 under Rosetta. There are no Skaffold build artifacts here, so the mismatch has no effect.
- `FailedToRetrieveImagePullSecret: Unable to retrieve some image pull secrets (kaniko-secret)` — the Calrissian service account references `kaniko-secret`, but `kaniko.enabled` is `False` in [`../charts/coder/values.yaml`](../charts/coder/values.yaml) so the secret is never created. The images are public and pods start normally.
- `client-side throttling, not priority and fairness` — kubectl rate limiting while Skaffold polls many resources at once.

### Adding the platform flags

You may also see this form:

```bash
skaffold dev -p macos --platform linux/amd64 --enable-platform-node-affinity=true
```

Those flags only affect images that **Skaffold itself builds**. This configuration builds
nothing (`No artifacts found to watch`), so they change nothing here. They are harmless
but not required.

## Quick start — minikube

```bash
minikube start
export LOCALSTACK_AUTH_TOKEN="<PASTE-YOUR-OWN-TOKEN-HERE>"
skaffold dev
```

No image pre-pull is needed: the default profile leaves `pullPolicy` at `IfNotPresent`,
so the cluster fetches images on its own.

## Access points

Skaffold prints the actual forwarded addresses. **Read that output rather than trusting
the table below**, because Skaffold silently picks the next free port when the configured
one is taken.

| Service | Configured local port | Notes |
|---|---|---|
| Code-server IDE | 8000 | **usually 8001 on Docker Desktop** |
| ZOO-Project API | 8080 | OGC API - Processes landing page |
| websocketd | 8888 | job status stream |

Docker Desktop's own backend listens on port 8000, so the code-server forward is almost
always bumped to **8001** on that platform. Look for the line:

```
Port forwarding service/code-server-service in namespace eoap-zoo-project, remote port 8080 -> http://localhost:8001
```

Confirm what is bound with `lsof -nP -iTCP -sTCP:LISTEN | grep 800`.

### Checking that the stack really answers

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/
curl -s http://localhost:8080/ogc-api/processes | jq -r '.processes[].id'
```

The first call returns `200` and redirects to `/ogc-api/api.html`, the interactive
OGC API - Processes documentation. The second lists the deployed processes; a freshly
deployed instance has exactly one, named `echo`.

Code-server answers `302` on `/`, which is its normal redirect, and websocketd answers
`404` to a plain HTTP request because it only speaks WebSocket. Neither is a fault.

### Reopening the forwards without redeploying

If the Skaffold process died but the pods are still running, recreate the forwards by hand
instead of redeploying the whole stack:

```bash
kubectl get pods -n eoap-zoo-project

kubectl port-forward -n eoap-zoo-project svc/zoo-project-dru-service 8080:80 &
kubectl port-forward -n eoap-zoo-project svc/code-server-service 8001:8080 &
kubectl port-forward -n eoap-zoo-project svc/zoo-project-dru-websocketd 8888:8888 &
```

Stop them again with `pkill -f 'kubectl port-forward'`.

## Profiles

### `macos` — Docker Desktop

Sets every storage class to `hostpath` and pins `zoofpm` / `zookernel` to
`imagePullPolicy: Never`. See the quick start above.

### `keda` — event-driven autoscaling

```bash
skaffold dev -p keda
```

Installs Kyverno first (so its CRDs exist), then enables KEDA scaling of ZOO-FPM workers
from PostgreSQL and RabbitMQ metrics, plus an eviction controller that protects workers
with running jobs.

### `argo` — Argo Workflows orchestration

```bash
kubectl create secret generic s3-service -n eoap-zoo-project \
  --from-literal=rootUser=test \
  --from-literal=rootPassword=test \
  --dry-run=client -o yaml | kubectl apply -f -

skaffold dev -p argo
```

Swaps in [`values_argo.yaml`](values_argo.yaml) and a different cookiecutter service
template. Adds two port forwards on top of the defaults: the Argo Workflows UI on **2746**
and LocalStack S3 on **9000**.

Note that the namespace must exist before the secret can be created, so run this after a
first deployment or create the namespace yourself.

### Combining profiles

`argo,macos` works:

```bash
skaffold dev -p argo,macos
```

> **`keda,macos` is currently broken — do not use it.**
>
> The `macos` profile patches Helm releases **by index** (`/deploy/helm/releases/0`,
> `/releases/1`). The `keda` profile inserts Kyverno at index 0, shifting every other
> release down by one. The storage class patches then land on the wrong releases:
> Kyverno receives `persistence.storageClass`, ZOO-Project receives
> `coder.storageClassName`, and the coder release receives nothing at all, keeping its
> `standard` default. On Docker Desktop the code-server volume never binds.
>
> Verify for yourself with `skaffold diagnose -p keda,macos --yaml-only`.
>
> Fixing this means making the `macos` patches target releases by name instead of by
> index, or duplicating them into a dedicated `keda-macos` profile.

## Cleaning up and switching profiles

Persistent volume claims are **immutable** once bound. Switching between profiles that
use different storage classes therefore fails with:

```
Error: INSTALLATION FAILED: server-side apply failed for object
eoap-zoo-project/zoo-project-dru-processing-services /v1, Kind=PersistentVolumeClaim:
PersistentVolumeClaim "zoo-project-dru-processing-services" is invalid:
spec: Forbidden: spec is immutable after creation
- "StorageClassName": "hostpath",
+ "StorageClassName": "standard",
```

Always tear down before switching:

```bash
./cleanup.sh
```

The script stops Skaffold, removes the Helm releases, strips finalizers from KEDA, Kyverno
and Argo resources, deletes CRDs, PVCs and PVs, and force-finalizes stuck namespaces. It
requires `jq`.

For a plain redeploy of the same profile, deleting the namespace is enough and much
faster:

```bash
kubectl delete ns eoap-zoo-project
```

> `cleanup.sh` deletes **all** persistent volumes in the cluster, not only those belonging
> to this project. Do not run it on a cluster shared with other work.

## Troubleshooting

**`Skaffold config file skaffold.yaml not found`**
You are in the wrong directory. Run Skaffold from `ogc-api-processes-with-zoo/`.

**`kubernetes cluster unreachable: Get "http://localhost:8080/version"`**
No Kubernetes cluster is running, or `kubectl` has no valid context. Start Docker
Desktop's Kubernetes or minikube, then check `kubectl get nodes`.

**`profile selection ["macos"] did not match those defined in any configurations`**
You are running Skaffold against a checkout that has no `macos` profile. The upstream
`eoap/dev-platform-eoap` repository does not define one — it only exists in this fork.
Do **not** re-clone the repository from inside this directory; a nested
`ogc-api-processes-with-zoo/dev-platform-eoap/` checkout is the usual cause. Delete it and
run from the outer checkout.

**Pods stuck on `pod has unbound immediate PersistentVolumeClaims`**
Storage class mismatch. Compare `kubectl get sc` with the profile you selected, and see
the profile table above. Inspect with
`kubectl get pvc -n eoap-zoo-project` — bound claims show the class actually in use.

**`ErrImageNeverPull` on zoofpm or zookernel**
The `macos` profile expects those amd64 images to already exist locally. Run the
`docker pull ... --platform linux/amd64` step, then confirm with
`docker image inspect <image> --format '{{.Os}}/{{.Architecture}}'`, which must print
`linux/amd64`.

**LocalStack restarts in a loop, then Skaffold rolls the whole deployment back**
`Back-off restarting failed container localstack` followed by three
`release ... uninstalled` lines means `LOCALSTACK_AUTH_TOKEN` is either missing or wrong.
Both are common: the variable dies with every new terminal, and the placeholder in this
document is easy to paste by accident.

Skaffold gives you **no warning whatsoever** for either case. A missing variable is
rendered as an empty value and the command still exits successfully:

```yaml
- name: LOCALSTACK_AUTH_TOKEN
  value:
```

LocalStack then fails license activation, exits with code 55 and crash-loops.

The pod logs tell you which of the two mistakes you made. `The credentials defined in your
environment are invalid` means a token was passed but is wrong, typically the
documentation placeholder copied as-is, or a token that has since been rotated. No
credentials message at all, with an empty value in the rendered manifest, means the
variable was never exported.

Check the variable before every deployment, and read the pod logs if it happens:

```bash
echo "${LOCALSTACK_AUTH_TOKEN:-MISSING}"
kubectl logs -n eoap-zoo-project -l app.kubernetes.io/name=localstack --tail=30
```

To stop losing it, persist the export in `~/.zshrc` instead of retyping it.

**`Insufficient memory` on code-server when you redeploy**
You launched Skaffold while a previous stack was still running, so Helm performed an
upgrade rather than an install. The coder Deployment has no `strategy` set, so Kubernetes
applies the default rolling update: for a single replica `maxSurge` rounds up to one and
`maxUnavailable` rounds down to zero, meaning the replacement pod must be scheduled
**before** the old one is stopped. Two code-server pods requesting 4 Gi each do not fit in
the memory Docker Desktop allocates.

Either tear the stack down before relaunching, or give Docker Desktop more memory in
Settings → Resources. Compare the two numbers:

```bash
kubectl get nodes -o jsonpath='{.items[0].status.allocatable.memory}'
kubectl describe node | sed -n '/Allocated resources/,/Events/p'
```

**Namespace stuck in `Terminating`**
Run `./cleanup.sh`, which removes the blocking finalizers.

**Port already in use**
Skaffold shifts to the next free port and prints the real one. Check occupancy with
`lsof -nP -iTCP -sTCP:LISTEN`.

**Notebook Python kernel hangs or shows import errors**
The workspace volume persists across redeploys and may hold a virtualenv built for a
different architecture. Recreate it:

```bash
kubectl -n eoap-zoo-project delete pvc code-server-pvc
```

Then redeploy so `init.sh` rebuilds `/workspace/.venv`.

## Note on the code-server image architecture

Earlier revisions of this document asked you to build `eoap-coder` locally for arm64 and
patch `skaffold.yaml`. **That workaround is obsolete — skip it.**

The GitHub Actions workflow now publishes a multi-architecture image
(see [`../.github/workflows/build.yaml`](../.github/workflows/build.yaml), which builds
`linux/amd64,linux/arm64`), and both [`skaffold.yaml`](skaffold.yaml) and
[`../charts/coder/values.yaml`](../charts/coder/values.yaml) point back at
`ghcr.io/eoap/dev-platform-eoap/eoap-coder:0.1.0`.

On Apple Silicon, Kubernetes pulls the arm64 variant natively. Confirm it on a running
deployment:

```bash
kubectl exec -n eoap-zoo-project deploy/code-server-deployment -- uname -m   # aarch64
kubectl exec -n eoap-zoo-project deploy/zoo-project-dru-zoofpm -- uname -m   # x86_64
```

The IDE runs native arm64 while the ZOO workers run amd64 under Rosetta, which is the
intended arrangement.

## Configuration files

| File | Purpose |
|---|---|
| [`skaffold.yaml`](skaffold.yaml) | releases, profiles and port forwards |
| [`values.yaml`](values.yaml) | Helm values for the default and KEDA deployments |
| [`values_argo.yaml`](values_argo.yaml) | Helm values for the Argo Workflows deployment |
| [`cleanup.sh`](cleanup.sh) | full teardown, needed when switching profiles |
| [`files/init.sh`](files/init.sh) | code-server bootstrap: clone, venv, S3 bucket |
| [`_skaffold.yaml`](_skaffold.yaml) | superseded earlier configuration, not used by Skaffold |

## More information

- [ZOO-Project documentation](https://zoo-project.github.io/docs/)
- [ZOO-Project Helm charts](https://github.com/ZOO-Project/charts)
- [OGC API - Processes specification](https://ogcapi.ogc.org/processes/)
