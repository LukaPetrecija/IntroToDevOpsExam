# LO5 — Solve Problems with Application Shipping by Using Containers

This document gives troubleshooting-style answers for LO5.  
For each broken command or manifest, it identifies:

1. **Mistake**
2. **Fixed command or manifest**
3. **Explanation**
4. **Useful verification commands**

Assumptions:

- You are using `podman` for local containers.
- You are using `kubectl` with a working Kubernetes or Minikube cluster.
- Commands are written for Linux/macOS shell. Windows PowerShell may require small quoting changes.
- Replace fake names, passwords, image tags, and paths with your real values.
- For private registries, never commit real credentials to GitHub.

---

## 1. Wrong Podman port mapping order

Broken command:

```bash
podman run -d --name web -p 80:8080 docker.io/library/nginx
```

### Mistake

The `-p` format is:

```text
HOST_PORT:CONTAINER_PORT
```

The command maps host port `80` to container port `8080`.

But the official nginx image listens on container port `80`, not `8080`.

### Fixed command

If you want to access nginx on host port `8080`:

```bash
podman rm -f web 2>/dev/null || true

podman run -d --name web -p 8080:80 docker.io/library/nginx
```

Then test:

```bash
curl http://localhost:8080
```

If you want to access nginx on host port `80`:

```bash
podman rm -f web 2>/dev/null || true

podman run -d --name web -p 80:80 docker.io/library/nginx
```

### Explanation

`-p 8080:80` means:

```text
host port 8080 → container port 80
```

The original command reversed the expected host/container mapping.

---

## 2. MySQL container exits because `MYSQL_ROOT_PASSWORD` has no value

Broken command:

```bash
podman run -d --name db -e MYSQL_ROOT_PASSWORD docker.io/library/mysql
```

### Mistake

`-e MYSQL_ROOT_PASSWORD` sets the variable name but does not provide a real password value. MySQL initialization requires a root password or another explicit initialization option.

### Diagnose

```bash
podman logs db
```

You will usually see an error similar to:

```text
Database is uninitialized and password option is not specified
```

### Fixed command

```bash
podman rm -f db 2>/dev/null || true

podman run -d --name db \
  -e MYSQL_ROOT_PASSWORD='StrongPassword123!' \
  docker.io/library/mysql:8
```

Verify:

```bash
podman ps
podman logs db
```

### Development-only alternative

For a throwaway lab container only:

```bash
podman run -d --name db \
  -e MYSQL_ALLOW_EMPTY_PASSWORD=yes \
  docker.io/library/mysql:8
```

Do **not** use empty passwords in production.

---

## 3. `-p` is ignored with `--network host`

Broken command:

```bash
podman run -d --name app --network host -p 8080:80 docker.io/library/nginx
```

### Mistake

`--network host` makes the container use the host network namespace. Port publishing with `-p` is not meaningful because there is no separate container network to forward into.

### Fixed command option A: use normal port publishing

```bash
podman rm -f app 2>/dev/null || true

podman run -d --name app -p 8080:80 docker.io/library/nginx
```

Test:

```bash
curl http://localhost:8080
```

### Fixed command option B: use host network and no `-p`

```bash
podman rm -f app 2>/dev/null || true

podman run -d --name app --network host docker.io/library/nginx
```

Then nginx listens directly on the host network, usually port `80`:

```bash
curl http://localhost:80
```

### Explanation

`-p` publishes ports from an isolated container network to the host. With host networking, the container already shares the host network, so the publish rule is effectively ignored.

---

## 4. BusyBox exits immediately

Broken command:

```bash
podman run -d --name c1 docker.io/library/busybox
```

### Mistake

BusyBox has no long-running default foreground service. The container starts, runs the default command, then exits successfully with status `0`.

### Diagnose

```bash
podman ps -a
podman logs c1
```

You will see the container in `Exited (0)`.

### Fixed command

Run a long-running foreground process:

```bash
podman rm -f c1 2>/dev/null || true

podman run -d --name c1 docker.io/library/busybox sleep 1d
```

Alternative:

```bash
podman run -d --name c1 docker.io/library/busybox sh -c "while true; do sleep 3600; done"
```

Verify:

```bash
podman ps
```

### Explanation

A container lives only as long as its main process. When PID 1 exits, the container stops.

---

## 5. Detached `--rm` container disappears before logs can be read

Broken command:

```bash
podman run --rm -d --name job docker.io/library/alpine echo hello
```

Then:

```bash
podman logs job
```

fails.

### Mistake

The container runs `echo hello`, exits immediately, and `--rm` removes it automatically. Because it is detached, you do not see the output in your terminal, and because it is removed, `podman logs job` cannot find it.

### Fixed command option A: do not detach

```bash
podman run --rm --name job docker.io/library/alpine echo hello
```

Output:

```text
hello
```

### Fixed command option B: keep it after exit so logs are available

```bash
podman rm -f job 2>/dev/null || true

podman run -d --name job docker.io/library/alpine echo hello
podman logs job
podman rm job
```

### Explanation

`--rm` is useful for temporary foreground jobs. Combining `--rm`, `-d`, and a very short command often removes the container before you can inspect it.

---

## 6. MySQL with `--memory 8m` never becomes healthy

Broken command:

```bash
podman run -d -p 8080:80 --memory 8m docker.io/library/mysql
```

### Mistakes

There are two problems:

1. `--memory 8m` is far too low for MySQL.
2. MySQL does not serve HTTP on port `80`, so `-p 8080:80` is also the wrong port mapping for a database.

### Diagnose

```bash
podman ps -a
podman logs <container-id-or-name>
```

You may see memory allocation errors, initialization failure, or restarts.

### Fixed command

```bash
podman rm -f mysqltest 2>/dev/null || true

podman run -d --name mysqltest \
  -e MYSQL_ROOT_PASSWORD='StrongPassword123!' \
  --memory 512m \
  -p 3306:3306 \
  docker.io/library/mysql:8
```

Verify:

```bash
podman ps
podman logs mysqltest
```

### Explanation

Databases need enough memory to initialize and run. An 8 MiB memory limit is not realistic for MySQL. Also, MySQL listens on port `3306`, not port `80`.

---

## 7. Container name resolution fails on the default network

Broken commands:

```bash
podman run -d --name a docker.io/library/alpine sleep 1d
podman run -d --name b docker.io/library/alpine sleep 1d
podman exec a ping b
```

### Mistake

Container-to-container DNS name resolution is not always enabled on Podman's default network, especially depending on rootless/rootful mode and network backend. The containers may be on a network where DNS aliases are not provided.

### Diagnose

```bash
podman network ls
podman inspect a --format '{{json .NetworkSettings.Networks}}'
podman inspect b --format '{{json .NetworkSettings.Networks}}'
```

### Fixed command

Create a user-defined bridge network and attach both containers to it:

```bash
podman rm -f a b 2>/dev/null || true
podman network rm labnet 2>/dev/null || true

podman network create labnet

podman run -d --name a --network labnet docker.io/library/alpine sleep 1d
podman run -d --name b --network labnet docker.io/library/alpine sleep 1d

podman exec a ping -c 3 b
```

### Explanation

User-defined Podman networks can provide DNS name resolution between containers. On a suitable network with DNS enabled, the container name `b` resolves from container `a`.

---

## 8. Bind mount files not visible or SELinux denies access

Broken command:

```bash
podman run -d --name web -v ./html:/usr/share/nginx/html docker.io/library/nginx
```

### Mistake

On SELinux-enabled systems such as Fedora, RHEL, CentOS Stream, or OpenShift-like hosts, the host directory may not have a label that allows container access.

Also, relative paths can be confusing. Use an absolute path for clarity.

### Fixed command for a private mount

```bash
podman rm -f web 2>/dev/null || true

mkdir -p ./html
echo "hello from bind mount" > ./html/index.html

podman run -d --name web \
  -p 8080:80 \
  -v "$(pwd)/html:/usr/share/nginx/html:Z" \
  docker.io/library/nginx
```

Test:

```bash
curl http://localhost:8080
```

### Fixed command for a shared mount

Use `:z` if multiple containers need to share the same host directory:

```bash
podman run -d --name web \
  -p 8080:80 \
  -v "$(pwd)/html:/usr/share/nginx/html:z" \
  docker.io/library/nginx
```

### Explanation

- `:Z` relabels the host path for private use by one container.
- `:z` relabels the host path for shared use by multiple containers.
- Without these flags, SELinux may block the container from reading the mounted files.

---

# Containerfile / Dockerfile Troubleshooting

## 9. General method for each Containerfile/Dockerfile mistake

When a build or image behaves incorrectly, check:

```bash
podman build -t test-image .
podman run --rm test-image
podman inspect test-image
```

Helpful questions:

- Does the image install dependencies correctly?
- Is the build cache being used efficiently?
- Does the container run a proper foreground command?
- Is `CMD` using exec form where signal handling matters?
- Are ports documented correctly?
- Is the final image unnecessarily large?
- Are files copied with the right ownership?
- Is a non-root user created before switching to it?

---

## 10. Debian image does not run `apt-get update`

Broken Containerfile:

```Dockerfile
FROM debian:12
RUN apt-get install -y nginx
```

### Mistake

The package index is not updated before installing. A fresh Debian image may not have current package lists.

### Fixed Containerfile

```Dockerfile
FROM debian:12

RUN apt-get update \
    && apt-get install -y --no-install-recommends nginx \
    && rm -rf /var/lib/apt/lists/*
```

### Explanation

`apt-get update` downloads package indexes. `apt-get install` uses those indexes. They should be in the same `RUN` layer so cached stale indexes are not reused.

---

## 11. Python app uses shell-form `CMD`

Broken Containerfile:

```Dockerfile
FROM python:3.11
COPY app.py /app/app.py
WORKDIR /app
CMD python app.py
```

### Mistake

`CMD python app.py` is shell form. It starts through `/bin/sh -c`, so the Python process may not receive Unix signals directly as PID 1. This can cause poor shutdown behavior.

### Fixed Containerfile

```Dockerfile
FROM python:3.11

WORKDIR /app
COPY app.py /app/app.py

CMD ["python", "app.py"]
```

### Explanation

Exec form is preferred for long-running applications:

```Dockerfile
CMD ["python", "app.py"]
```

The application process becomes PID 1 and receives signals such as `SIGTERM` more directly.

---

## 12. Node build invalidates `npm install` cache on every source change

Broken Containerfile:

```Dockerfile
FROM node:20
COPY . /app
WORKDIR /app
RUN npm install
```

### Mistake

The whole source tree is copied before `npm install`. Any source file change invalidates the cache for the dependency installation layer.

### Fixed Containerfile

```Dockerfile
FROM node:20

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

CMD ["npm", "start"]
```

### Better production option

If `package-lock.json` exists:

```Dockerfile
RUN npm ci
```

### Explanation

Dependency metadata changes less often than source code. Copying `package.json` and `package-lock.json` first lets the build cache reuse `npm install` or `npm ci` when only application source files change.

---

## 13. Setting `PATH=/app/bin` breaks standard commands

Broken Containerfile:

```Dockerfile
FROM alpine:3.20
ENV PATH=/app/bin
RUN apk add --no-cache curl
```

### Mistake

This replaces the whole `PATH` with `/app/bin`. Standard directories such as `/bin`, `/usr/bin`, and `/sbin` are removed. After that, commands like `sh`, `apk`, or `curl` may not be found.

### Fixed Containerfile

```Dockerfile
FROM alpine:3.20

ENV PATH="/app/bin:${PATH}"

RUN apk add --no-cache curl
```

### Explanation

Always append or prepend to the existing `PATH` instead of replacing it completely.

Correct pattern:

```Dockerfile
ENV PATH="/custom/path:${PATH}"
```

---

## 14. `EXPOSE` says 8080, but the app listens on 3000

Broken Containerfile:

```Dockerfile
FROM ubuntu:24.04
EXPOSE 8080
CMD ["python3", "-m", "http.server", "3000"]
```

### Mistake

`EXPOSE 8080` documents port `8080`, but the Python HTTP server listens on port `3000`.

### Fixed Containerfile option A: make the app listen on 8080

```Dockerfile
FROM ubuntu:24.04

RUN apt-get update \
    && apt-get install -y --no-install-recommends python3 \
    && rm -rf /var/lib/apt/lists/*

EXPOSE 8080
CMD ["python3", "-m", "http.server", "8080"]
```

Run:

```bash
podman build -t pyserver .
podman run --rm -p 8080:8080 pyserver
```

### Fixed Containerfile option B: document and map port 3000

```Dockerfile
FROM ubuntu:24.04

RUN apt-get update \
    && apt-get install -y --no-install-recommends python3 \
    && rm -rf /var/lib/apt/lists/*

EXPOSE 3000
CMD ["python3", "-m", "http.server", "3000"]
```

Run:

```bash
podman build -t pyserver .
podman run --rm -p 8080:3000 pyserver
```

### Explanation

The exposed port and the actual application listening port should match, or the runtime mapping must map host port to the actual container port.

---

## 15. `-p 8080:8080` does not work when the process listens on 3000

Broken run idea:

```bash
podman run -p 8080:8080 pyserver
```

### Mistake

The container process listens on port `3000`, but the command maps host port `8080` to container port `8080`.

### Fixed command

If the app listens on container port `3000`:

```bash
podman run --rm -p 8080:3000 pyserver
```

Then test:

```bash
curl http://localhost:8080
```

### What `EXPOSE` actually does

`EXPOSE` is documentation metadata. It does **not** publish the port to the host by itself. You still need `-p` or `--publish` when running the container.

---

## 16. Debian image is huge because package cache is kept

Broken Containerfile:

```Dockerfile
FROM debian:12
RUN apt-get update && apt-get install -y build-essential
```

### Mistake

The apt package lists remain in the image. Also, recommended packages may increase image size.

### Fixed Containerfile

```Dockerfile
FROM debian:12

RUN apt-get update \
    && apt-get install -y --no-install-recommends build-essential \
    && rm -rf /var/lib/apt/lists/*
```

### Explanation

Cleaning must happen in the same `RUN` layer. If you clean in a later layer, the earlier layer still contains the package index data.

For build tools, a multi-stage build is often even better because build dependencies do not need to remain in the runtime image.

---

## 17. Non-root user is selected before copying/installing files

Broken Containerfile:

```Dockerfile
FROM python:3.11
USER appuser
COPY requirements.txt /app/requirements.txt
RUN pip install -r /app/requirements.txt
```

### Mistakes

1. `appuser` may not exist.
2. The image switches to the non-root user too early.
3. `/app` may not exist or may not be writable.
4. `pip install` may try to install globally without permission.
5. `COPY` files may be owned by root unless ownership is specified.

### Fixed Containerfile

```Dockerfile
FROM python:3.11

RUN useradd --create-home --shell /bin/bash appuser

WORKDIR /app

COPY requirements.txt /app/requirements.txt

RUN pip install --no-cache-dir -r /app/requirements.txt

COPY --chown=appuser:appuser . /app

USER appuser

CMD ["python", "app.py"]
```

### Alternative: install into a virtual environment

```Dockerfile
FROM python:3.11

RUN useradd --create-home --shell /bin/bash appuser

WORKDIR /app

COPY requirements.txt .

RUN python -m venv /opt/venv \
    && /opt/venv/bin/pip install --no-cache-dir -r requirements.txt

ENV PATH="/opt/venv/bin:${PATH}"

COPY --chown=appuser:appuser . .

USER appuser

CMD ["python", "app.py"]
```

### Explanation

Install system/global dependencies before switching to a non-root user. Copy application files with correct ownership if the non-root user needs to read or write them.

---

## 18. Go final image is huge because it includes the full compiler image

Broken Containerfile:

```Dockerfile
FROM golang:1.22
COPY . /src
WORKDIR /src
RUN go build -o app .
CMD ["/src/app"]
```

### Mistake

The runtime image includes the Go compiler, build cache, source code, and build tools. This can easily be around 1 GB.

### Fixed multi-stage Containerfile

```Dockerfile
FROM golang:1.22 AS builder

WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /out/app .

FROM alpine:3.20

COPY --from=builder /out/app /app

CMD ["/app"]
```

### Even smaller option

For a statically linked Go binary:

```Dockerfile
FROM golang:1.22 AS builder

WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /out/app .

FROM scratch

COPY --from=builder /out/app /app

CMD ["/app"]
```

### Explanation

Multi-stage builds separate the build environment from the runtime environment. The final image contains only what is needed to run the app.

---

# Kubernetes Troubleshooting

## 19. Pod is stuck `Pending`

### Diagnose

```bash
kubectl get pod <pod-name>
kubectl describe pod <pod-name>
```

Look at the `Events` section.

Common causes:

### Scheduling problem

Event example:

```text
0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector.
```

Check:

```bash
kubectl get nodes --show-labels
kubectl describe node <node-name>
```

Fix:

- Correct `nodeSelector` or affinity.
- Add the required label to a node.

```bash
kubectl label node <node-name> disktype=ssd
```

### Resource problem

Event example:

```text
0/3 nodes are available: 3 Insufficient cpu.
```

Fix:

- Lower resource requests.
- Add more nodes.
- Free capacity.

```bash
kubectl describe node <node-name>
kubectl top nodes
kubectl top pods
```

### PVC problem

Event example:

```text
pod has unbound immediate PersistentVolumeClaims
```

Fix:

```bash
kubectl get pvc
kubectl describe pvc <pvc-name>
kubectl get storageclass
```

Create or fix the PVC or StorageClass.

### Explanation

`Pending` means the Pod has been accepted by the API server but has not successfully started on a node. The `describe pod` events usually show the exact reason.

---

## 20. YAML indentation error after removing spaces

Example broken manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bad-web
spec:
replicas: 2
  selector:
    matchLabels:
      app: bad-web
```

### Mistake

YAML is indentation-sensitive. `replicas` is incorrectly placed at the same level as `spec`, and the remaining indentation becomes invalid.

### Diagnose

```bash
kubectl apply -f bad-web.yaml
```

Possible error:

```text
error: error parsing bad-web.yaml: error converting YAML to JSON: yaml: line X: did not find expected key
```

### Fixed manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bad-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: bad-web
  template:
    metadata:
      labels:
        app: bad-web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

### Validate before applying

```bash
kubectl apply --dry-run=server --validate=true -f bad-web.yaml
```

### Explanation

YAML structure is based on spaces. Removing spaces can move a field into the wrong object or make the file impossible to parse.

---

## 21. Pod is in `ImagePullBackOff`

### Diagnose

```bash
kubectl get pod <pod-name>
kubectl describe pod <pod-name>
```

Look for events such as:

```text
Failed to pull image
ErrImagePull
ImagePullBackOff
pull access denied
manifest unknown
```

### Common cause A: image typo

Broken:

```yaml
image: nginxx:1.27
```

Fixed:

```yaml
image: nginx:1.27
```

Apply:

```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
```

### Common cause B: missing or wrong tag

Broken:

```yaml
image: nginx:no-such-tag
```

Fixed:

```yaml
image: nginx:1.27
```

### Common cause C: private registry without pull secret

Create pull Secret:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=<registry-server> \
  --docker-username=<username> \
  --docker-password=<password-or-token> \
  --docker-email=<email>
```

Reference it:

```yaml
spec:
  imagePullSecrets:
    - name: regcred
```

### Explanation

`ImagePullBackOff` means Kubernetes tried to pull the image, failed, and is backing off before retrying.

---

## 22. Pod is in `CrashLoopBackOff`

### Diagnose

```bash
kubectl get pod <pod-name>
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

If the Pod has multiple containers:

```bash
kubectl logs <pod-name> -c <container-name> --previous
```

### Example broken container

```yaml
containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "cat /missing/file"]
```

The container exits immediately because the command fails.

### Fix

Correct the command or provide the missing file:

```yaml
containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo started; sleep 3600"]
```

### Explanation

`CrashLoopBackOff` means the container repeatedly starts, crashes, and Kubernetes waits longer between restarts. `--previous` is important because the current container may have already restarted and lost the visible crash output.

---

## 23. Pod was `OOMKilled`

### Diagnose

```bash
kubectl describe pod <pod-name>
```

Look for:

```text
Last State: Terminated
Reason: OOMKilled
Exit Code: 137
```

JSON/YAML check:

```bash
kubectl get pod <pod-name> -o yaml | grep -A20 lastState
```

JSONPath:

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].lastState}{"\n"}'
```

### Fix option A: raise memory limit

```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"
```

Apply:

```bash
kubectl apply -f fixed-pod-or-deployment.yaml
```

### Fix option B: fix the app

Examples:

- Reduce memory usage.
- Fix memory leaks.
- Configure JVM/Node/Python memory settings.
- Stream data instead of loading all of it into RAM.

### Explanation

`OOMKilled` means the container exceeded its memory limit and the kernel killed the process.

---

## 24. Deployment Pods never become Ready because readiness probe fails

### Diagnose

```bash
kubectl get deployment <deployment-name>
kubectl get pods -l app=<label>
kubectl describe pod <pod-name>
```

Look for events like:

```text
Readiness probe failed
connection refused
HTTP probe failed with statuscode: 404
```

### Example broken readiness probe

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
```

But the app only serves `/` on port `80`.

### Fixed probe for nginx

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
```

### Verify

```bash
kubectl rollout status deployment/<deployment-name>
kubectl describe pod <pod-name> | grep -A8 Readiness
kubectl get endpoints <service-name>
```

### Explanation

A Pod can be `Running` but not `Ready`. Services only send traffic to ready endpoints. A bad readiness probe can remove healthy Pods from Service traffic.

---

## 25. Service returns nothing because labels do not match

### Diagnose

```bash
kubectl get svc <service-name> -o yaml
kubectl get pods --show-labels
kubectl get endpoints <service-name>
kubectl get endpointslice -l kubernetes.io/service-name=<service-name>
```

If endpoints are empty:

```text
ENDPOINTS   <none>
```

the Service selector is probably wrong or Pods are not Ready.

### Example broken Service

Deployment Pods:

```yaml
labels:
  app: web
```

Service selector:

```yaml
selector:
  app: nginx
```

### Fixed Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f web-svc.yaml
kubectl get endpoints web-svc
```

### Explanation

A Service routes to Pods selected by labels. If the selector does not match any ready Pods, the Service has no endpoints and returns nothing.

---

## 26. Deployment selector does not match Pod template labels

Broken manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mismatch
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

### Error

```bash
kubectl apply -f mismatch.yaml
```

Expected error:

```text
selector does not match template labels
```

### Fixed manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mismatch
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

### Explanation

The Deployment selector must match the labels on the Pod template. Otherwise, the Deployment could not reliably manage the Pods it creates.

---

## 27. Resource requests exceed every node's capacity

Broken manifest example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: too-large
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      resources:
        requests:
          cpu: "100"
          memory: "500Gi"
```

### Diagnose

```bash
kubectl apply -f too-large.yaml
kubectl describe pod too-large
```

Expected event:

```text
0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory.
```

### Fix

Use realistic requests:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

Apply:

```bash
kubectl apply -f fixed.yaml
```

### Explanation

The scheduler places Pods based on `resources.requests`, not limits. If no node has enough allocatable CPU or memory for the request, the Pod stays `Pending`.

---

## 28. Pod mounts a PVC that does not exist

Broken Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-missing
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: missing-pvc
```

### Diagnose

```bash
kubectl apply -f pvc-missing.yaml
kubectl describe pod pvc-missing
```

Expected event:

```text
persistentvolumeclaim "missing-pvc" not found
```

### Fixed PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: missing-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply:

```bash
kubectl apply -f pvc.yaml
kubectl get pvc
kubectl get pod pvc-missing
```

### Explanation

PVCs are namespace-scoped. The Pod can only mount a PVC that exists in the same namespace.

---

## 29. Ubuntu container with no long-running command completes

Example broken Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-empty
spec:
  restartPolicy: Always
  containers:
    - name: ubuntu
      image: ubuntu:24.04
```

### Mistake

Ubuntu is a base image. It does not run a server by default. Without a long-running command, the container exits. With `restartPolicy: Always`, Kubernetes restarts it repeatedly, which can become `CrashLoopBackOff`.

### Fixed Pod for debugging

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-debug
spec:
  containers:
    - name: ubuntu
      image: ubuntu:24.04
      command: ["sleep", "infinity"]
```

### Fixed Pod for a real app

Run your real app command, for example:

```yaml
command: ["python3", "-m", "http.server", "8080"]
```

### Explanation

Containers are not virtual machines. They need a foreground process. When the main process exits, the container exits.

---

## 30. Use events to triage what happened to a Pod over time

Command:

```bash
kubectl get events --sort-by=.lastTimestamp
```

Filter by involved object name:

```bash
kubectl get events --sort-by=.lastTimestamp \
  --field-selector involvedObject.name=<pod-name>
```

Newer command if available:

```bash
kubectl events --for pod/<pod-name>
```

Useful full diagnosis flow:

```bash
kubectl get pod <pod-name> -o wide
kubectl describe pod <pod-name>
kubectl get events --sort-by=.lastTimestamp \
  --field-selector involvedObject.name=<pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

### Explanation

Events show scheduling, image pull attempts, mount failures, probe failures, restarts, and other timeline information. They are often the fastest way to understand why a Pod changed state.

---

## 31. Use `kubectl exec` to debug DNS inside a Pod

Start a debug Pod:

```bash
kubectl run dns-debug --image=busybox:1.36 --restart=Never --command -- sleep 3600
```

Open shell:

```bash
kubectl exec -it dns-debug -- sh
```

Inside:

```sh
cat /etc/resolv.conf
nslookup kubernetes.default.svc.cluster.local
nslookup <service-name>.<namespace>.svc.cluster.local
exit
```

Cleanup:

```bash
kubectl delete pod dns-debug
```

### Explanation

`/etc/resolv.conf` shows cluster DNS search domains and nameserver. `nslookup` proves whether Service DNS names resolve from inside the cluster.

---

## 32. Use an ephemeral debug container for a distroless/no-shell container

Problem:

Some containers are distroless or minimal and do not include `sh`, `bash`, `curl`, `nslookup`, or package managers.

### Debug command

```bash
kubectl debug -it <pod-name> \
  --image=busybox:1.36 \
  --target=<container-name> \
  -- sh
```

Inside the ephemeral container:

```sh
ps
cat /etc/resolv.conf
nslookup kubernetes.default.svc.cluster.local
wget -qO- http://<service-name>:<port>
exit
```

### Explanation

An ephemeral debug container is temporarily added to the existing Pod for troubleshooting. `--target` asks Kubernetes to target the namespaces of the existing container when supported by the runtime.

This is useful when the original application container has no shell.

---

## 33. Use a throwaway Pod to test Service connectivity from inside the cluster

Command:

```bash
kubectl run tmp --rm -it --restart=Never --image=busybox:1.36 -- sh
```

Inside:

```sh
nslookup <service-name>
wget -qO- http://<service-name>:<port>
nc -vz <service-name> <port>
exit
```

Example:

```sh
nslookup web-svc
wget -qO- http://web-svc:80
```

### Explanation

Testing from inside the cluster separates internal Service/DNS problems from external access problems such as NodePort, Ingress, firewall, or Minikube tunnel issues.

---

## 34. Node is `NotReady`

### Check nodes

```bash
kubectl get nodes -o wide
kubectl describe node <node-name>
```

Look for conditions:

```text
Ready
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
```

### Common checks

#### Kubelet

On a normal Linux node:

```bash
sudo systemctl status kubelet
sudo journalctl -u kubelet -xe
```

On Minikube:

```bash
minikube status
minikube logs
```

If using SSH into Minikube:

```bash
minikube ssh
sudo journalctl -u kubelet -xe
```

#### Disk pressure

```bash
kubectl describe node <node-name>
```

Inside node:

```bash
df -h
sudo crictl images
sudo crictl ps -a
```

#### CNI/network plugin

```bash
kubectl get pods -n kube-system
kubectl describe node <node-name>
kubectl logs -n kube-system <cni-pod-name>
```

### Explanation

`NotReady` means the control plane cannot treat the node as healthy for scheduling/running workloads. Common root causes include kubelet down, disk pressure, container runtime problems, or CNI/network plugin failure.

---

## 35. Pod stuck `Terminating`

### Diagnose

```bash
kubectl get pod <pod-name> -o yaml
kubectl describe pod <pod-name>
```

Check:

- `metadata.finalizers`
- `deletionTimestamp`
- long `terminationGracePeriodSeconds`
- node availability
- stuck volume detach/unmount
- application not handling SIGTERM

### Graceful delete

```bash
kubectl delete pod <pod-name>
```

### Force delete

```bash
kubectl delete pod <pod-name> --grace-period=0 --force
```

### Explanation

A Pod can stay `Terminating` while Kubernetes waits for finalizers, graceful shutdown, or cleanup. Finalizers are keys that tell Kubernetes not to fully delete an object until a controller has performed cleanup.

### Risk of force deletion

Force deletion removes the API object immediately. The container may still be running temporarily on an unreachable node. This can cause duplicate processes, storage corruption, or unsafe cleanup if used carelessly.

Use force deletion only when you understand the risk.

---

## 36. Wrong `apiVersion` / `kind` pairing

Broken manifest:

```yaml
apiVersion: apps/v1
kind: Pod
metadata:
  name: bad-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

### Mistake

`Pod` belongs to core API group `v1`, not `apps/v1`.

### Diagnose

```bash
kubectl apply -f bad-pod.yaml
```

Expected error idea:

```text
no matches for kind "Pod" in version "apps/v1"
```

### Fixed manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

### Common correct pairings

```text
Pod                  → v1
Service              → v1
ConfigMap            → v1
Secret               → v1
PersistentVolumeClaim → v1
Deployment           → apps/v1
StatefulSet          → apps/v1
DaemonSet            → apps/v1
Job                  → batch/v1
CronJob              → batch/v1
Ingress              → networking.k8s.io/v1
```

---

## 37. `CreateContainerConfigError` because a `configMapKeyRef` key is missing

Broken manifest example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-settings
data:
  APP_MODE: production
---
apiVersion: v1
kind: Pod
metadata:
  name: cm-key-error
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo $LOG_LEVEL; sleep 3600"]
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-settings
              key: LOG_LEVEL
```

### Diagnose

```bash
kubectl apply -f cm-key-error.yaml
kubectl get pod cm-key-error
kubectl describe pod cm-key-error
```

Expected event:

```text
couldn't find key LOG_LEVEL in ConfigMap default/app-settings
```

### Fix option A: add the missing key

```bash
kubectl create configmap app-settings \
  --from-literal=APP_MODE=production \
  --from-literal=LOG_LEVEL=info \
  -o yaml --dry-run=client | kubectl apply -f -
```

Restart Pod if needed:

```bash
kubectl delete pod cm-key-error
kubectl apply -f cm-key-error.yaml
```

### Fix option B: correct the key name in the Pod

```yaml
key: APP_MODE
```

### Fix option C: make it optional

```yaml
configMapKeyRef:
  name: app-settings
  key: LOG_LEVEL
  optional: true
```

### Explanation

The container cannot be configured because Kubernetes cannot resolve the requested ConfigMap key.

---

## 38. Pod cannot mount Secret from another namespace

### Mistake

Secrets are namespace-scoped. A Pod can only reference a Secret in the same namespace.

Broken example:

```text
Secret exists in namespace: default
Pod runs in namespace: app
```

Pod:

```yaml
volumes:
  - name: secret
    secret:
      secretName: db-creds
```

### Diagnose

```bash
kubectl get secret db-creds -n default
kubectl get secret db-creds -n app
kubectl describe pod <pod-name> -n app
```

Expected event:

```text
secret "db-creds" not found
```

### Fix

Create or copy the Secret into the same namespace as the Pod.

Example using literals:

```bash
kubectl create namespace app 2>/dev/null || true

kubectl create secret generic db-creds \
  -n app \
  --from-literal=username=dbuser \
  --from-literal=password='p@ssw0rd'
```

Or export and apply into another namespace:

```bash
kubectl get secret db-creds -n default -o yaml \
  | sed 's/namespace: default/namespace: app/' \
  | kubectl apply -f -
```

Be careful not to expose real secret data in scripts or Git.

### Explanation

Kubernetes deliberately scopes Secrets by namespace to reduce accidental cross-application access.

---

## 39. Rollout stuck with `ProgressDeadlineExceeded`

### Diagnose

```bash
kubectl rollout status deployment/<deployment-name>
kubectl describe deployment <deployment-name>
kubectl get rs -l app=<label>
kubectl get pods -l app=<label>
kubectl describe pod <pod-name>
```

Expected condition:

```text
Progressing=False
Reason=ProgressDeadlineExceeded
```

### Common causes

- Bad image tag causing `ImagePullBackOff`
- New Pods crash with `CrashLoopBackOff`
- Readiness probe never succeeds
- Resource requests cannot be scheduled
- PVC mount failure
- Bad config/secret reference
- App starts but listens on the wrong port

### Example remediation

If the image tag is bad:

```bash
kubectl set image deployment/<deployment-name> <container-name>=nginx:1.27
kubectl rollout status deployment/<deployment-name>
```

Or roll back:

```bash
kubectl rollout undo deployment/<deployment-name>
kubectl rollout status deployment/<deployment-name>
```

### Explanation

`ProgressDeadlineExceeded` means the Deployment did not make progress within `.spec.progressDeadlineSeconds`. Fix the cause or roll back to the last working revision.

---

## 40. Catch indentation and field typos before applying

Examples of mistakes:

```yaml
imagePullpolicy: Always
```

Wrong capitalization. Correct field:

```yaml
imagePullPolicy: Always
```

Another mistake:

```yaml
containers:
  - name: nginx
    image: nginx:1.27
ports:
  - containerPort: 80
```

Here `ports` is misnested. It should be inside the container item.

### Validate before applying

```bash
kubectl apply --dry-run=server --validate=true -f manifest.yaml
```

Also useful:

```bash
kubectl explain deployment.spec.template.spec.containers.imagePullPolicy
kubectl explain pod.spec.containers.ports
```

### Fixed example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: valid-nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 80
```

### Explanation

`--dry-run=server` asks the API server to validate the manifest without saving it. It catches many schema errors and wrong fields before they break a real deployment.

---

## 41. App logs say it cannot reach its database Service

Use this order.

### 1. App Pod is running

```bash
kubectl get pod <app-pod> -o wide
kubectl logs <app-pod>
```

### 2. Database Service exists

```bash
kubectl get svc db
kubectl describe svc db
```

### 3. Endpoints are populated

```bash
kubectl get endpoints db
kubectl get endpointslice -l kubernetes.io/service-name=db
```

If endpoints are empty, check selector labels:

```bash
kubectl get svc db -o jsonpath='{.spec.selector}{"\n"}'
kubectl get pods --show-labels
```

### 4. DNS resolves

From the app Pod if it has shell tools:

```bash
kubectl exec -it <app-pod> -- sh
```

Inside:

```sh
nslookup db
nslookup db.<namespace>.svc.cluster.local
exit
```

If the app image has no shell, use a throwaway debug Pod:

```bash
kubectl run tmp --rm -it --restart=Never --image=busybox:1.36 -- sh
```

Inside:

```sh
nslookup db
exit
```

### 5. Correct port

Check Service port and targetPort:

```bash
kubectl get svc db -o yaml
kubectl get pod <db-pod> -o yaml | grep -A10 ports:
```

Test from a debug Pod:

```sh
nc -vz db 5432
```

or for MySQL:

```sh
nc -vz db 3306
```

### Identify the broken link

Examples:

- App Pod not running → fix app crash.
- Service missing → create Service.
- Endpoints empty → fix selector labels or readiness probe.
- DNS fails → check CoreDNS.
- Port fails → fix Service `targetPort` or app listening port.

---

## 42. Read container state, lastState, and restartCount with JSONPath

Commands:

```bash
kubectl get pod <pod-name> \
  -o jsonpath='{range .status.containerStatuses[*]}name={.name} state={.state} lastState={.lastState} restarts={.restartCount}{"\n"}{end}'
```

Readable version:

```bash
kubectl get pod <pod-name> -o jsonpath='
{range .status.containerStatuses[*]}
Container: {.name}
State: {.state}
LastState: {.lastState}
RestartCount: {.restartCount}
{end}
'
```

For all Pods:

```bash
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{" "}{range .status.containerStatuses[*]}{.name}{" restarts="}{.restartCount}{" state="}{.state}{" "}{end}{"\n"}{end}'
```

### How to interpret

- `restartCount` increasing → repeated crashes, failed probes, OOM kills, or process exits.
- `lastState.terminated.reason=OOMKilled` → memory limit issue.
- `lastState.terminated.exitCode=1` → application exited with error.
- `state.waiting.reason=CrashLoopBackOff` → repeated crash loop.
- `state.waiting.reason=ImagePullBackOff` → image pull problem.

### Explanation

`kubectl get pod -o jsonpath` is useful when you need exact status fields without reading the full YAML.

---

## 43. Compare OpenShift Route to Kubernetes Ingress

### Kubernetes Ingress

A Kubernetes `Ingress` exposes HTTP and HTTPS routes from outside the cluster to Services inside the cluster.

Basic example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  rules:
    - host: web.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 80
```

Important points:

- Ingress is a Kubernetes API object.
- It needs an Ingress Controller to actually route traffic.
- It supports host/path-based HTTP(S) routing.
- Behavior can vary depending on the controller, for example NGINX Ingress, Traefik, HAProxy, cloud provider controllers, etc.

### OpenShift Route

An OpenShift `Route` is an OpenShift-specific object for exposing Services externally.

Basic example:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: web
spec:
  host: web.example.com
  to:
    kind: Service
    name: web-svc
  port:
    targetPort: http
```

Secure edge-terminated route:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: web
spec:
  host: web.example.com
  to:
    kind: Service
    name: web-svc
  port:
    targetPort: http
  tls:
    termination: edge
```

### Main differences

| Topic | Kubernetes Ingress | OpenShift Route |
|---|---|---|
| API | Standard Kubernetes object | OpenShift-specific object |
| Controller | Requires an Ingress Controller | OpenShift includes router/Ingress infrastructure |
| Port/protocol focus | HTTP/HTTPS | HTTP/HTTPS, with OpenShift-specific routing features |
| TLS | Uses `tls` section and Secrets | Supports edge, passthrough, and re-encrypt termination |
| Portability | More portable across Kubernetes distributions | Best inside OpenShift |
| Developer experience | Depends on installed controller | Integrated with OpenShift tooling and `oc expose` |

### When to use which

Use **Ingress** when:

- You want a portable Kubernetes-native manifest.
- You are targeting standard Kubernetes clusters.
- Your organization has selected an Ingress Controller.

Use **OpenShift Route** when:

- You are deploying on OpenShift.
- You want the integrated OpenShift routing model.
- You need OpenShift route features such as edge, passthrough, or re-encrypt TLS termination.

### Explanation

OpenShift supports Kubernetes Ingress, but Routes are a long-standing OpenShift-native way to expose applications. In a migration from OpenShift to vanilla Kubernetes, Routes usually need to be converted to Ingress or Gateway API resources.

---

# Quick Troubleshooting Cheat Sheet

## Pod status

```bash
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

## Deployment rollout

```bash
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl describe deployment <name>
kubectl rollout undo deployment/<name>
```

## Events

```bash
kubectl get events --sort-by=.lastTimestamp
kubectl events --for pod/<pod-name>
```

## Services

```bash
kubectl get svc
kubectl describe svc <name>
kubectl get endpoints <name>
kubectl get endpointslice -l kubernetes.io/service-name=<name>
```

## DNS and connectivity

```bash
kubectl run tmp --rm -it --restart=Never --image=busybox:1.36 -- sh
```

Inside:

```sh
cat /etc/resolv.conf
nslookup <service-name>
wget -qO- http://<service-name>:<port>
nc -vz <service-name> <port>
```

## Image problems

```bash
kubectl describe pod <pod-name>
kubectl get secret
kubectl create secret docker-registry regcred ...
```

## Resource problems

```bash
kubectl describe pod <pod-name>
kubectl describe node <node-name>
kubectl top nodes
kubectl top pods
```

---

# References

Official documentation used as the basis for this answer:

- [Podman run manual](https://docs.podman.io/en/latest/markdown/podman-run.1.html)
- [Podman network create manual](https://docs.podman.io/en/latest/markdown/podman-network-create.1.html)
- [Podman volume mount options and SELinux labels](https://docs.podman.io/en/v4.6.1/markdown/options/volume.html)
- [Docker build best practices](https://docs.docker.com/build/building/best-practices/)
- [Docker build cache documentation](https://docs.docker.com/build/cache/)
- [Docker build cache optimization](https://docs.docker.com/build/cache/optimize/)
- [Kubernetes: Debug Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Kubernetes: Debug Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [Kubernetes: Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes: Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes: DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Kubernetes: Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Kubernetes: Ephemeral Containers](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/)
- [OpenShift Routes](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/ingress_and_load_balancing/routes)
