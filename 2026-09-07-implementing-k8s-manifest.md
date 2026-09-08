# Kubernetes Debugging and Partial Restart Cheat Sheet

This cheat sheet is based on the problems that appeared during our live Kubernetes session: backend crashes, missing environment variables, Jobs created in the wrong namespace, PVCs stuck in `Pending`, wrong `StorageClass`, health probe problems, Service/container port confusion, and the tendency to redeploy the whole namespace instead of fixing only the affected resource.

## Quick Command Reference

### Cluster and resource overview

```bash
kubectl get pods -n dev
```

Show Pods in the `dev` namespace.

```bash
kubectl get pods -n dev -o wide
```

Show Pods with additional information such as node and IP.

```bash
kubectl get pods -n dev -w
```

Watch Pod state changes in real time.

```bash
kubectl get all -n dev
```

Show the most common workload and Service resources.

```bash
kubectl get pods -A
```

Show Pods in all namespaces.

```bash
kubectl get jobs -A
```

Find Jobs across all namespaces.

```bash
kubectl get events -n dev --sort-by=.lastTimestamp
```

Show recent Kubernetes events in chronological order.

### Diagnose a Pod

```bash
kubectl describe pod <pod> -n dev
```

Show Pod configuration, status, probes, mounts, container state, and Events.

```bash
kubectl logs <pod> -n dev
```

Show application logs from the current container instance.

```bash
kubectl logs <pod> -n dev --previous
```

Show logs from the previous crashed container instance.

```bash
kubectl logs -f <pod> -n dev
```

Follow Pod logs in real time.

```bash
kubectl logs deployment/backend -n dev
```

Show logs from a Pod managed by the backend Deployment.

```bash
kubectl logs -f deployment/backend -n dev
```

Follow backend logs.

### Restart and rollout

```bash
kubectl rollout restart deployment/backend -n dev
```

Restart only the backend Deployment.

```bash
kubectl rollout status deployment/backend -n dev
```

Wait for and inspect rollout progress.

```bash
kubectl rollout history deployment/backend -n dev
```

Show previous Deployment revisions.

```bash
kubectl rollout undo deployment/backend -n dev
```

Rollback to the previous Deployment revision.

```bash
kubectl delete pod <pod> -n dev
```

Delete one Pod; its controller normally creates a replacement.

```bash
kubectl scale deployment/backend --replicas=0 -n dev
```

Temporarily stop backend Pods.

```bash
kubectl scale deployment/backend --replicas=1 -n dev
```

Start backend again.

### Work inside a container

```bash
kubectl exec -it <pod> -n dev -- sh
```

Open a shell inside a running container.

```bash
kubectl exec -it deployment/backend -n dev -- sh
```

Open a shell inside a Pod belonging to the backend Deployment.

```bash
kubectl exec deployment/backend -n dev -- env
```

Show environment variables inside the container.

```bash
kubectl exec deployment/backend -n dev -- printenv PROJECT_NAME
```

Check one environment variable.

### Temporary debugging Pod

```bash
kubectl run debug \
  --rm -it \
  --restart=Never \
  --image=curlimages/curl \
  -n dev \
  -- sh
```

Create a temporary Pod for network and HTTP debugging.

```bash
curl http://backend
```

Test a Service from inside the cluster.

### ConfigMaps

```bash
kubectl get configmap -n dev
```

List ConfigMaps.

```bash
kubectl get configmap backend-configmap -n dev -o yaml
```

Show the effective ConfigMap.

```bash
kubectl describe configmap backend-configmap -n dev
```

Show ConfigMap information in human-readable form.

### Secrets

```bash
kubectl get secrets -n dev
```

List Secrets in the namespace.

```bash
kubectl describe secret <secret-name> -n dev
```

Inspect Secret metadata without normally printing decoded values.

### Services and networking

```bash
kubectl get svc -n dev
```

List Services.

```bash
kubectl describe svc backend -n dev
```

Show backend Service ports, selectors, and endpoints.

```bash
kubectl get endpoints backend -n dev
```

Check which Pod IPs the Service currently targets.

```bash
kubectl get endpointslices -n dev
```

Inspect modern Service endpoint discovery resources.

```bash
kubectl get pods -n dev --show-labels
```

Show Pod labels for comparison with Service selectors.

```bash
kubectl port-forward svc/backend 8080:80 -n dev
```

Forward local port `8080` to Service port `80`.

### Jobs

```bash
kubectl get jobs -n dev
```

List Jobs.

```bash
kubectl describe job prestart -n dev
```

Inspect the prestart Job.

```bash
kubectl logs job/prestart -n dev
```

Show logs from the Job.

```bash
kubectl delete job prestart -n dev
```

Delete the completed or failed Job so it can be created again.

### Storage

```bash
kubectl get pvc -n dev
```

List PersistentVolumeClaims.

```bash
kubectl describe pvc postgres-vol -n dev
```

Inspect why a PVC is `Pending`, `Bound`, or failing.

```bash
kubectl get pv
```

List PersistentVolumes.

```bash
kubectl get storageclass
```

List available StorageClasses.

```bash
kubectl get sc
```

Short form of the previous command.

### Kustomize and deployment

```bash
kubectl kustomize k8s/overlays/dev
```

Render the final YAML produced by Kustomize without applying it.

```bash
kubectl diff -k k8s/overlays/dev
```

Show what would change in the cluster.

```bash
kubectl apply -k k8s/overlays/dev
```

Apply the Kustomize overlay.

### Inspect Kubernetes API fields

```bash
kubectl explain deployment
```

Show documentation for a Kubernetes resource.

```bash
kubectl explain deployment.spec
```

Explain a specific section.

```bash
kubectl explain deployment.spec.template.spec.containers.livenessProbe
```

Explain a deeply nested field.

```bash
kubectl explain pvc.spec.storageClassName
```

Explain the PVC StorageClass field.

### Temporary live modifications

```bash
kubectl edit deployment backend -n dev
```

Edit the live Deployment.

```bash
kubectl edit configmap backend-configmap -n dev
```

Edit the live ConfigMap.

Use these mainly for troubleshooting. Permanent changes should go into Git.

## Detailed Explanation

### 1. The most important idea: almost never delete the whole namespace

During the session, several local problems required debugging:
- missing `PROJECT_NAME`;
- a Job created in `default` instead of `dev`;
- registry credentials existing only in `dev`;
- `postgres-vol` PVC stuck in `Pending`;
- incorrect `storageClassName`;
- backend `CrashLoopBackOff`;
- health probes killing the backend;
- confusion between Service port and container port.

The dangerous troubleshooting model is:

```text
Something is broken
        ↓
kubectl delete namespace dev
        ↓
kubectl create namespace dev
        ↓
kubectl apply -k ...
```

For normal Kubernetes operations, this is far too destructive.

It is similar to:

```text
There is an application error
        ↓
Reinstall the operating system
```

A namespace should normally be deleted only when you intentionally want to completely destroy an environment.

Instead, think about the individual resources:

```text
Namespace
├── Deployment backend
│   └── ReplicaSet
│       └── Pod
│
├── Deployment frontend
│   └── Pod
│
├── Deployment postgres
│   └── Pod
│
├── Job prestart
├── Service backend
├── ConfigMap backend-config
├── Secret backend-secret
└── PVC postgres-vol
```

If only the backend is broken:

```text
Do not touch:
- frontend
- postgres
- PVC
- namespace
- unrelated Secrets

Work with:
- backend Deployment
- backend Pods
- backend ConfigMap
- backend Secret
```

## 2. First command for almost every problem

Start with:

```bash
kubectl get pods -n dev
```

Better:

```bash
kubectl get pods -n dev -o wide
```

During live debugging:

```bash
kubectl get pods -n dev -w
```

`-w` means `--watch`.

Example:

```text
NAME                       READY   STATUS
backend-7d9c...             0/1     CrashLoopBackOff
frontend-6b...              1/1     Running
postgres-deployment-...     1/1     Running
```

This already tells us something important:

```text
frontend  → healthy
postgres  → healthy
backend   → broken
```

Therefore, there is no reason to destroy PostgreSQL, frontend, or the namespace.

## 3. The basic Kubernetes troubleshooting triangle

For Pod problems, remember these three commands:

```bash
kubectl get pods -n dev
kubectl describe pod <pod> -n dev
kubectl logs <pod> -n dev
```

They answer three different questions.

### `kubectl get`

Answers:

> What is happening?

```bash
kubectl get pods -n dev
```

Typical states:

```text
Pending
Running
Completed
CrashLoopBackOff
ImagePullBackOff
Error
Terminating
```

### `kubectl describe`

Answers:

> Why is Kubernetes doing this?

```bash
kubectl describe pod backend-xxx -n dev
```

Pay particular attention to:

```text
State:
Last State:
Reason:
Restart Count:
Conditions:
Events:
```

The `Events` section is often extremely important.

During our session, storage diagnostics showed:

```text
ProvisioningFailed
storageclass.storage.k8s.io "local-path" not found
```

That immediately points to a storage configuration problem rather than an application problem.

### `kubectl logs`

Answers:

> What does the application itself report?

```bash
kubectl logs backend-xxx -n dev
```

For a Deployment:

```bash
kubectl logs deployment/backend -n dev
```

Live output:

```bash
kubectl logs -f deployment/backend -n dev
```

This is where errors such as the missing Pydantic setting become visible:

```text
PROJECT_NAME
Field required
```

## 4. One of the most useful missing commands: `--previous`

For `CrashLoopBackOff`, Kubernetes may be doing this:

```text
start container
      ↓
application crashes
      ↓
restart container
      ↓
application crashes
      ↓
restart container
```

If you run:

```bash
kubectl logs <pod> -n dev
```

you are looking at the current container instance.

Sometimes it has barely started and contains little useful information.

Use:

```bash
kubectl logs <pod> -n dev --previous
```

Example:

```bash
kubectl logs backend-7d96c7c8b9-x7t2m \
  -n dev \
  --previous
```

This shows logs from the previous crashed instance.

For `CrashLoopBackOff`, this should be one of your standard commands.

## 5. Inspect Events for the whole namespace

Instead of describing every resource separately, inspect namespace events:

```bash
kubectl get events -n dev \
  --sort-by=.lastTimestamp
```

Common useful events include:

```text
FailedScheduling
FailedMount
FailedPull
BackOff
Unhealthy
ProvisioningFailed
Pulling
Pulled
Created
Started
Killing
```

This often tells you where to look before you even inspect application logs.

## 6. Restart only the backend

This is probably one of the most important commands that was missing from your workflow.

```bash
kubectl rollout restart deployment/backend -n dev
```

Then:

```bash
kubectl rollout status deployment/backend -n dev
```

Conceptually:

```text
Deployment
    │
    ├─ old Pod
    │
    └─ new Pod
          ↓
       becomes Ready
          ↓
       old Pod removed
```

The rest of the environment remains unchanged.

PostgreSQL is not restarted.

Frontend is not restarted.

PVCs are not deleted.

Namespace is not deleted.

## 7. Delete only one Pod

If a Pod belongs to a Deployment:

```bash
kubectl delete pod backend-xxxxx -n dev
```

The Deployment controller notices:

```text
desired replicas = 1
actual replicas  = 0
```

and creates a replacement.

So:

```bash
kubectl delete pod
```

does not necessarily mean:

```text
delete application
```

For a controller-managed Pod, it is often effectively a restart.

You can verify who controls the Pod:

```bash
kubectl describe pod <pod> -n dev
```

Look for:

```text
Controlled By:
```

For example:

```text
Controlled By: ReplicaSet/backend-7d96c7c8b9
```

## 8. `rollout restart` vs `delete pod`

For normal team operations, prefer:

```bash
kubectl rollout restart deployment/backend -n dev
```

It is clearer because it expresses the real intention:

> restart this Deployment.

Advantages:
- explicit;
- works with all replicas;
- follows Deployment rollout behavior;
- easier to understand in production operations.

Deleting one Pod is convenient for a quick experiment:

```bash
kubectl delete pod <pod>
```

## 9. ConfigMap changed, but the Pod did not

This directly applies to our `PROJECT_NAME` problem.

Suppose the Deployment uses:

```yaml
envFrom:
  - configMapRef:
      name: backend-configmap
```

and we change:

```yaml
data:
  PROJECT_NAME: Full Stack FastAPI Project
```

Then run:

```bash
kubectl apply -k k8s/overlays/dev
```

The ConfigMap changes.

But an already-running process does not magically receive new environment variables.

Environment variables are created when the container starts.

Therefore:

```text
ConfigMap changed
      ↓
existing Pod still has old environment
```

Restart the backend:

```bash
kubectl rollout restart deployment/backend -n dev
```

## 10. Typical ConfigMap debugging cycle

Edit the manifest:

```bash
vim k8s/.../backend-configmap.yaml
```

Apply:

```bash
kubectl apply -k k8s/overlays/dev
```

Restart only the affected workload:

```bash
kubectl rollout restart deployment/backend -n dev
```

Wait:

```bash
kubectl rollout status deployment/backend -n dev
```

Inspect logs:

```bash
kubectl logs -f deployment/backend -n dev
```

This is the normal workflow.

Not:

```bash
kubectl delete namespace dev
```

## 11. Inspect the actual ConfigMap in Kubernetes

List ConfigMaps:

```bash
kubectl get configmap -n dev
```

Show the actual object:

```bash
kubectl get configmap backend-configmap \
  -n dev \
  -o yaml
```

Or:

```bash
kubectl describe configmap backend-configmap -n dev
```

This lets you answer:

> Did Kubernetes actually receive my change?

For the session problem, you could directly check whether:

```yaml
PROJECT_NAME:
```

exists.

## 12. Inspect the environment inside the actual Pod

If the container is running:

```bash
kubectl exec -it deployment/backend -n dev -- env
```

or:

```bash
kubectl exec -it deployment/backend -n dev -- printenv
```

For one variable:

```bash
kubectl exec deployment/backend \
  -n dev \
  -- printenv PROJECT_NAME
```

Expected:

```text
Full Stack FastAPI Project
```

This is extremely useful when:

```text
ConfigMap looks correct
        ↓
application says variable is missing
```

You can verify what the application container actually received.

## 13. Enter a running container

Use:

```bash
kubectl exec -it <pod> -n dev -- sh
```

or:

```bash
kubectl exec -it deployment/backend -n dev -- sh
```

Depending on the image, Bash may also exist:

```bash
kubectl exec -it deployment/backend -n dev -- bash
```

Inside the container, inspect:

```bash
env
ls
pwd
cat ...
hostname
```

You can also test the application locally from inside its container:

```bash
curl localhost:8000/api/v1/utils/health-check/
```

This separates:

```text
application itself
```

from:

```text
Service / networking
```

## 14. What if the container crashes too quickly for `exec`?

Sometimes:

```text
container starts
↓
immediately crashes
```

and you cannot enter it.

Kubernetes also provides:

```bash
kubectl debug
```

It can create debugging containers or copies of Pods for troubleshooting.

This becomes especially useful with minimal production images that do not contain:

```text
bash
curl
ping
ps
netstat
```

This is a more advanced topic, but it is worth knowing that the mechanism exists.

## 15. Create a temporary debugging Pod

A very useful technique for cluster networking:

```bash
kubectl run debug \
  --rm -it \
  --restart=Never \
  --image=curlimages/curl \
  -n dev \
  -- sh
```

From inside:

```bash
curl http://backend
```

or:

```bash
curl http://backend:80/api/v1/utils/health-check/
```

This asks:

> Can another Pod inside the cluster reach the backend Service?

That helps divide the problem into layers:

```text
Application problem?
Service problem?
DNS problem?
Network problem?
Ingress problem?
```

## 16. Understand Service port vs container port

During the session, there was confusion between:

```text
container port = 8000
Service port   = 80
local port     = 8080
```

The correct port-forward was:

```bash
kubectl port-forward svc/backend 8080:80 -n dev
```

The path is:

```text
localhost:8080
      │
      ▼
Service backend:80
      │
      ▼
targetPort:8000
      │
      ▼
FastAPI container:8000
```

These are three different ports and do not have to be equal.

## 17. Debug a Service

Start with:

```bash
kubectl get svc -n dev
```

Then:

```bash
kubectl describe svc backend -n dev
```

Important fields:

```text
Selector
Port
TargetPort
Endpoints
```

The Service exists independently from whether a healthy Pod actually matches it.

## 18. A Service can exist but have no backend Pods

Check:

```bash
kubectl get endpoints backend -n dev
```

Or:

```bash
kubectl get endpointslices -n dev
```

If you see:

```text
ENDPOINTS   <none>
```

then the Service currently has no Pod to send traffic to.

One common cause:

```text
Service selector
      !=
Pod labels
```

## 19. Compare Service selectors with Pod labels

Show Pod labels:

```bash
kubectl get pods -n dev --show-labels
```

Inspect Service:

```bash
kubectl get svc backend -n dev -o yaml
```

Example Service:

```yaml
selector:
  app: backend
```

Pod must have something matching:

```yaml
metadata:
  labels:
    app: backend
```

Otherwise:

```text
Service
   ↓
no matching Pod
   ↓
no endpoints
```

## 20. Namespaces were a real problem in our session

The `prestart` Job appeared in:

```text
default
```

while registry credentials existed in:

```text
dev
```

Important rule:

> Most application resources are namespace-scoped.

For example:

```text
dev/backend-secret
```

and:

```text
default/backend-secret
```

are different resources.

Likewise:

```text
dev/backend
default/backend
```

can both exist at the same time.

## 21. Find resources if you do not know their namespace

Use:

```bash
kubectl get pods -A
```

For Jobs:

```bash
kubectl get jobs -A
```

For Secrets:

```bash
kubectl get secrets -A
```

For the session problem:

```bash
kubectl get jobs -A
```

would quickly reveal something like:

```text
NAMESPACE   NAME
default     prestart
```

This is much faster than searching manifests blindly.

## 22. Set `dev` as the default namespace for the current context

You can configure:

```bash
kubectl config set-context --current --namespace=dev
```

Then:

```bash
kubectl get pods
```

implicitly means:

```bash
kubectl get pods -n dev
```

Check the active context:

```bash
kubectl config view --minify
```

For learning, explicitly writing:

```bash
-n dev
```

is still useful because it makes namespace awareness visible.

## 23. Let Kustomize define the namespace

A safer design is to put the namespace into the overlay.

For `dev`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: dev
```

For production:

```yaml
namespace: prod
```

Then:

```bash
kubectl apply -k k8s/overlays/dev
```

already knows that the generated resources belong to `dev`.

This reduces accidental deployment to `default`.

## 24. Render Kustomize before applying it

Before:

```bash
kubectl apply -k ...
```

run:

```bash
kubectl kustomize k8s/overlays/dev
```

This shows the final generated manifests after combining:

```text
base
+
overlay
+
patches
+
namespace
+
transformations
```

For easier inspection:

```bash
kubectl kustomize k8s/overlays/dev | less
```

This is particularly useful when you are learning Kustomize because it removes the abstraction:

> What will actually be sent to the Kubernetes API?

## 25. Use `kubectl diff` before `apply`

Before changing the cluster:

```bash
kubectl diff -k k8s/overlays/dev
```

Then:

```bash
kubectl apply -k k8s/overlays/dev
```

Good operational workflow:

```text
Git manifest
     ↓
kubectl diff
     ↓
review changes
     ↓
kubectl apply
```

This becomes increasingly important for production.

## 26. A better standard deployment loop

For the project, a useful routine is:

```bash
kubectl diff -k k8s/overlays/dev
```

Then:

```bash
kubectl apply -k k8s/overlays/dev
```

Then:

```bash
kubectl get pods -n dev
```

Then:

```bash
kubectl rollout status deployment/backend -n dev
```

Finally:

```bash
kubectl get events -n dev --sort-by=.lastTimestamp
```

This gives immediate feedback about whether the deployment produced healthy resources.

## 27. Deployment changes normally trigger automatic rollout

If you modify the Pod template inside a Deployment, Kubernetes creates a new ReplicaSet automatically.

Examples include changing:

```yaml
image:
```

```yaml
env:
```

```yaml
startupProbe:
```

```yaml
livenessProbe:
```

```yaml
readinessProbe:
```

```yaml
resources:
```

After:

```bash
kubectl apply -k ...
```

Kubernetes performs the rollout.

You normally do not need:

```text
scale to 0
scale back to 1
```

## 28. Inspect Deployment rollout

Check progress:

```bash
kubectl rollout status deployment/backend -n dev
```

Inspect history:

```bash
kubectl rollout history deployment/backend -n dev
```

This is useful when a new version fails.

## 29. Roll back a bad Deployment

Instead of destroying the environment:

```bash
kubectl rollout undo deployment/backend -n dev
```

Conceptually:

```text
revision 5
   ↓
broken
   ↓
rollback
   ↓
revision 4
```

This is one of Kubernetes' important operational features.

## 30. Scale only one component

Stop the backend:

```bash
kubectl scale deployment/backend \
  --replicas=0 \
  -n dev
```

Start it again:

```bash
kubectl scale deployment/backend \
  --replicas=1 \
  -n dev
```

Useful for controlled experiments.

For example:

```text
stop backend
keep postgres running
inspect database
start backend again
```

## 31. Scaling is not the normal restart mechanism

This works:

```text
replicas 1
↓
replicas 0
↓
replicas 1
```

but for restart use:

```bash
kubectl rollout restart deployment/backend -n dev
```

It expresses the intention more accurately.

## 32. Jobs behave differently from Deployments

The project has a `prestart` Job.

Deployment:

```text
should continue running
```

Job:

```text
starts
↓
performs task
↓
Completes
```

Typical uses:

```text
database migration
initialization
one-time setup
data preparation
```

Once a Job has completed, Kubernetes does not treat it like a permanently running application.

## 33. Run the prestart Job again

A simple development workflow:

```bash
kubectl delete job prestart -n dev
```

Then:

```bash
kubectl apply -k k8s/overlays/dev
```

Now Kubernetes creates the Job again.

Check:

```bash
kubectl get jobs -n dev
```

Logs:

```bash
kubectl logs job/prestart -n dev
```

This is much more precise than recreating the namespace.

## 34. Diagnose a Job

Inspect:

```bash
kubectl describe job prestart -n dev
```

Logs:

```bash
kubectl logs job/prestart -n dev
```

Also inspect its Pod:

```bash
kubectl get pods -n dev
```

A Job manages one or more Pods, so failures may need Pod-level troubleshooting as well.

## 35. PVC debugging was another major issue

During the session:

```text
postgres-vol → Pending
```

because the manifest requested:

```yaml
storageClassName: local-path
```

but Minikube had:

```text
standard
```

available.

Start with:

```bash
kubectl get pvc -n dev
```

Then:

```bash
kubectl describe pvc postgres-vol -n dev
```

And:

```bash
kubectl get storageclass
```

or:

```bash
kubectl get sc
```

## 36. Storage troubleshooting sequence

Use:

```bash
kubectl get pvc -n dev
```

Then:

```bash
kubectl describe pvc postgres-vol -n dev
```

Then:

```bash
kubectl get pv
```

Then:

```bash
kubectl get sc
```

This lets you understand:

```text
PVC request
     ↓
StorageClass
     ↓
Provisioner
     ↓
PV
     ↓
Pod mount
```

## 37. Do not delete PVCs for a normal restart

These commands are fundamentally different:

```bash
kubectl delete pod postgres-xxxx -n dev
```

and:

```bash
kubectl delete pvc postgres-vol -n dev
```

A Pod represents a running workload.

A PVC represents storage.

Conceptually:

```text
Pod
≈ process/runtime
```

```text
PVC
≈ persistent data
```

In a production database environment:

```bash
kubectl delete pvc ...
```

can become a data-loss operation depending on the underlying storage and reclaim policy.

Never treat PVC deletion as a restart mechanism.

## 38. Why the PVC needed recreation in our case

A PVC created with the wrong storage configuration may need to be recreated after fixing the manifest.

For example:

```text
PVC
storageClassName = local-path
        ↓
StorageClass does not exist
        ↓
PVC stays Pending
```

After changing it:

```text
storageClassName = standard
```

the old PVC may still require recreation.

This is relatively safe when:
- the PVC is still `Pending`;
- no actual volume was provisioned;
- no application data exists.

It becomes dangerous after real data is stored.

## 39. Make storage portable between Minikube and K3s

Our session exposed an environment-specific difference:

```text
Minikube → standard
K3s      → local-path
```

Hardcoding either one into the common `base` makes the manifests less portable.

A better model:

```text
base
├── postgres Deployment
└── PVC

overlays
├── minikube
│   └── storage patch → standard
│
└── k3s
    └── storage patch → local-path
```

Or, where appropriate, let Kubernetes use the default StorageClass instead of explicitly naming one.

## 40. `Pending` usually means Kubernetes has not started the application yet

If a Pod is:

```text
Pending
```

do not immediately search application logs.

Possible causes include:

```text
PVC unavailable
insufficient CPU
insufficient memory
node selector mismatch
taints
affinity rules
scheduling problems
volume problems
```

Start with:

```bash
kubectl describe pod <pod> -n dev
```

and:

```bash
kubectl get events -n dev --sort-by=.lastTimestamp
```

The container may not even exist yet.

## 41. Standard `CrashLoopBackOff` workflow

If:

```text
STATUS = CrashLoopBackOff
```

use:

```bash
kubectl logs <pod> -n dev
```

Then:

```bash
kubectl logs <pod> -n dev --previous
```

Then:

```bash
kubectl describe pod <pod> -n dev
```

Investigate:

```text
application exception
missing environment variable
wrong command
wrong arguments
bad ConfigMap
bad Secret
failed dependency
probe failure
OOMKilled
permissions
filesystem problem
```

Our backend incident involved both application configuration and probe behavior.

## 42. Standard `ImagePullBackOff` workflow

If:

```text
ErrImagePull
```

or:

```text
ImagePullBackOff
```

run:

```bash
kubectl describe pod <pod> -n dev
```

Inspect Events.

Typical causes:

```text
wrong repository
wrong image name
wrong tag
private registry
missing imagePullSecret
Secret in wrong namespace
invalid credentials
architecture incompatibility
```

Do not restart everything.

First understand why the image cannot be pulled.

## 43. Registry Secrets are namespace-scoped

This mattered in the session because registry credentials existed in `dev` while another workload appeared in `default`.

Check:

```bash
kubectl get secrets -n dev
```

Then:

```bash
kubectl describe secret <registry-secret> -n dev
```

A Pod in:

```text
default
```

cannot simply reference:

```text
dev/my-registry-secret
```

Namespace boundaries matter.

## 44. Prefer immutable image tags

The session included discussion about overwriting the same image tag and image caching.

This is difficult to reason about:

```text
backend:dev
backend:dev
backend:dev
backend:dev
```

All tags look identical even though the image content changed.

Better:

```text
backend:a81c2f3
backend:b77f102
backend:6ff34da
```

For example, use a Git commit SHA.

Then:

```text
Git commit a81c2f3
        ↓
Docker image backend:a81c2f3
        ↓
Kubernetes Deployment
```

The deployed version becomes unambiguous.

## 45. Change the image for one Deployment

For an immediate manual update:

```bash
kubectl set image deployment/backend \
  backend=ghcr.io/.../backend:a81c2f3 \
  -n dev
```

This triggers a rollout.

However, in a GitOps-oriented repository, the permanent image version should also be updated in Git.

Otherwise:

```text
Git configuration
        !=
live cluster configuration
```

This is called configuration drift.

## 46. Understand the three probe types

The session provided a real example of why this matters.

The backend was being killed before it could fully start, and adding a `startupProbe` helped solve the problem.

There are three separate concepts.

### `startupProbe`

Question:

> Has the application finished starting?

Example:

```yaml
startupProbe:
  httpGet:
    path: /api/v1/utils/health-check/
    port: 8000
  periodSeconds: 5
  failureThreshold: 30
```

This can give the application:

```text
5 seconds × 30 failures = approximately 150 seconds
```

to start.

### `readinessProbe`

Question:

> Is this Pod ready to receive traffic?

If readiness fails:

```text
Pod process may stay alive
      ↓
Service should stop sending traffic to it
```

This is useful when an application is alive but temporarily unable to serve requests.

### `livenessProbe`

Question:

> Is this application still alive, or should Kubernetes restart it?

If liveness repeatedly fails:

```text
Kubernetes restarts the container
```

That makes liveness a powerful but potentially dangerous mechanism.

## 47. Do not make liveness depend too strongly on external systems

For example, imagine a health endpoint checks PostgreSQL.

Then:

```text
PostgreSQL unavailable
        ↓
backend liveness returns failure
        ↓
Kubernetes kills backend
        ↓
backend restarts
        ↓
PostgreSQL still unavailable
        ↓
backend killed again
```

Now you have two problems instead of one.

A liveness probe should usually answer something closer to:

> Is this application process healthy enough that restarting it might help?

Readiness can be stricter.

## 48. Diagnose probe failures

Run:

```bash
kubectl describe pod <backend-pod> -n dev
```

The Pod description contains probe configuration.

Events may show:

```text
Startup probe failed
Readiness probe failed
Liveness probe failed
```

These messages are important because a container can be restarted by Kubernetes even if the application itself did not crash.

That was relevant to the backend behavior seen during the session.

## 49. `Running` does not mean `Ready`

A Pod can show:

```text
READY   STATUS
0/1     Running
```

This means:

```text
container process is running
```

but:

```text
Pod is not ready for traffic
```

Typical reason:

```text
readinessProbe failing
```

Inspect:

```bash
kubectl describe pod <pod> -n dev
kubectl logs <pod> -n dev
```

Do not assume that `Running` means the application is healthy.

## 50. Get a useful environment snapshot

For this project:

```bash
kubectl get \
  pods,deployments,jobs,services,pvc,configmaps \
  -n dev
```

This gives a compact view of most important components.

You can also create:

```bash
alias kdev='kubectl -n dev'
```

Then:

```bash
kdev get pods
kdev get svc
kdev get pvc
kdev get jobs
```

This reduces typing while preserving namespace safety.

## 51. `kubectl get all` does not actually mean everything

This command is useful:

```bash
kubectl get all -n dev
```

But it does not show every possible Kubernetes object.

You may still need:

```bash
kubectl get pvc -n dev
kubectl get cm -n dev
kubectl get secret -n dev
```

So interpret `get all` as:

> show many common workload resources

not:

> show every resource in Kubernetes.

## 52. Use `kubectl explain` while learning manifests

Instead of searching documentation every time:

```bash
kubectl explain deployment
```

Then:

```bash
kubectl explain deployment.spec
```

Then:

```bash
kubectl explain deployment.spec.strategy
```

For probes:

```bash
kubectl explain \
  deployment.spec.template.spec.containers.livenessProbe
```

For PVC:

```bash
kubectl explain pvc.spec.storageClassName
```

This is Kubernetes API documentation directly in the terminal.

It is especially useful while learning how manifests are structured.

## 53. Use `kubectl edit` for experiments, not permanent configuration

You can edit the live object:

```bash
kubectl edit deployment backend -n dev
```

or:

```bash
kubectl edit configmap backend-configmap -n dev
```

This can be useful for testing a hypothesis quickly.

But it creates a risk:

```text
Git manifest = version A

cluster object = version B
```

Therefore, a good troubleshooting workflow is:

```text
kubectl edit
      ↓
test hypothesis
      ↓
confirmed
      ↓
make same change in Git
      ↓
kubectl apply
```

Git should remain the source of truth.

## 54. `kubectl patch`

Another useful troubleshooting tool is:

```bash
kubectl patch ...
```

It lets you modify a small part of a live Kubernetes object without manually editing the full YAML.

This is useful for quick experiments.

For long-term project configuration:

```text
Kustomize + Git
```

is preferable.

## 55. Debugging decision tree

Keep this close during the next session:

```text
Something is broken
        │
        ▼
kubectl get pods -n dev
        │
        ├── Pending
        │     │
        │     ├─ kubectl describe pod
        │     ├─ kubectl get events
        │     ├─ kubectl get pvc
        │     └─ kubectl get sc
        │
        ├── ImagePullBackOff
        │     │
        │     └─ kubectl describe pod
        │          ├─ correct image?
        │          ├─ correct tag?
        │          └─ imagePullSecret?
        │
        ├── CrashLoopBackOff
        │     │
        │     ├─ kubectl logs
        │     ├─ kubectl logs --previous
        │     └─ kubectl describe pod
        │          ├─ application error?
        │          ├─ environment?
        │          ├─ ConfigMap?
        │          ├─ Secret?
        │          └─ probes?
        │
        ├── Running 0/1
        │     │
        │     ├─ readinessProbe
        │     ├─ kubectl logs
        │     └─ kubectl describe pod
        │
        └── Running 1/1
              │
              └─ application unreachable
                    │
                    ├─ kubectl get svc
                    ├─ kubectl get endpoints
                    ├─ kubectl port-forward
                    └─ curl from debug Pod
```

## 56. What to do after fixing a specific problem

### Backend ConfigMap changed

```bash
kubectl apply -k k8s/overlays/dev
kubectl rollout restart deployment/backend -n dev
kubectl rollout status deployment/backend -n dev
kubectl logs -f deployment/backend -n dev
```

### Backend image changed

```bash
kubectl apply -k k8s/overlays/dev
kubectl rollout status deployment/backend -n dev
kubectl logs -f deployment/backend -n dev
```

If the image field in the Pod template changed, an additional manual restart is normally unnecessary.

### Backend is simply stuck

```bash
kubectl rollout restart deployment/backend -n dev
```

### Prestart Job must run again

```bash
kubectl delete job prestart -n dev
kubectl apply -k k8s/overlays/dev
kubectl logs -f job/prestart -n dev
```

### PostgreSQL Pod needs restart

```bash
kubectl rollout restart deployment/postgres-deployment -n dev
```

Do not delete the PVC.

### PVC is `Pending`

```bash
kubectl describe pvc postgres-vol -n dev
kubectl get sc
```

Only if it is safe and contains no required data:

```bash
kubectl delete pvc postgres-vol -n dev
kubectl apply -k k8s/overlays/dev
```

## 57. Commands to learn first

There is no need to memorize the entire `kubectl` command set.

For your current stage, concentrate on:

```text
kubectl get
kubectl describe
kubectl logs
kubectl logs --previous
kubectl exec
kubectl port-forward
kubectl get events

kubectl apply
kubectl diff
kubectl delete

kubectl rollout restart
kubectl rollout status
kubectl rollout history
kubectl rollout undo

kubectl scale

kubectl get pvc
kubectl get sc

kubectl kustomize
kubectl explain
```

These commands cover a large part of everyday Kubernetes troubleshooting.

## 58. Main knowledge gaps visible during the session

The session suggests that the most important areas to study next are:

|Topic|How the gap appeared|
|---|---|
|Pod lifecycle|A failed Pod was treated too much like a failure of the whole environment|
|Controllers|Deployment's ability to recreate Pods was underused|
|Partial restart|Full or large redeploys were used where `rollout restart` would be enough|
|Logs and Events|Troubleshooting was less systematic than `logs → previous → describe → events`|
|ConfigMap lifecycle|It was not fully clear when changing a ConfigMap requires Pod restart|
|Job lifecycle|`prestart` behaved differently from a Deployment|
|Namespaces|The Job and registry Secret ended up in different namespaces|
|Storage|Minikube `standard` and K3s `local-path` were mixed|
|Service networking|Service port and container port caused confusion|
|Probes|The roles of startup/readiness/liveness were not yet clearly separated|

These correspond directly to issues encountered in the live session.

## 59. Recommended team debugging routine

When something fails, do not immediately change anything.

First collect evidence.

```bash
kubectl get pods -n dev -o wide
```

Then:

```bash
kubectl get events -n dev \
  --sort-by=.lastTimestamp
```

Then:

```bash
kubectl describe pod <problem-pod> -n dev
```

Then:

```bash
kubectl logs <problem-pod> -n dev
```

If the container restarted:

```bash
kubectl logs <problem-pod> \
  -n dev \
  --previous
```

Now classify the failure:

```text
Problem is in:

[ ] Application
[ ] Configuration
[ ] Probe
[ ] Image
[ ] Registry
[ ] Storage
[ ] Service
[ ] Namespace
[ ] Scheduler
[ ] Node
```

Only then change something.

A good engineering loop is:

```text
Observe
   ↓
Form hypothesis
   ↓
Change ONE thing
   ↓
Observe
   ↓
Confirm or reject hypothesis
```

A bad debugging loop is:

```text
Change five things
       ↓
Redeploy everything
       ↓
It works
       ↓
We do not know which change fixed it
```

The second approach removes an important part of learning: understanding cause and effect.

## 60. Ultra-short live debugging cheat sheet

Keep this block open during the next meeting:

```bash
# --------------------------------------------------
# OVERVIEW
# --------------------------------------------------

kubectl get pods -n dev -o wide

kubectl get events -n dev \
  --sort-by=.lastTimestamp

# --------------------------------------------------
# POD DIAGNOSTICS
# --------------------------------------------------

kubectl describe pod <pod> -n dev

kubectl logs <pod> -n dev

kubectl logs <pod> -n dev --previous

kubectl logs -f deployment/backend -n dev

# --------------------------------------------------
# RESTART ONLY BACKEND
# --------------------------------------------------

kubectl rollout restart deployment/backend -n dev

kubectl rollout status deployment/backend -n dev

# --------------------------------------------------
# CONTAINER
# --------------------------------------------------

kubectl exec -it deployment/backend \
  -n dev \
  -- sh

kubectl exec deployment/backend \
  -n dev \
  -- printenv PROJECT_NAME

# --------------------------------------------------
# CONFIG
# --------------------------------------------------

kubectl get cm -n dev

kubectl get cm backend-configmap \
  -n dev \
  -o yaml

# --------------------------------------------------
# SERVICE / NETWORK
# --------------------------------------------------

kubectl get svc -n dev

kubectl describe svc backend -n dev

kubectl get endpoints backend -n dev

kubectl port-forward \
  svc/backend \
  8080:80 \
  -n dev

# --------------------------------------------------
# JOBS
# --------------------------------------------------

kubectl get jobs -n dev

kubectl describe job prestart -n dev

kubectl logs job/prestart -n dev

kubectl delete job prestart -n dev

# --------------------------------------------------
# STORAGE
# --------------------------------------------------

kubectl get pvc -n dev

kubectl describe pvc postgres-vol -n dev

kubectl get pv

kubectl get sc

# --------------------------------------------------
# KUSTOMIZE
# --------------------------------------------------

kubectl kustomize k8s/overlays/dev

kubectl diff -k k8s/overlays/dev

kubectl apply -k k8s/overlays/dev

# --------------------------------------------------
# ROLLOUT
# --------------------------------------------------

kubectl rollout history deployment/backend -n dev

kubectl rollout undo deployment/backend -n dev

# --------------------------------------------------
# FIND WRONG NAMESPACE
# --------------------------------------------------

kubectl get pods -A

kubectl get jobs -A
```

## Main Operational Rule

For normal troubleshooting, treat:

```bash
kubectl delete namespace dev
```

as an exceptional command, not a standard debugging command.

Instead:

```text
Find the broken resource
        ↓
Prove the cause with kubectl
        ↓
Fix only that resource/configuration
        ↓
Restart only the dependent workload
        ↓
Observe the result
```

This is the important transition from:

```text
"I know how to deploy an application to Kubernetes"
```

to:

```text
"I know how to operate and troubleshoot an application that is already running in Kubernetes."
```

That second skill is exactly what was missing most during the live session.