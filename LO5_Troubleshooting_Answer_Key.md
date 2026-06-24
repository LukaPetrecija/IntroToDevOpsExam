# LO5 — Solve problems with application shipping by using containers
### Detailed answer key (mistake → fix → why)

*Format: each item shows the broken command/manifest, the corrected version, and a full explanation of the root cause. Two debugging reflexes to carry through the whole exam: for **containers**, the container lives only as long as its **PID 1** process; for **Kubernetes**, almost every failure is read from `kubectl describe <obj>` **Events** or `kubectl logs [--previous]`.*

---

## Part A — `podman run` mistakes

### 1. Reversed port mapping
**Broken:** `podman run -d --name web -p 80:8080 docker.io/library/nginx`
**Fix:** `podman run -d --name web -p 80:80 docker.io/library/nginx` (or `-p 8080:80` to use host port 8080)
**Why:** `-p` is **`HOST:CONTAINER`**. nginx listens on container port **80**, but the command forwards host 80 to container **8080**, where nothing is listening — so nothing answers. The **container-side** number must match the port the app actually binds (80). The "reversed" part is putting the wrong port on the container side.

### 2. Empty required environment variable
**Broken:** `podman run -d --name db -e MYSQL_ROOT_PASSWORD docker.io/library/mysql`
**Fix:** `podman run -d --name db -e MYSQL_ROOT_PASSWORD=secret docker.io/library/mysql`
**Why:** `-e MYSQL_ROOT_PASSWORD` **without `=value`** passes the variable with the value taken from your shell (usually unset/empty). The MySQL image **refuses to initialize** unless exactly one of `MYSQL_ROOT_PASSWORD`, `MYSQL_ALLOW_EMPTY_PASSWORD=1`, or `MYSQL_RANDOM_ROOT_PASSWORD=1` is set, so init aborts and the container exits. `podman logs db` shows the "Database is uninitialized and password option is not specified" message. Always use `-e KEY=value` for required secrets.

### 3. `-p` ignored with host networking
**Broken:** `podman run -d --name app --network host -p 8080:80 docker.io/library/nginx`
**Fix:** drop one of them — either `podman run -d --name app --network host nginx` (app uses host ports directly) **or** `podman run -d --name app -p 8080:80 nginx` (bridge network with mapping).
**Why:** with **`--network host`** the container **shares the host's network namespace** — there is no separate container network to NAT into, so **port publishing (`-p`) has nothing to map and is ignored**. nginx binds directly to the host's port 80. `-p` only makes sense on an isolated (bridge) network where a host port is forwarded into the container.

### 4. Container exits immediately (Exited 0)
**Broken:** `podman run -d --name c1 docker.io/library/busybox`
**Fix:** give it a long-running process, e.g. `podman run -d --name c1 docker.io/library/busybox sleep infinity` (or `tail -f /dev/null`).
**Why:** a container runs **only as long as its main process (PID 1)** runs. busybox's default command is `sh`; detached (`-d`) with no input, `sh` immediately hits EOF and exits cleanly → **Exited (0)**. There's no "background idle" — you must give PID 1 something that keeps running.

### 5. `--rm` + detached short job: logs vanish
**Broken:** `podman run --rm -d --name job docker.io/library/alpine echo hello` then `podman logs job` → *no such container*.
**Fix:** drop `--rm` (`podman run -d --name job alpine echo hello; podman logs job`), or run it **foreground** (`podman run --rm alpine echo hello`) so output prints to your terminal.
**Why:** `--rm` **auto-deletes the container the instant it exits**. `echo hello` finishes in milliseconds, so by the time you run `podman logs job` the container — and its logs — are already gone. `--rm` is for foreground throwaways where you watch the output live; it's incompatible with "run detached, inspect logs later."

### 6. Memory limit too small for MySQL
**Broken:** `podman run -d -p 8080:80 --memory 8m docker.io/library/mysql`
**Fix:** raise the limit, e.g. `--memory 512m` (or more) — and note port `80` is also wrong for MySQL (it listens on 3306), but the asked-about problem is memory.
**Why:** `--memory 8m` caps the container at **8 MB**, far below what MySQL needs even to start. The kernel **OOM-kills** the process during initialization, so it never becomes healthy and keeps dying. `podman logs`/`podman inspect` would show it being killed. The memory **limit** is the constraint.

### 7. Container-name DNS fails on the default network
**Broken:**
```
podman run -d --name a docker.io/library/alpine sleep 1d
podman run -d --name b docker.io/library/alpine sleep 1d
podman exec a ping b      # "bad address 'b'"
```
**Fix:** put both on a **user-defined network**:
```
podman network create mynet
podman run -d --name a --network mynet alpine sleep 1d
podman run -d --name b --network mynet alpine sleep 1d
podman exec a ping b      # resolves now
```
**Why:** the **default bridge network does not provide name-based DNS** between containers — only IP works there. **User-defined networks** enable an embedded DNS resolver (aardvark-dns) so containers can resolve **each other by name**. This is the same reason the TempConverter project created `tempconverter-net`: it's what lets the app reach the database by the name `tempconverter-db`.

### 8. Bind mount missing SELinux relabel (and absolute path)
**Broken:** `podman run -d --name web -v ./html:/usr/share/nginx/html docker.io/library/nginx`
**Fix:** `podman run -d --name web -v "$(pwd)/html:/usr/share/nginx/html:Z" docker.io/library/nginx`
**Why:** two issues. On **SELinux-enforcing** hosts (Fedora/RHEL), a bind mount without a relabel flag is **denied** access by SELinux; appending **`:Z`** (private relabel) or **`:z`** (shared) tells podman to relabel the host directory so the container can read it. Also, a **relative path** like `./html` is fragile/ambiguous — use an **absolute path**. With those fixed, the files appear inside the container.

---

## Part B — Dockerfile / Containerfile mistakes

*(Item 9 is the section header: "for each Dockerfile, identify the mistake, fix it, explain.")*

### 10. `apt-get install` without `update`
**Broken:**
```dockerfile
FROM debian:12
RUN apt-get install -y nginx
```
**Fix:**
```dockerfile
FROM debian:12
RUN apt-get update && apt-get install -y nginx && rm -rf /var/lib/apt/lists/*
```
**Why:** the base image ships with **empty package index lists** (`/var/lib/apt/lists` is cleared to save space), so `apt-get install` can't find `nginx` ("Unable to locate package"). You must run **`apt-get update`** first to fetch the indexes. Combine update + install in **one `RUN`** so the refreshed index isn't cached separately from the install (avoids stale-cache bugs).

### 11. Shell-form CMD breaks signal handling
**Broken:**
```dockerfile
CMD python app.py
```
**Fix:**
```dockerfile
CMD ["python", "app.py"]
```
**Why:** **shell form** (`CMD python app.py`) runs as **`/bin/sh -c "python app.py"`**, making **`sh` PID 1** and python its child. `sh` doesn't forward signals, so `SIGTERM` from `podman stop`/Kubernetes never reaches python — the app can't shut down gracefully and is eventually **killed** after the grace period. **Exec form** (the JSON array) runs python **directly as PID 1**, so it receives signals and can stop cleanly.

### 12. Bad layer ordering kills npm caching
**Broken:**
```dockerfile
FROM node:20
COPY . /app
WORKDIR /app
RUN npm install
```
**Fix:**
```dockerfile
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
```
**Why:** `COPY . /app` copies **all source first**, so **any** source change invalidates that layer and **every** layer after it — forcing `npm install` to rerun on every build. Dependencies change far less often than source, so **copy only `package.json`/`package-lock.json` first, install, then copy the rest**. Now `npm install`'s layer is cached and reused unless the dependency files actually change.

### 13. Overwriting PATH breaks the shell
**Broken:**
```dockerfile
FROM alpine:3.20
ENV PATH=/app/bin
RUN apk add --no-cache curl
```
**Fix:**
```dockerfile
ENV PATH="/app/bin:${PATH}"
```
**Why:** `ENV PATH=/app/bin` **replaces the entire PATH**, throwing away `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`. After that, the shell can't find `apk`, `sh`, `curl`, etc. — "not found." Always **append** to PATH (`/app/bin:${PATH}`) so the system directories remain.

### 14 & 15. EXPOSE/published port vs the port the app actually listens on
**Broken:**
```dockerfile
FROM ubuntu:24.04
EXPOSE 8080
CMD ["python3", "-m", "http.server", "3000"]
```
(then `-p 8080:8080` answers nothing)
**Fix (make the ports agree):**
```dockerfile
FROM ubuntu:24.04
EXPOSE 8080
CMD ["python3", "-m", "http.server", "8080"]
```
run with `-p 8080:8080`. (Also note `ubuntu:24.04` may not include `python3` — you might need `RUN apt-get update && apt-get install -y python3`.)
**Why:** the server **listens on 3000**, but you publish **8080→8080**, so traffic arrives at a port nothing is bound to. **`EXPOSE` does *not* publish or open anything** — it's **documentation/metadata** declaring which port the image *intends* to use; only `-p`/`ports:` actually map a host port. The real fix is to make three things consistent: the **app's listen port**, the **EXPOSE**, and the **`-p` container side**.

### 16. Huge image — clean apt cache in the same layer
**Broken:**
```dockerfile
FROM debian:12
RUN apt-get update && apt-get install -y build-essential
```
**Fix:**
```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends build-essential && \
    rm -rf /var/lib/apt/lists/*
```
**Why:** the downloaded package indexes and `.deb` caches bloat the image. They must be removed **in the same `RUN` layer** that created them — deleting them in a *later* layer doesn't shrink the image, because the files still exist in the earlier layer (layers are additive). `--no-install-recommends` further avoids pulling optional extras.

### 17. USER before COPY/install → permission denied
**Broken:**
```dockerfile
FROM python:3.11
USER appuser
COPY requirements.txt /app/requirements.txt
RUN pip install -r /app/requirements.txt
```
**Fix:**
```dockerfile
FROM python:3.11
RUN useradd --create-home appuser
WORKDIR /app
COPY --chown=appuser:appuser requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
USER appuser
CMD ["python", "app.py"]
```
**Why:** several problems. `appuser` is **switched to before it's even created** (no `useradd`), and **`COPY` defaults to root ownership**, so the non-root user can't write where needed. Worse, `pip install` into the **system site-packages** requires root and fails with **permission denied** as `appuser`. Do the privileged work (create user, install deps) **as root first**, set ownership with `--chown`, and **switch to `USER appuser` last** (or use `pip install --user`).

### 18. Multi-stage build to shrink a Go image
**Broken:**
```dockerfile
FROM golang:1.22
COPY . /src
WORKDIR /src
RUN go build -o app .
CMD ["/src/app"]
```
**Fix:**
```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /app .

FROM gcr.io/distroless/static   # or alpine, or scratch
COPY --from=build /app /app
CMD ["/app"]
```
**Why:** the single-stage image ships the **entire Go toolchain, source, and build cache (~1 GB)**, none of which the running binary needs. A **multi-stage build** compiles in the heavy `golang` stage, then **copies only the finished binary** into a tiny final base (distroless/alpine/scratch), yielding an image of a few MB. `CGO_ENABLED=0` makes a static binary so it runs on `scratch`/`distroless`.

---

## Part C — Kubernetes troubleshooting

### 19. Pod stuck `Pending` — diagnose with describe
```bash
kubectl describe pod <pod>     # read the Events at the bottom
```
**How to read it:** the scheduler explains *why* it couldn't place the pod:
- **`Insufficient cpu`/`memory`** → resource **requests** exceed free capacity → lower requests or add a node.
- **`node(s) didn't match node selector`/`affinity`** → no node has the required label → label a node or fix the selector.
- **`node(s) had taint ... that the pod didn't tolerate`** → add a toleration.
- **`pod has unbound immediate PersistentVolumeClaims`** → the PVC isn't bound → create/fix the PVC/StorageClass.
**Why:** `Pending` means **not yet scheduled onto any node**; the Events tell you exactly which scheduling constraint failed.

### 20. YAML indentation broken by removing spaces
```bash
kubectl apply -f deploy.yaml
# error: "error converting YAML to JSON: yaml: line N: did not find expected key"
#  or: "json: cannot unmarshal ... mapping values are not allowed in this context"
```
**Fix:** restore the correct indentation (consistent **2-space** levels; never tabs), then re-apply.
**Why:** **YAML is whitespace-sensitive** — indentation defines structure (a key's nesting and which mapping it belongs to). Removing a couple of spaces re-parents a field or breaks a mapping, so the parser fails before Kubernetes even sees the object. Validate with `kubectl apply --dry-run=server -f deploy.yaml` (item 40).

### 21. `ImagePullBackOff`
```bash
kubectl describe pod <pod>     # Events show the pull error
```
**Causes & fixes:**
- **Image name/typo or wrong tag** → "manifest ... not found" → correct the `image:` field.
- **Tag doesn't exist** → fix the tag (avoid relying on a missing `:latest`).
- **Private registry, no credentials** → "unauthorized"/"pull access denied" → create a **docker-registry Secret** and add `imagePullSecrets`.
**Why:** `ImagePullBackOff` means the kubelet **can't pull the image** and is backing off between retries. The describe Events name the precise reason.

### 22. `CrashLoopBackOff`
```bash
kubectl logs <pod> --previous   # logs from the crashed (previous) container
kubectl describe pod <pod>      # exit code / reason
```
**Fix:** read the crash output and address the root cause (missing env var/config, bad command, unreachable dependency, unhandled exception), then redeploy.
**Why:** `CrashLoopBackOff` = the container **starts, exits, and is restarted repeatedly**, with the kubelet waiting longer between attempts. Because the current container is freshly restarted, you need **`--previous`** to see the logs from the instance that actually crashed.

### 23. `OOMKilled`
```bash
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}{"\n"}'
# -> OOMKilled
kubectl describe pod <pod>      # Last State: Terminated, Reason: OOMKilled
```
**Fix:** raise the container's **memory limit** (`resources.limits.memory`) if the app legitimately needs more, **or** fix the app's excessive memory use / leak.
**Why:** the container exceeded its **memory limit**, so the kernel's OOM killer terminated it. Confirm via `lastState.terminated.reason: OOMKilled`. Limits are a **hard cap**: over the memory limit → killed (CPU over-limit only throttles).

### 24. Pods never become `Ready` (failing readiness probe)
```bash
kubectl describe pod <pod>     # "Readiness probe failed: ..."
```
**Fix:** correct the probe (right **path/port**, longer `initialDelaySeconds`/`failureThreshold` for slow starts) or fix the app so it actually serves the probe endpoint.
**Why:** a pod is only `Ready` when its **readiness probe passes**; until then it's excluded from Service Endpoints (gets no traffic) and rollouts stall. A wrong port/path or too-aggressive timing keeps it perpetually not-ready even though the container is running.

### 25. Service returns nothing — label mismatch
```bash
kubectl get endpoints <svc>    # empty / <none>
kubectl get svc <svc> -o jsonpath='{.spec.selector}{"\n"}'
kubectl get pods --show-labels
```
**Fix:** make the **Service selector** match the **pods' labels** (edit whichever is wrong), then Endpoints populate.
**Why:** a Service finds its pods purely by **label selector**. If the selector and the pod labels differ, **no pods are selected**, **Endpoints is empty**, and the Service has nothing to route to — so requests get nothing. Empty Endpoints is the definitive symptom.

### 26. Deployment selector ≠ template labels
**Error on apply:** `selector does not match template labels` (the Deployment is rejected).
**Fix:** make `spec.selector.matchLabels` **identical to** `spec.template.metadata.labels`:
```yaml
spec:
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }   # must match exactly
```
**Why:** a Deployment uses its **selector to claim the pods its template creates**; if they don't match, it could never manage its own pods, so the API server **rejects the object at validation time**. (Also, `selector` is immutable after creation.)

### 27. Requests exceed every node's capacity
```bash
kubectl describe pod <pod>     # "0/N nodes are available: Insufficient cpu (or memory)"
```
**Fix:** lower `resources.requests` to fit a node, or add/enlarge a node.
**Why:** the scheduler must find a node with **at least `requests` worth of free, allocatable** CPU/memory. If a single pod **requests more than any node has**, it can never be placed → permanent `Pending` with "Insufficient …". Requests are about *scheduling fit*, not actual usage.

### 28. Pod mounts a non-existent PVC
```bash
kubectl describe pod <pod>     # "persistentvolumeclaim \"X\" not found" / FailedMount, pod Pending
```
**Fix:** create the PVC (and ensure a matching PV/StorageClass) before/with the pod:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: X }
spec:
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: 1Gi } }
```
**Why:** a pod referencing a PVC **can't start until that claim exists and is bound** — the kubelet reports `FailedMount` and the pod stays `Pending`. Create the claim (dynamic provisioning via a StorageClass will create the PV).

### 29. ubuntu pod with no long-running command
**Symptom:** `Completed`, then `CrashLoopBackOff`.
**Fix:** give it a real long-running command:
```yaml
command: ["sh","-c","while true; do sleep 3600; done"]   # or your actual app
```
**Why:** `ubuntu`'s default command (`bash`) **exits immediately** with no TTY/work → the container "Completed." With the default `restartPolicy: Always`, Kubernetes keeps **restarting a thing that instantly exits**, which presents as **CrashLoopBackOff**. A pod needs a process that **stays running**.

### 30. Triage a pod's whole timeline with events
```bash
kubectl get events --sort-by=.lastTimestamp
kubectl get events --field-selector involvedObject.name=<pod> --sort-by=.lastTimestamp
# newer: kubectl events --for pod/<pod>
```
**Why:** Events are the cluster's **chronological audit log** — scheduling decisions, pulls, mounts, probe failures, restarts. Sorting by `lastTimestamp` lets you **replay everything that happened to the pod in order**, which often reveals a root cause that a single `describe` snapshot misses.

### 31. Shell into a pod and debug DNS
```bash
kubectl exec -it <pod> -- sh
# inside:
cat /etc/resolv.conf            # nameserver = CoreDNS (e.g. 10.96.0.10), search domains
nslookup my-service             # does the service resolve?
nslookup my-service.my-ns.svc.cluster.local
```
**Why:** `/etc/resolv.conf` shows the **CoreDNS** nameserver and the **search domains** (`<ns>.svc.cluster.local`, etc.) that let short names resolve. `nslookup` confirms whether a Service name resolves. DNS failure vs resolution success tells you whether the problem is **naming** or something downstream (Endpoints/port).

### 32. Ephemeral debug container for distroless / no-shell images
```bash
kubectl debug -it <pod> --image=busybox --target=<container>
```
**Why:** **distroless/scratch** images have **no shell or tools**, so `kubectl exec ... sh` fails. `kubectl debug` injects an **ephemeral container** into the *running* pod that **shares the target container's namespaces** (process, network), giving you busybox tools to inspect the broken container **without rebuilding or restarting** it.

### 33. Throwaway pod to test Service connectivity
```bash
kubectl run tmp --rm -it --image=busybox -- sh
# inside: wget -qO- my-service ; nc -zv my-service 80
```
**Why:** `--rm -it` gives a **disposable, interactive** pod that's deleted on exit — a clean network vantage point *inside* the cluster to test DNS + reachability of a Service, isolating cluster-internal problems from outside-the-cluster ones.

### 34. Node `NotReady`
```bash
kubectl describe node <node>          # Conditions + Events
minikube logs                          # kubelet/runtime logs on minikube
minikube ssh -- 'systemctl status kubelet'
```
**What to check:** the **kubelet** (is it running/healthy?), node **Conditions** (`MemoryPressure`, **`DiskPressure`**, `PIDPressure`, `NetworkUnavailable`), the **container runtime**, and the **CNI/network plugin** (a broken CNI keeps a node NotReady).
**Why:** `NotReady` means the kubelet isn't reporting healthy. The node Conditions and kubelet logs pinpoint whether it's resource pressure, a stopped kubelet, or a networking/CNI failure.

### 35. Pod stuck `Terminating` — finalizers, grace period, force delete
```bash
kubectl get pod <pod> -o jsonpath='{.metadata.finalizers}{"\n"}'
kubectl delete pod <pod> --grace-period=0 --force
```
**Why:** on deletion a pod first gets a **deletionTimestamp** and a **grace period** (default 30 s) to shut down cleanly; **finalizers** are keys that block final removal until some controller does cleanup (e.g. detaching a volume) and removes its finalizer. If a finalizer's controller is stuck, the pod hangs in `Terminating`. `--grace-period=0 --force` removes it from the API **immediately**. **Risk:** cleanup may not have actually happened (a volume might still be attached, or for a StatefulSet a replacement could start while the old process still runs → split-brain/data corruption). Force-delete only when you understand the cleanup was done or doesn't matter.

### 36. Wrong apiVersion/kind pairing
**Broken:** `apiVersion: apps/v1` + `kind: Pod`
**Error:** `no matches for kind "Pod" in version "apps/v1"`
**Fix:** `apiVersion: v1` for a Pod (Pods are in the **core** group). (`apps/v1` is for Deployment/ReplicaSet/StatefulSet/DaemonSet.)
**Why:** every object belongs to a specific **group/version/kind**. The API server only accepts a `kind` under the **API version that actually defines it**. Mispairing them means the kind doesn't exist in that version, so the object is rejected. Check with `kubectl explain <kind>` or `kubectl api-resources`.

### 37. `CreateContainerConfigError` — missing ConfigMap key
```bash
kubectl describe pod <pod>     # "couldn't find key X in ConfigMap default/Y"
```
**Fix:** add the missing key to the ConfigMap, or correct the `configMapKeyRef.key`/`name` in the pod spec.
**Why:** the container references a specific key via `valueFrom.configMapKeyRef` (or a Secret), but that **key (or the whole ConfigMap) doesn't exist**, so the kubelet **can't assemble the container's config** and fails before starting it. Same pattern applies to `secretKeyRef`.

### 38. Secret in a different namespace
**Symptom:** the pod can't mount/reference the Secret.
**Fix:** create the Secret **in the same namespace as the pod** (Secrets can't be referenced across namespaces):
```bash
kubectl create secret generic mysecret --from-literal=k=v -n <pod-namespace>
```
**Why:** **Secrets (and ConfigMaps) are namespaced** and can **only be referenced by pods in the same namespace**. There's no cross-namespace reference; you must place a copy of the Secret in each namespace that needs it.

### 39. Rollout stuck — `ProgressDeadlineExceeded`
```bash
kubectl rollout status deployment/<dep>
kubectl describe deployment <dep>      # Conditions: Progressing=False, ProgressDeadlineExceeded
kubectl get pods                       # find the unhealthy new pods
```
**Fix:** address why new pods aren't becoming **Available** — bad image (item 21), failing probe (item 24), insufficient resources (item 27) — or `kubectl rollout undo deployment/<dep>`.
**Why:** a Deployment fails the rollout if new pods don't become available within **`progressDeadlineSeconds`** (default 600 s). The condition tells you it timed out; the *real* cause is always one of the pod-level failures, which you find via the new pods' status/logs.

### 40. Catch typos before applying
```bash
kubectl apply -f manifest.yaml --dry-run=server     # validates against the API schema
# (client-side, no API contact:)
kubectl apply -f manifest.yaml --dry-run=client
```
**Examples caught:** `imagePullpolicy` (wrong case — must be **`imagePullPolicy`**), misnested `ports`, unknown fields.
**Why:** **`--dry-run=server`** sends the object to the API server for **full validation (schema + admission)** *without persisting it*, so it reports unknown/misspelled fields and structural errors before they cause a half-broken deploy. `--dry-run=client` only renders locally and catches less. Fix the flagged field (e.g. correct the camelCase) and re-apply.

### 41. App can't reach its database Service — verify in order
```bash
kubectl get pods                       # 1) is the DB pod Running & Ready?
kubectl get svc db                     # 2) does the Service exist (right name/port)?
kubectl get endpoints db               # 3) are Endpoints populated? (empty => selector/readiness)
kubectl exec -it <app-pod> -- nslookup db   # 4) does DNS resolve?
kubectl describe svc db                # 5) correct port / targetPort?
```
**Why:** walk the chain **pod running → Service exists → Endpoints populated → DNS resolves → correct port**, and the **first step that fails is the broken link**. Empty Endpoints points at a **label/readiness** problem; DNS failure at **naming/namespace**; everything green but still failing points at a **port/targetPort** mismatch or the app not listening.

### 42. Read state / lastState / restartCount for repeated restarts
```bash
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].restartCount}{"\n"}'
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].state}{"\n"}'
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}{" "}{.status.containerStatuses[0].lastState.terminated.exitCode}{"\n"}'
```
**Why:** **`state`** is the container *now* (Running/Waiting/Terminated), **`lastState`** is how it ended *last time* (reason + exit code — e.g. `OOMKilled`, `Error` exit 1), and **`restartCount`** shows how often it's flapping. Together they tell you whether restarts are from OOM, crashes, or probe failures — the root cause of `CrashLoopBackOff`.

### 43. OpenShift Route vs Kubernetes Ingress
| | **Ingress** (Kubernetes) | **Route** (OpenShift) |
|---|---|---|
| Origin | Standard K8s object | OpenShift-native (predates Ingress) |
| Controller | Needs a separately installed **Ingress controller** | Built-in **HAProxy router**, ready out of the box |
| Hostname | You specify it | Can be **auto-generated** |
| TLS | Edge (via a TLS Secret) | **edge**, **passthrough**, **re-encrypt** |
| Portability | Portable across any K8s | OpenShift-specific |

**Explanation:** both expose HTTP(S) services to the outside and do host/path routing. **Ingress** is the portable Kubernetes standard but **inert without a controller**. **Route** is OpenShift's older, built-in equivalent backed by an integrated router, with richer TLS modes (notably **re-encrypt**) and automatic hostnames. OpenShift supports **both** — it transparently turns Ingress objects into Routes — so you get portability *and* the native features.

---

*End of LO5 answer key. Master shortcut for the exam: **containers** fail because **PID 1 exits** or because of **ports / env / mounts / limits**; **Kubernetes** problems are always read from `kubectl describe` **Events** or `kubectl logs --previous`, and connectivity issues follow the **pod → Service → Endpoints → DNS → port** chain.*
