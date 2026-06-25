# Container Exam Mega-Script: LO4 + LO5 + LO6

A single GitHub-ready study guide for the exam topics:

- **LO4:** Kubernetes application delivery: Deployments, rollouts, workloads, Services, storage, ConfigMaps, Secrets, probes.
- **LO5:** Troubleshooting container shipping problems with Podman, Containerfiles/Dockerfiles, and Kubernetes debugging.
- **LO6:** Theoretical evaluation of Kubernetes, Docker, Podman, OpenShift, k3s, CI/CD, storage, networking, security, and enterprise suitability.

Use this document as a practical cheat sheet. For exam answers, always try to include:

1. **What is happening**
2. **How to prove it**
3. **The correct command/YAML**
4. **Why the fix works**

---

## 1. Big Picture

### Container

A container is a process running with isolation. It is not a full virtual machine. The container stops when its main process stops.

```bash
podman run docker.io/library/alpine echo hello
```

This exits immediately because `echo hello` finishes.

Keep it running:

```bash
podman run -d --name test docker.io/library/alpine sleep 1d
```

### Image

An image is a packaged filesystem and metadata used to start containers.

```bash
podman pull docker.io/library/nginx:1.27
podman images
```

### Orchestration

Orchestration manages containers at scale:

- Scheduling
- Scaling
- Self-healing
- Rolling updates
- Service discovery
- Storage
- Configuration
- Secrets
- Health checks
- Load balancing
- Security policy

Kubernetes and OpenShift are orchestration platforms. Podman and Docker are container engines. Docker Compose is a simpler single-host multi-container tool.

---

## 2. Podman Runtime Cheat Sheet

### Run foreground

```bash
podman run --rm docker.io/library/alpine echo hello
```

### Run detached

```bash
podman run -d --name web docker.io/library/nginx
```

### List containers

```bash
podman ps
podman ps -a
```

### Logs

```bash
podman logs web
podman logs -f web
```

### Exec into container

```bash
podman exec -it web sh
```

### Stop/remove

```bash
podman stop web
podman rm web
podman rm -f web
```

### Port publishing

Format:

```text
-p HOST_PORT:CONTAINER_PORT
```

Example:

```bash
podman run -d --name web -p 8080:80 docker.io/library/nginx
curl http://localhost:8080
```

Meaning:

```text
Host port 8080 -> container port 80
```

### Environment variables

```bash
podman run -d --name db \
  -e MYSQL_ROOT_PASSWORD='StrongPassword123!' \
  docker.io/library/mysql:8
```

### Bind mount with SELinux label

```bash
mkdir -p html
echo "hello" > html/index.html

podman run -d --name web \
  -p 8080:80 \
  -v "$(pwd)/html:/usr/share/nginx/html:Z" \
  docker.io/library/nginx
```

SELinux suffixes:

- `:Z` private relabel for one container.
- `:z` shared relabel for multiple containers.

### User-defined network

```bash
podman network create labnet

podman run -d --name a --network labnet docker.io/library/alpine sleep 1d
podman run -d --name b --network labnet docker.io/library/alpine sleep 1d

podman exec a ping -c 3 b
```

---

## 3. Common Podman Mistakes

### 3.1 Reversed port mapping

Wrong:

```bash
podman run -d --name web -p 80:8080 docker.io/library/nginx
```

Nginx listens on container port `80`, not `8080`.

Correct:

```bash
podman run -d --name web -p 8080:80 docker.io/library/nginx
```

Explanation:

```text
-p hostPort:containerPort
```

---

### 3.2 MySQL environment variable has no value

Wrong:

```bash
podman run -d --name db -e MYSQL_ROOT_PASSWORD docker.io/library/mysql
```

Correct:

```bash
podman run -d --name db \
  -e MYSQL_ROOT_PASSWORD='StrongPassword123!' \
  docker.io/library/mysql:8
```

Diagnose:

```bash
podman logs db
```

---

### 3.3 `--network host` makes `-p` useless

Wrong:

```bash
podman run -d --name app --network host -p 8080:80 docker.io/library/nginx
```

Correct normal mode:

```bash
podman run -d --name app -p 8080:80 docker.io/library/nginx
```

Correct host mode:

```bash
podman run -d --name app --network host docker.io/library/nginx
```

Explanation: host networking shares the host network namespace, so port publishing is ignored/effectively unnecessary.

---

### 3.4 BusyBox exits immediately

Wrong:

```bash
podman run -d --name c1 docker.io/library/busybox
```

Correct:

```bash
podman run -d --name c1 docker.io/library/busybox sleep 1d
```

Reason: BusyBox has no long-running default foreground service.

---

### 3.5 `--rm` plus detached short job

Problem:

```bash
podman run --rm -d --name job docker.io/library/alpine echo hello
podman logs job
```

The job finishes and `--rm` removes the container before logs can be inspected.

Better:

```bash
podman run --rm docker.io/library/alpine echo hello
```

or:

```bash
podman run -d --name job docker.io/library/alpine echo hello
podman logs job
podman rm job
```

---

### 3.6 Memory limit too low

Wrong:

```bash
podman run -d --memory 8m docker.io/library/mysql
```

Better:

```bash
podman run -d --name mysqltest \
  -e MYSQL_ROOT_PASSWORD='StrongPassword123!' \
  --memory 512m \
  -p 3306:3306 \
  docker.io/library/mysql:8
```

Reason: MySQL needs much more than 8 MiB and listens on `3306`, not HTTP port `80`.

---

### 3.7 Container name resolution fails

Wrong:

```bash
podman run -d --name a docker.io/library/alpine sleep 1d
podman run -d --name b docker.io/library/alpine sleep 1d
podman exec a ping b
```

Correct:

```bash
podman network create labnet
podman run -d --name a --network labnet docker.io/library/alpine sleep 1d
podman run -d --name b --network labnet docker.io/library/alpine sleep 1d
podman exec a ping -c 3 b
```

Reason: use a user-defined network for container DNS/name resolution.

---

### 3.8 SELinux bind mount denial

Wrong:

```bash
podman run -d --name web -v ./html:/usr/share/nginx/html docker.io/library/nginx
```

Correct:

```bash
podman run -d --name web \
  -p 8080:80 \
  -v "$(pwd)/html:/usr/share/nginx/html:Z" \
  docker.io/library/nginx
```

Reason: SELinux may block access unless the directory is relabeled with `:Z` or `:z`.

---

# 4. Containerfile / Dockerfile Mega Notes

## 4.1 Best practices

Good Containerfiles usually:

- Use specific image tags.
- Run package update and install in the same layer.
- Clean package manager cache in the same layer.
- Use exec-form `CMD`.
- Copy dependency manifests before source code for better cache usage.
- Avoid root where possible.
- Use multi-stage builds for compiled apps.
- Do not bake secrets into images.
- Match the actual app port and `EXPOSE`.
- Keep runtime images small.

---

## 4.2 Debian install mistake

Wrong:

```Dockerfile
FROM debian:12
RUN apt-get install -y nginx
```

Correct:

```Dockerfile
FROM debian:12

RUN apt-get update \
    && apt-get install -y --no-install-recommends nginx \
    && rm -rf /var/lib/apt/lists/*
```

Explanation: `apt-get update` downloads package indexes. Cleaning in the same layer reduces image size.

---

## 4.3 Shell form vs exec form

Wrong:

```Dockerfile
FROM python:3.11
COPY app.py /app/app.py
WORKDIR /app
CMD python app.py
```

Correct:

```Dockerfile
FROM python:3.11

WORKDIR /app
COPY app.py .

CMD ["python", "app.py"]
```

Explanation: exec form lets the app receive signals more directly as PID 1.

---

## 4.4 Node layer caching

Wrong:

```Dockerfile
FROM node:20
COPY . /app
WORKDIR /app
RUN npm install
```

Correct:

```Dockerfile
FROM node:20

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

CMD ["npm", "start"]
```

Explanation: source changes should not always trigger a full dependency reinstall.

---

## 4.5 PATH mistake

Wrong:

```Dockerfile
FROM alpine:3.20
ENV PATH=/app/bin
RUN apk add --no-cache curl
```

Correct:

```Dockerfile
FROM alpine:3.20

ENV PATH="/app/bin:${PATH}"

RUN apk add --no-cache curl
```

Explanation: replacing `PATH` removes directories such as `/bin` and `/usr/bin`.

---

## 4.6 EXPOSE mismatch

Wrong:

```Dockerfile
FROM ubuntu:24.04
EXPOSE 8080
CMD ["python3", "-m", "http.server", "3000"]
```

Correct option A:

```Dockerfile
FROM ubuntu:24.04

RUN apt-get update \
    && apt-get install -y --no-install-recommends python3 \
    && rm -rf /var/lib/apt/lists/*

EXPOSE 8080
CMD ["python3", "-m", "http.server", "8080"]
```

Correct option B:

```Dockerfile
FROM ubuntu:24.04

RUN apt-get update \
    && apt-get install -y --no-install-recommends python3 \
    && rm -rf /var/lib/apt/lists/*

EXPOSE 3000
CMD ["python3", "-m", "http.server", "3000"]
```

Run if app listens on `3000`:

```bash
podman run -p 8080:3000 image
```

Important: `EXPOSE` is documentation metadata. It does not publish the port by itself.

---

## 4.7 Image too large

Wrong:

```Dockerfile
FROM debian:12
RUN apt-get update && apt-get install -y build-essential
```

Better:

```Dockerfile
FROM debian:12

RUN apt-get update \
    && apt-get install -y --no-install-recommends build-essential \
    && rm -rf /var/lib/apt/lists/*
```

Best for compiled apps: multi-stage build.

---

## 4.8 Non-root user ordering

Wrong:

```Dockerfile
FROM python:3.11
USER appuser
COPY requirements.txt /app/requirements.txt
RUN pip install -r /app/requirements.txt
```

Correct:

```Dockerfile
FROM python:3.11

RUN useradd --create-home --shell /bin/bash appuser

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=appuser:appuser . .

USER appuser

CMD ["python", "app.py"]
```

Explanation: create the user, install dependencies, copy app files with correct ownership, then switch to non-root.

---

## 4.9 Go multi-stage build

Bad large image:

```Dockerfile
FROM golang:1.22
COPY . /src
WORKDIR /src
RUN go build -o app .
CMD ["/src/app"]
```

Better:

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

Static minimal version:

```Dockerfile
FROM scratch

COPY --from=builder /out/app /app

CMD ["/app"]
```

---

# 5. Kubernetes Core Objects

## 5.1 Object map

| Object | Purpose |
|---|---|
| Pod | Smallest deployable unit; one or more containers |
| Deployment | Manages stateless replicated Pods |
| ReplicaSet | Keeps desired number of Pods |
| StatefulSet | Stable Pod names and storage |
| DaemonSet | One Pod per eligible node |
| Job | Run task to completion |
| CronJob | Run Jobs on schedule |
| Service | Stable network endpoint for Pods |
| ConfigMap | Non-secret config |
| Secret | Sensitive config |
| PVC | Request for persistent storage |
| StorageClass | Storage provisioning class |
| Ingress | HTTP/HTTPS external routing |
| Namespace | Logical isolation boundary |

## 5.2 Basic kubectl

```bash
kubectl get pods
kubectl get deploy
kubectl get rs
kubectl get svc
kubectl get events --sort-by=.lastTimestamp
kubectl describe pod <pod>
kubectl logs <pod>
kubectl apply -f file.yaml
kubectl delete -f file.yaml
kubectl apply --dry-run=server --validate=true -f file.yaml
```

## 5.3 Common apiVersion/kind pairs

| Kind | apiVersion |
|---|---|
| Pod | v1 |
| Service | v1 |
| ConfigMap | v1 |
| Secret | v1 |
| PersistentVolumeClaim | v1 |
| Deployment | apps/v1 |
| StatefulSet | apps/v1 |
| DaemonSet | apps/v1 |
| Job | batch/v1 |
| CronJob | batch/v1 |
| Ingress | networking.k8s.io/v1 |

Wrong:

```yaml
apiVersion: apps/v1
kind: Pod
```

Correct:

```yaml
apiVersion: v1
kind: Pod
```

---

# 6. Deployments, ReplicaSets, Scaling, Rollouts

## 6.1 Imperative Deployment

```bash
kubectl create deployment web \
  --image=nginx:1.25 \
  --replicas=3 \
  --port=80
```

Export YAML:

```bash
kubectl get deployment web -o yaml > web-generated.yaml
```

Generate without creating:

```bash
kubectl create deployment web \
  --image=nginx:1.25 \
  --replicas=3 \
  --port=80 \
  --dry-run=client -o yaml > web.yaml
```

## 6.2 Deployment manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
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
          image: nginx:1.25
          ports:
            - containerPort: 80
```

Apply:

```bash
kubectl apply -f web.yaml
```

## 6.3 Scale

With command:

```bash
kubectl scale deployment/web --replicas=5
```

With manifest:

```yaml
spec:
  replicas: 5
```

Verify:

```bash
kubectl get deploy web
kubectl get rs -l app=web -o wide
kubectl get pods -l app=web
```

Scaling replica count alone usually does not create a new ReplicaSet. Changing the Pod template does.

## 6.4 Rolling update

```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
```

Show ReplicaSets:

```bash
kubectl get rs -l app=web -o wide
```

## 6.5 Rollout history and rollback

```bash
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
kubectl rollout undo deployment/web --to-revision=1
```

Populate change cause:

```bash
kubectl annotate deployment/web \
  kubernetes.io/change-cause="Upgrade nginx from 1.25 to 1.27" \
  --overwrite
```

## 6.6 revisionHistoryLimit

```yaml
spec:
  revisionHistoryLimit: 3
```

Means Kubernetes keeps up to 3 old ReplicaSets for rollback.

Patch:

```bash
kubectl patch deployment web --type=merge -p '{"spec":{"revisionHistoryLimit":3}}'
```

## 6.7 Selector relationship

```text
Deployment selector -> ReplicaSet selector -> Pod labels
```

Valid:

```yaml
selector:
  matchLabels:
    app: web
template:
  metadata:
    labels:
      app: web
```

Invalid:

```yaml
selector:
  matchLabels:
    app: web
template:
  metadata:
    labels:
      app: api
```

Expected error:

```text
selector does not match template labels
```

---

# 7. Deployment Update Strategies

## 7.1 RollingUpdate

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

Meaning:

- `maxSurge: 1`: one extra Pod above desired replicas is allowed.
- `maxUnavailable: 0`: no available replicas should be intentionally taken down.
- Better availability, but needs extra capacity.

Patch:

```bash
kubectl patch deployment web --type=merge -p '{
  "spec": {
    "strategy": {
      "type": "RollingUpdate",
      "rollingUpdate": {
        "maxSurge": 1,
        "maxUnavailable": 0
      }
    }
  }
}'
```

## 7.2 Recreate

```yaml
spec:
  strategy:
    type: Recreate
```

Meaning: stop old Pods first, then create new Pods.

Use when old and new versions cannot run at the same time, for example a non-backward-compatible database schema migration.

## 7.3 Bad image rollout

```bash
kubectl set image deployment/web nginx=nginx:no-such-tag
kubectl rollout status deployment/web --timeout=60s
```

Diagnose:

```bash
kubectl describe pod <new-pod>
kubectl get rs -l app=web -o wide
```

Expected:

```text
ErrImagePull
ImagePullBackOff
```

Why old Pods keep serving: rolling updates keep old healthy Pods until new Pods are available.

Recover:

```bash
kubectl rollout undo deployment/web
```

---

# 8. Pods, Sidecars, Init Containers, Workload Types

## 8.1 Bare Pod vs Deployment-managed Pod

Bare Pod:

```bash
kubectl run sleeper \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sh -c "sleep 3600"
```

Delete:

```bash
kubectl delete pod sleeper
```

A bare Pod does not return.

Deployment-managed Pod:

```bash
POD=$(kubectl get pod -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod "$POD"
kubectl get pods -l app=web
```

The ReplicaSet creates a replacement.

## 8.2 Multi-container Pod sharing emptyDir

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-emptydir
spec:
  restartPolicy: Never
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "echo hello > /data/message.txt; sleep 3600"]
      volumeMounts:
        - name: shared
          mountPath: /data
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "while [ ! -f /data/message.txt ]; do sleep 1; done; cat /data/message.txt; sleep 3600"]
      volumeMounts:
        - name: shared
          mountPath: /data
  volumes:
    - name: shared
      emptyDir: {}
```

Verify:

```bash
kubectl logs shared-emptydir -c reader
kubectl exec shared-emptydir -c reader -- cat /data/message.txt
```

## 8.3 Sidecar

A sidecar is a second container in the same Pod that supports the main container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:
  containers:
    - name: web
      image: nginx:1.27
      volumeMounts:
        - name: shared
          mountPath: /shared
    - name: sidecar
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date >> /shared/timestamps.txt; sleep 5; done"]
      volumeMounts:
        - name: shared
          mountPath: /shared
  volumes:
    - name: shared
      emptyDir: {}
```

Same-Pod containers share:

- Pod IP
- localhost network
- mounted volumes

## 8.4 Init container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
    - name: init-writer
      image: busybox:1.36
      command: ["sh", "-c", "echo 'created by init' > /work/index.html"]
      volumeMounts:
        - name: work
          mountPath: /work
  containers:
    - name: main
      image: busybox:1.36
      command: ["sh", "-c", "cat /work/index.html; sleep 3600"]
      volumeMounts:
        - name: work
          mountPath: /work
  volumes:
    - name: work
      emptyDir: {}
```

Init containers run before app containers.

## 8.5 StatefulSet with headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
    - name: redis
      port: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7
          ports:
            - name: redis
              containerPort: 6379
```

Stable names:

```text
redis-0
redis-1
redis-2
```

DNS:

```text
redis-0.redis.default.svc.cluster.local
redis-1.redis.default.svc.cluster.local
redis-2.redis.default.svc.cluster.local
```

## 8.6 StatefulSet ordering

Scale down:

```bash
kubectl scale statefulset redis --replicas=0
kubectl get pods -l app=redis -w
```

Expected termination order:

```text
redis-2 -> redis-1 -> redis-0
```

Scale up:

```bash
kubectl scale statefulset redis --replicas=3
```

Expected creation order:

```text
redis-0 -> redis-1 -> redis-2
```

## 8.7 DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: busybox-agent
spec:
  selector:
    matchLabels:
      app: busybox-agent
  template:
    metadata:
      labels:
        app: busybox-agent
    spec:
      containers:
        - name: agent
          image: busybox:1.36
          command: ["sh", "-c", "while true; do echo agent on $(hostname); sleep 30; done"]
```

A DaemonSet runs one Pod on every eligible node.

## 8.8 Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: calc-once
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: calc
          image: perl:5.38-alpine
          command: ["perl", "-e", "$sum=0; $sum += $_ for (1..100); print qq(sum=$sum\n);"]
```

Commands:

```bash
kubectl apply -f calc-job.yaml
kubectl get jobs
kubectl logs job/calc-once
```

## 8.9 CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: date-every-minute
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: date
              image: busybox:1.36
              command: ["sh", "-c", "date"]
```

Suspend:

```bash
kubectl patch cronjob date-every-minute -p '{"spec":{"suspend":true}}'
```

Manual run:

```bash
kubectl create job date-manual --from=cronjob/date-every-minute
kubectl logs job/date-manual
```

---

# 9. ConfigMaps, Secrets, Volumes, Storage

## 9.1 ConfigMap

```bash
kubectl create configmap app-config \
  --from-literal=message="hello from configmap" \
  --from-literal=color="blue"
```

Mount as files:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-volume-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: config
          mountPath: /etc/config
          readOnly: true
  volumes:
    - name: config
      configMap:
        name: app-config
```

Verify:

```bash
kubectl exec config-volume-pod -- ls -l /etc/config
kubectl exec config-volume-pod -- cat /etc/config/message
```

## 9.2 ConfigMap single key with subPath

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-subpath-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: config
          mountPath: /app/message.txt
          subPath: message
          readOnly: true
  volumes:
    - name: config
      configMap:
        name: app-config
        items:
          - key: message
            path: message
```

Use when you need one file at a specific path without replacing the whole directory.

Caveat: `subPath` mounts do not receive live updates like normal projected ConfigMap/Secret volumes.

## 9.3 Secret from literals

```bash
kubectl create secret generic db-creds \
  --from-literal=username=dbuser \
  --from-literal=password='p@ssw0rd'
```

Show encoded data:

```bash
kubectl get secret db-creds -o yaml
```

Decode:

```bash
echo 'ZGJ1c2Vy' | base64 -d
```

Remember: base64 is encoding, not encryption.

## 9.4 Mount Secret as volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: secret
          mountPath: /etc/secret
          readOnly: true
  volumes:
    - name: secret
      secret:
        secretName: db-creds
        defaultMode: 0400
```

Verify:

```bash
kubectl exec secret-volume-pod -- ls -l /etc/secret
kubectl exec secret-volume-pod -- cat /etc/secret/username
```

## 9.5 Secret as env vars

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo user=$username; sleep 3600"]
      envFrom:
        - secretRef:
            name: db-creds
```

## 9.6 Docker registry Secret

```bash
kubectl create secret docker-registry dockerhub-creds \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<username> \
  --docker-password=<password-or-token> \
  --docker-email=<email>
```

Use in Pod template:

```yaml
imagePullSecrets:
  - name: dockerhub-creds
```

## 9.7 TLS Secret

```bash
kubectl create secret tls web-tls \
  --cert=tls.crt \
  --key=tls.key
```

Often consumed by Ingress for HTTPS.

## 9.8 emptyDir

```yaml
volumes:
  - name: cache
    emptyDir: {}
```

`emptyDir` is created for the Pod and deleted when the Pod is deleted.

## 9.9 emptyDir sizeLimit

```yaml
volumes:
  - name: cache
    emptyDir:
      medium: Memory
      sizeLimit: 10Mi
```

Exceeding it may cause write failures or eviction under storage pressure.

## 9.10 readOnly volume mount

```yaml
volumeMounts:
  - name: data
    mountPath: /data
    readOnly: true
```

Test:

```bash
kubectl exec readonly-volume -- sh -c 'echo test > /data/file.txt'
```

Expected:

```text
Read-only file system
```

## 9.11 PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Use in Pod:

```yaml
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-data
```

List storage:

```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc
```

## 9.12 StatefulSet PVCs

A StatefulSet with `volumeClaimTemplates` creates one PVC per replica.

Example names:

```text
data-redis-0
data-redis-1
data-redis-2
```

Deleting the StatefulSet does not automatically delete PVCs. This protects data.

---

# 10. Services, Networking, DNS

## 10.1 ClusterIP

```bash
kubectl expose deployment web \
  --name=web-clusterip \
  --port=80 \
  --target-port=80 \
  --type=ClusterIP
```

YAML:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-clusterip
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

DNS:

```text
web-clusterip.default.svc.cluster.local
```

Test:

```bash
kubectl run tmp --rm -it --restart=Never --image=busybox:1.36 -- sh
```

Inside:

```sh
nslookup web-clusterip.default.svc.cluster.local
wget -qO- http://web-clusterip
```

## 10.2 NodePort

```bash
kubectl expose deployment web \
  --name=web-nodeport \
  --port=80 \
  --target-port=80 \
  --type=NodePort
```

Minikube:

```bash
minikube service web-nodeport --url
```

Direct:

```bash
NODE_PORT=$(kubectl get svc web-nodeport -o jsonpath='{.spec.ports[0].nodePort}')
curl "http://$(minikube ip):$NODE_PORT"
```

## 10.3 LoadBalancer

```bash
kubectl expose deployment web \
  --name=web-lb \
  --port=80 \
  --target-port=80 \
  --type=LoadBalancer
```

On Minikube, `EXTERNAL-IP` may stay `<pending>` because there is no cloud load balancer.

Use:

```bash
minikube tunnel
```

## 10.4 Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
    - name: redis
      port: 6379
```

Used with StatefulSets for stable per-Pod DNS.

## 10.5 Endpoints and EndpointSlices

```bash
kubectl get endpoints web-clusterip
kubectl get endpointslice -l kubernetes.io/service-name=web-clusterip
```

If endpoints are empty:

- Service selector does not match Pod labels.
- Pods are not Ready.
- Pods are in another namespace.
- Pods do not exist.

## 10.6 Cross-namespace DNS

From namespace `other`, full name works:

```text
web-clusterip.default.svc.cluster.local
```

Short name usually fails:

```text
web-clusterip
```

because it resolves in the current namespace first.

## 10.7 port vs targetPort vs nodePort

```yaml
ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

Flow:

```text
client -> nodeIP:30080 -> Service port 80 -> Pod targetPort 8080
```

## 10.8 One Service across two Deployments

If two Deployments have Pods with:

```yaml
labels:
  app: shared-echo
```

a Service with:

```yaml
selector:
  app: shared-echo
```

will load-balance to both.

---

# 11. Probes

## 11.1 Liveness probe

Restarts container if unhealthy.

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
```

## 11.2 Readiness probe

Controls whether Pod receives Service traffic.

```yaml
readinessProbe:
  tcpSocket:
    port: 6379
  initialDelaySeconds: 5
  periodSeconds: 10
```

## 11.3 Startup probe

Protects slow-starting apps.

```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
  periodSeconds: 10
  failureThreshold: 12
```

Budget:

```text
10s * 12 = 120 seconds
```

## 11.4 No probes

If no probes are defined:

- Kubernetes does not know if app is internally healthy.
- Pod may be considered Ready as soon as container is running.
- Crashed processes still restart, but stuck/unhealthy apps may not.

---

# 12. Kubernetes Troubleshooting Playbook

## 12.1 Universal first commands

```bash
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl get events --sort-by=.lastTimestamp
```

## 12.2 Pending

```bash
kubectl describe pod <pod>
```

Common events:

| Event | Meaning | Fix |
|---|---|---|
| Insufficient cpu | Requests too high | Lower requests or add nodes |
| Insufficient memory | Memory request too high | Lower request or add nodes |
| node selector mismatch | No matching node | Change selector or label node |
| unbound PVC | Storage not bound | Create/fix PVC or StorageClass |
| taint not tolerated | Node blocks Pod | Add toleration or use another node |

## 12.3 ImagePullBackOff

```bash
kubectl describe pod <pod>
```

Causes:

- Image typo
- Bad tag
- Private registry without Secret
- Registry unavailable
- Pull limit

Fix image:

```bash
kubectl set image deployment/web nginx=nginx:1.27
```

## 12.4 CrashLoopBackOff

```bash
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl describe pod <pod>
```

Causes:

- Bad command
- Missing file
- App exception
- Permission problem
- Missing env var
- Dependency unavailable

## 12.5 OOMKilled

```bash
kubectl describe pod <pod>
kubectl get pod <pod> -o yaml | grep -A20 lastState
```

Fix:

```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"
```

or reduce app memory usage.

## 12.6 Pods never Ready

Check:

```bash
kubectl describe pod <pod>
kubectl get endpoints <service>
```

Common reason:

```text
Readiness probe failed
```

Fix path, port, delay, timeout, or application health endpoint.

## 12.7 Service returns nothing

```bash
kubectl get svc <svc> -o yaml
kubectl get endpoints <svc>
kubectl get pods --show-labels
```

If no endpoints:

- Selector mismatch
- Pods not Ready
- Wrong namespace

## 12.8 ConfigMap key missing

Error:

```text
CreateContainerConfigError
couldn't find key LOG_LEVEL in ConfigMap
```

Fix:

```bash
kubectl create configmap app-settings \
  --from-literal=LOG_LEVEL=info \
  --dry-run=client -o yaml | kubectl apply -f -
```

## 12.9 Secret wrong namespace

Secrets are namespace-scoped.

```bash
kubectl get secret db-creds -n app
kubectl get secret db-creds -n default
```

Fix by creating Secret in same namespace as Pod.

## 12.10 Stuck rollout

```bash
kubectl rollout status deployment/<name>
kubectl describe deployment <name>
kubectl get pods
kubectl describe pod <pod>
```

Condition:

```text
ProgressDeadlineExceeded
```

Fix bad image, crash, readiness probe, config, secret, resource, or PVC issue. Or roll back:

```bash
kubectl rollout undo deployment/<name>
```

## 12.11 DNS debugging

```bash
kubectl run dns-debug --rm -it --restart=Never --image=busybox:1.36 -- sh
```

Inside:

```sh
cat /etc/resolv.conf
nslookup kubernetes.default.svc.cluster.local
nslookup <service>.<namespace>.svc.cluster.local
```

## 12.12 Ephemeral debug container

For no-shell/distroless containers:

```bash
kubectl debug -it <pod> \
  --image=busybox:1.36 \
  --target=<container> \
  -- sh
```

## 12.13 Throwaway connectivity test

```bash
kubectl run tmp --rm -it --restart=Never --image=busybox:1.36 -- sh
```

Inside:

```sh
nslookup <service>
wget -qO- http://<service>:<port>
nc -vz <service> <port>
```

## 12.14 Node NotReady

```bash
kubectl get nodes -o wide
kubectl describe node <node>
```

Check:

- kubelet
- disk pressure
- memory pressure
- CNI/network plugin
- container runtime
- certificates
- node reachability

Minikube:

```bash
minikube status
minikube logs
minikube ssh
sudo journalctl -u kubelet -xe
```

## 12.15 Pod stuck Terminating

```bash
kubectl get pod <pod> -o yaml
kubectl describe pod <pod>
```

Reasons:

- finalizers
- grace period
- node unavailable
- volume detach
- app not handling SIGTERM

Force delete:

```bash
kubectl delete pod <pod> --grace-period=0 --force
```

Risk: the process may still be running on the node temporarily, which can cause duplicate writers or storage corruption.

## 12.16 JSONPath for state

```bash
kubectl get pod <pod> \
  -o jsonpath='{range .status.containerStatuses[*]}name={.name} state={.state} lastState={.lastState} restarts={.restartCount}{"\n"}{end}'
```

---

# 13. LO6 Theory Mega Summary

## 13.1 Kubernetes networking vs Docker networking

Docker networking is mostly simple and single-host oriented. It uses bridge networks and host port publishing.

Kubernetes networking is cluster-oriented. Each Pod gets an IP, Services provide stable virtual endpoints, and DNS provides stable names. A CNI plugin implements the actual network.

Conclusion:

- Docker is easier for local development and single-host apps.
- Kubernetes is better for multi-node production systems.

## 13.2 Kubernetes storage vs Podman storage

Podman volumes are simple and useful on one host.

Kubernetes storage is more flexible for stateful workloads because it supports:

- PV
- PVC
- StorageClass
- CSI
- Dynamic provisioning
- StatefulSet volume templates

Conclusion: Podman is simpler; Kubernetes is more powerful for clusters.

## 13.3 Operators vs manifests

Plain manifests describe resources.

Operators add automation using:

- CRDs
- Controllers
- Reconciliation loops

Operators are useful for databases and complex stateful systems because they can automate backups, failover, upgrades, and recovery.

Trade-off:

- Manifests are simpler.
- Operators automate more but add complexity.

## 13.4 Kubernetes vs OpenShift developer experience

Plain Kubernetes:

- Flexible
- Portable
- Requires choosing many tools
- More YAML and platform knowledge

OpenShift:

- Integrated console
- `oc` CLI
- Routes
- S2I
- ImageStreams
- BuildConfigs
- Operators
- Stronger default security

Conclusion: Kubernetes optimizes flexibility; OpenShift optimizes integrated enterprise experience.

## 13.5 Migrating OpenShift to Kubernetes

Usually easy:

- Deployments
- Services
- ConfigMaps
- Secrets
- PVCs
- Jobs
- CronJobs

Needs conversion:

| OpenShift | Kubernetes replacement |
|---|---|
| Route | Ingress or Gateway API |
| BuildConfig | CI/CD pipeline |
| ImageStream | Registry tags |
| DeploymentConfig | Deployment |
| SCC | Pod Security Admission / policy tools |
| Template | Helm or Kustomize |

Conclusion: migration is realistic but not copy-paste if OpenShift-specific features are used.

## 13.6 CI/CD agents in Kubernetes vs VMs

Kubernetes agents:

- Autoscale
- Fresh environment per job
- Good for container-native pipelines
- Risk noisy-neighbor effects
- Privileged builds can be risky

VM agents:

- Stronger isolation
- Easier persistent cache
- Simpler privileged builds
- Less elastic

Conclusion: use Kubernetes for elastic CI/CD; use VMs or separate build clusters for risky privileged builds.

## 13.7 Cloud integration

Kubernetes integrates with clouds through:

- Cloud Controller Manager
- LoadBalancer Services
- CSI storage drivers
- Cluster Autoscaler
- Managed Kubernetes services
- Cloud identity/networking integrations

Conclusion: Kubernetes is strong in cloud environments because infrastructure can be provisioned dynamically.

## 13.8 OpenShift security vs vanilla Kubernetes

Vanilla Kubernetes provides security building blocks, but the operator must configure them.

OpenShift is more opinionated and enterprise-focused:

- SCCs
- non-root defaults
- integrated OAuth
- policy controls
- Red Hat support
- compliance features

Conclusion: regulated enterprises often choose OpenShift because it provides stronger defaults, support, and governance.

## 13.9 Learning curve

Docker Compose is easiest.

Plain Kubernetes is flexible but has a steep learning curve.

OpenShift adds concepts but provides a guided developer experience.

Conclusion: OpenShift helps developers, but platform engineers still need deep Kubernetes/OpenShift knowledge.

## 13.10 Full Kubernetes vs k3s

k3s is lightweight Kubernetes for:

- edge
- labs
- small on-prem clusters
- resource-constrained systems

Full Kubernetes is better for:

- large enterprise platforms
- complex add-ons
- multi-team governance
- high scale

Conclusion: full Kubernetes is overkill for very small deployments; k3s is often better.

## 13.11 Docker Compose vs Kubernetes for small teams

Use Compose when:

- one host is enough
- few services exist
- team must ship fast
- low operational overhead matters

Use Kubernetes when:

- multi-node resilience is needed
- autoscaling is needed
- many services exist
- rolling updates and self-healing matter

Conclusion: start simple unless Kubernetes benefits clearly outweigh complexity.

## 13.12 Vendor lock-in

Kubernetes reduces orchestration lock-in, but cloud-specific storage, load balancers, identity, and annotations can still create lock-in.

OpenShift adds Red Hat-specific features such as Routes, S2I, BuildConfigs, ImageStreams, and SCCs.

Conclusion: Kubernetes is more portable; OpenShift is more integrated and supported.

## 13.13 Recommend OpenShift over Kubernetes when

Choose OpenShift when a company wants:

- vendor support
- integrated developer console
- built-in routing
- stronger security defaults
- standardized platform
- compliance support
- enterprise lifecycle

Avoid OpenShift when:

- team is small
- cost must be minimal
- portability matters most
- workload is simple

## 13.14 Kubernetes storage vs VM storage

VM storage is often manually attached, formatted, mounted, and managed.

Kubernetes storage is declarative through PVCs and dynamically provisioned through StorageClasses and CSI.

Conclusion: Kubernetes is better for self-service cloud-native storage; VMs are simpler for traditional static workloads.

## 13.15 Startup with two engineers

Recommendation: use Docker Compose, Podman, or a simple managed container service first.

Reason: Kubernetes adds cluster, networking, storage, monitoring, backup, security, and upgrade overhead.

Move to Kubernetes when scaling and resilience needs justify it.

## 13.16 Docker vs Podman

Docker:

- daemon-based
- huge ecosystem
- common developer workflow

Podman:

- daemonless
- strong rootless support
- good systemd integration
- strong Linux/SELinux integration

Security-conscious teams may prefer Podman because it avoids a long-running privileged daemon by default.

## 13.17 Self-managed vs managed Kubernetes

Self-managed:

- maximum control
- high operational burden

Managed:

- less control
- lower operational burden
- easier cloud integration

Conclusion: managed Kubernetes is usually better unless the organization needs deep infrastructure control.

## 13.18 Media streaming with spiky global traffic

Recommended architecture:

- Managed Kubernetes for APIs/workers
- CDN for video delivery
- object storage for media
- autoscaling
- multi-region deployment
- observability

Important: do not serve all video bytes directly from Pods if a CDN can do it better.

## 13.19 Outgrowing Compose or Swarm

Migrate to Kubernetes when:

- single host is a bottleneck
- manual recovery is common
- deployments are risky
- scaling is needed
- many services exist
- ecosystem standardization matters

Conclusion: migration is worth it when operational pain exceeds Kubernetes complexity.

## 13.20 Kubernetes for microservices

Choose Kubernetes for serious microservices because it provides:

- service discovery
- scaling
- rolling updates
- self-healing
- config/secrets
- health checks
- network policy
- ecosystem integrations

But for very small systems, it may be too complex.

## 13.21 Kubernetes for enterprise apps

Kubernetes is strong for enterprise apps because it has:

- scalability
- portability
- standard APIs
- large ecosystem
- community and vendor support
- declarative operations

But enterprises still need security, monitoring, logging, backup, policy, and lifecycle management.

## 13.22 OpenShift for microservices

OpenShift is strong for enterprise microservices because it combines Kubernetes with:

- integrated developer tools
- Routes
- Operators
- security defaults
- support
- standardized workflows

It may be too heavy for small/simple systems.

## 13.23 OpenShift for enterprise apps

OpenShift is a strong enterprise platform because it provides:

- Kubernetes foundation
- Red Hat support
- integrated console
- routing
- security defaults
- operators
- hybrid cloud options
- compliance-friendly tooling

Trade-off: cost, platform weight, and Red Hat-specific concepts.

---

# 14. Final Exam Quick Reference

## 14.1 Podman

```bash
podman run --rm image command
podman run -d --name name image command
podman ps
podman ps -a
podman logs name
podman exec -it name sh
podman rm -f name
podman network create net
podman run --network net ...
podman run -p 8080:80 ...
podman run -v "$(pwd)/html:/usr/share/nginx/html:Z" ...
```

## 14.2 Kubernetes essentials

```bash
kubectl apply -f file.yaml
kubectl delete -f file.yaml
kubectl get all
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl get events --sort-by=.lastTimestamp
```

## 14.3 Rollouts

```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
kubectl get rs -l app=web -o wide
```

## 14.4 Services

```bash
kubectl get svc
kubectl describe svc <svc>
kubectl get endpoints <svc>
kubectl get endpointslice -l kubernetes.io/service-name=<svc>
```

## 14.5 Storage

```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc
kubectl describe pvc <pvc>
```

## 14.6 Status meanings

| Status | Meaning |
|---|---|
| Pending | Not scheduled or waiting for resources/storage |
| ContainerCreating | Image/volume/container setup in progress |
| Running | Container process is running |
| Completed | Main process finished successfully |
| CrashLoopBackOff | Container repeatedly crashes |
| ImagePullBackOff | Image pull failed and retries are delayed |
| ErrImagePull | Immediate image pull error |
| CreateContainerConfigError | Config/Secret/env/volume issue |
| OOMKilled | Exceeded memory limit |
| Terminating | Deletion cleanup not finished |
| NotReady | Pod/node not ready |

## 14.7 Troubleshooting table

| Problem | Check | Fix |
|---|---|---|
| Wrong port | Service YAML, Pod ports | Fix `port`, `targetPort`, `containerPort` |
| Service no traffic | `kubectl get endpoints` | Fix selector or readiness |
| ImagePullBackOff | `kubectl describe pod` | Fix image/tag/pull secret |
| CrashLoopBackOff | `kubectl logs --previous` | Fix command/config/app |
| Pending | `kubectl describe pod` | Fix resources/PVC/nodeSelector |
| OOMKilled | Last state | Raise memory or fix app |
| Probe failure | Pod events | Fix probe path/port/delay |
| Secret missing | `kubectl get secret -n ns` | Create in same namespace |
| Config key missing | Pod events | Add key or fix reference |
| Bad YAML | dry-run server validation | Fix indentation/fields |

## 14.8 Good answer format

For practical tasks:

1. Identify the mistake.
2. Show the diagnostic command.
3. Show the fixed command/YAML.
4. Explain why it works.

For theory tasks:

1. Compare both options.
2. Give strengths and weaknesses.
3. Give a realistic scenario.
4. Defend a final recommendation.

---

# 15. References

Official documentation useful for this exam:

- Kubernetes Deployments: <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
- Kubernetes rolling updates: <https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/>
- Kubernetes Services: <https://kubernetes.io/docs/concepts/services-networking/service/>
- Kubernetes DNS: <https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/>
- Kubernetes networking: <https://kubernetes.io/docs/concepts/services-networking/>
- Kubernetes NetworkPolicy: <https://kubernetes.io/docs/concepts/services-networking/network-policies/>
- Kubernetes PersistentVolumes: <https://kubernetes.io/docs/concepts/storage/persistent-volumes/>
- Kubernetes StorageClasses: <https://kubernetes.io/docs/concepts/storage/storage-classes/>
- Kubernetes volumes: <https://kubernetes.io/docs/concepts/storage/volumes/>
- Kubernetes probes: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/>
- Kubernetes debug Pods: <https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/>
- Kubernetes ephemeral containers: <https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/>
- Kubernetes operator pattern: <https://kubernetes.io/docs/concepts/extend-kubernetes/operator/>
- Kubernetes custom resources: <https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/>
- Kubernetes cloud controller manager: <https://kubernetes.io/docs/concepts/architecture/cloud-controller/>
- Podman run manual: <https://docs.podman.io/en/latest/markdown/podman-run.1.html>
- Podman volume mount options: <https://docs.podman.io/en/latest/markdown/options/volume.html>
- Docker build best practices: <https://docs.docker.com/build/building/best-practices/>
- Docker build cache: <https://docs.docker.com/build/cache/>
- k3s documentation: <https://docs.k3s.io/>
- OpenShift Routes: <https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/ingress_and_load_balancing/routes>
- OpenShift Security Context Constraints: <https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/authentication_and_authorization/managing-pod-security-policies>
