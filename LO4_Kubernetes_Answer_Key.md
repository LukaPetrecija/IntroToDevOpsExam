# LO4 — Accelerated delivery of multilayer applications using containers
### Detailed answer key (commands + explanations)

*Assumptions: a running cluster (the questions mention **minikube**), the `default` namespace unless stated, and `kubectl` configured. A handy trick used throughout: build manifests with `--dry-run=client -o yaml` to generate clean YAML you can edit and `kubectl apply -f`.*

**Two facts that come up constantly:**
- When you run `kubectl create deployment web --image=nginx:1.25`, the **container is named after the image** → `nginx`. You need that name for `kubectl set image`.
- A **Deployment owns a ReplicaSet, which owns Pods**. The Deployment changes the *pod template*; the ReplicaSet just keeps *N copies*. A new ReplicaSet is created only when the **template changes** (e.g. image), **not** when you only change the replica count.

---

## Deployments, rollouts & rollbacks

### 1. Create `web` (nginx:1.25, 3 replicas) imperatively, then export YAML
```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl get deployment web -o yaml > web.yaml
# cleaner, schema-only export (no status/managedFields):
kubectl create deployment web --image=nginx:1.25 --replicas=3 --dry-run=client -o yaml > web.yaml
```
**Explanation:** `kubectl create deployment` is the *imperative* way — one command builds the object directly on the server. The `--replicas` flag sets the desired count. Exporting the live object with `-o yaml` captures the running state but includes server-managed fields (`status`, `managedFields`, `creationTimestamp`). `--dry-run=client` builds the object *locally without sending it*, so `-o yaml` gives a clean manifest you can commit and re-apply declaratively. This is the standard "imperative → declarative" workflow.

### 2. Authenticated DockerHub pulls to avoid the pull limit — which resource?
A **docker-registry Secret** of type `kubernetes.io/dockerconfigjson`, referenced through **`imagePullSecrets`**.
```bash
kubectl create secret docker-registry dockerhub \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=USER --docker-password=PASS --docker-email=you@example.com
```
Reference it in the pod spec:
```yaml
spec:
  imagePullSecrets:
    - name: dockerhub
```
**Explanation:** anonymous DockerHub pulls are rate-limited per IP; authenticating raises the limit. The credentials live in this special Secret (its single key `.dockerconfigjson` is the Docker config). The kubelet uses it when pulling. You can attach it per-pod via `imagePullSecrets`, or once to a **ServiceAccount** so every pod using that account inherits it.

### 3. Scale `web` 3 → 5 two ways, and show the ReplicaSet
```bash
# Way 1 — imperative
kubectl scale deployment web --replicas=5

# Way 2 — declarative: set replicas: 5 in web.yaml, then
kubectl apply -f web.yaml
# (or interactively: kubectl edit deployment web)

kubectl get rs -l app=web      # one RS, DESIRED/CURRENT now 5
```
**Explanation:** scaling changes only the replica *count*, not the pod template, so **no new ReplicaSet is created** — the *same* RS goes from 3 to 5. `kubectl get rs` shows DESIRED=5. This is the key difference from a rollout (item 4), which *does* spawn a new RS.

### 4. Rolling update nginx:1.25 → nginx:1.27 and watch it
```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
```
**Explanation:** changing the image changes the pod template, so the Deployment creates a **new ReplicaSet** and gradually shifts pods from old to new (RollingUpdate). `kubectl rollout status` blocks and prints progress until all new pods are Ready and old ones are gone. `nginx=` targets the container named `nginx`.

### 5. Rollout history, rollback, and CHANGE-CAUSE
```bash
kubectl rollout history deployment/web                 # list revisions
kubectl rollout history deployment/web --revision=2    # detail of one revision
kubectl rollout undo deployment/web                    # roll back to previous
kubectl rollout undo deployment/web --to-revision=1    # roll back to a specific revision
```
Populate CHANGE-CAUSE:
```bash
kubectl annotate deployment/web kubernetes.io/change-cause="upgrade nginx to 1.27" --overwrite
```
**Explanation:** each template change is a numbered **revision** backed by an old ReplicaSet. `rollout undo` re-applies a previous revision's template. The **CHANGE-CAUSE** column comes from the annotation `kubernetes.io/change-cause`; it's blank unless you set it (the old `--record` flag is deprecated). It documents *why* each revision happened, which is invaluable when deciding what to roll back to.

### 6. Strategy RollingUpdate with maxSurge: 1, maxUnavailable: 0
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```
**Explanation:** `maxUnavailable: 0` means the number of **available** pods may never drop below the desired count *during* the update — so capacity is fully preserved (zero downtime). `maxSurge: 1` allows **one extra** pod above desired temporarily. The controller therefore creates 1 new pod, waits until it is Ready, then removes 1 old pod, and repeats — strictly "create-then-delete," one at a time. Safest for availability; slightly slower and needs room for the extra pod.

### 7. Switch strategy to Recreate — when is it required?
```yaml
spec:
  strategy:
    type: Recreate
```
**Explanation:** `Recreate` **terminates all old pods first, then creates the new ones** → a brief outage. It's required when the two versions **cannot run at the same time**, for example: a database **schema migration** where old and new code can't share the schema; an app backed by a **ReadWriteOnce** volume that only one pod can mount at a time; or any "single-writer" resource. RollingUpdate would briefly run both versions, which here would corrupt data or fail to mount — so you accept downtime instead.

### 8. Add CPU/memory requests and limits, verify in the running pod
```yaml
resources:
  requests: { cpu: "250m", memory: "128Mi" }
  limits:   { cpu: "500m", memory: "256Mi" }
```
```bash
kubectl get pod <pod> -o jsonpath='{.spec.containers[0].resources}{"\n"}'
kubectl describe pod <pod>   # shows Requests/Limits
```
**Explanation:** **requests** are what the scheduler guarantees and uses to place the pod (it must fit on a node with that much free). **limits** are hard caps: exceeding the **memory** limit → the container is **OOMKilled**; exceeding the **CPU** limit → it is **throttled** (not killed). `250m` = 0.25 of a CPU core; `128Mi` = 128 mebibytes.

### 9. revisionHistoryLimit: 3 — effect
```yaml
spec:
  revisionHistoryLimit: 3
```
**Explanation:** the Deployment keeps the old **ReplicaSets** that back previous revisions so you can roll back to them. `revisionHistoryLimit: 3` keeps only the **3 most recent** old RSs; older ones are garbage-collected. Consequence: you can only `rollout undo` to a revision whose RS still exists — beyond 3 generations back, rollback is no longer possible. Default is 10. Setting it low keeps the namespace tidy but shortens your rollback window.

### 10. Set image to a non-existent tag — why does the rollout get stuck but old pods keep serving?
```bash
kubectl set image deployment/web nginx=nginx:doesnotexist
kubectl rollout status deployment/web      # hangs
kubectl get pods                           # new pod: ImagePullBackOff / ErrImagePull
kubectl rollout undo deployment/web        # recover
```
**Explanation:** the new ReplicaSet's pod can't pull the image, so it **never becomes Ready**. Under RollingUpdate with `maxUnavailable` (default 25%), the controller **will not delete an old pod until a new one is Ready**. Since no new pod is ever Ready, the old, healthy pods are never removed and **keep serving traffic** — the rollout simply stalls. This is a deliberate safety property: a bad image can't take your service down. You recover with `rollout undo`.

### 11. List only one Deployment's pods; explain the selector chain
```bash
kubectl get pods -l app=web
kubectl get pods -l app=web -L pod-template-hash   # also show the RS hash
```
**Explanation:** the **Deployment**'s `selector.matchLabels` (e.g. `app=web`) chooses which **ReplicaSets** it owns. Each **ReplicaSet** selects **Pods** using that label **plus** an automatically added `pod-template-hash` label that is unique per template generation. That hash is what lets two ReplicaSets (old and new during a rollout) coexist while each owns only its own pods. So: Deployment →(label selector)→ ReplicaSet →(label + pod-template-hash)→ Pods.

### 12. `kubectl expose` — what object, and how is the selector derived?
```bash
kubectl expose deployment web --port=80 --target-port=80
kubectl get svc web -o yaml   # see the selector
```
**Explanation:** this creates a **Service** (default `type: ClusterIP`). Its **selector is copied from the Deployment's `spec.selector.matchLabels`** (e.g. `app=web`), so the Service automatically targets the pods that the Deployment manages. `--port` is the Service's port; `--target-port` is the container port it forwards to.

### 13. Add a sidecar container; how do the two containers share network and volumes?
```yaml
spec:
  template:
    spec:
      volumes:
        - name: shared
          emptyDir: {}
      containers:
        - name: web
          image: nginx:1.25
          volumeMounts: [{ name: shared, mountPath: /usr/share/nginx/html }]
        - name: sidecar
          image: busybox
          command: ["sh","-c","while true; do date >> /data/index.html; sleep 5; done"]
          volumeMounts: [{ name: shared, mountPath: /data }]
```
**Explanation:** all containers in a **pod** share the same **network namespace** — they have the *same IP*, reach each other over **`localhost`**, and therefore must use **different ports**. They can also share storage by mounting the **same volume** (here an `emptyDir`) at any path each. Because they're in one pod they're always **co-scheduled on the same node** and live/die together. This is the classic sidecar pattern (e.g. a logging or proxy helper).

### 14. Deployment for httpd:2.4 (2 replicas, named port, nodeSelector); why Pending if nothing matches?
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: web2 }
spec:
  replicas: 2
  selector: { matchLabels: { app: web2 } }
  template:
    metadata: { labels: { app: web2 } }
    spec:
      nodeSelector: { disktype: ssd }
      containers:
        - name: httpd
          image: httpd:2.4
          ports: [{ name: http, containerPort: 80 }]
```
**Explanation:** `nodeSelector: {disktype: ssd}` restricts scheduling to nodes **labelled** `disktype=ssd`. If **no node carries that label**, the scheduler can find no valid node, so the pods sit in **Pending** (a `kubectl describe pod` event reads "0/N nodes are available: … didn't match node selector"). Fix by labelling a node (`kubectl label node <n> disktype=ssd`) or removing the selector. A **named port** lets a Service reference it by name (`targetPort: http`) instead of a number.

### 15. Bare Pod (busybox sleep) vs Deployment-managed pod on delete
```bash
kubectl run sleeper --image=busybox --command -- sleep 3600
kubectl delete pod sleeper
```
**Explanation:** a **bare Pod** has **no controller** watching it, so when you delete it (or it crashes/the node dies) it is **gone for good** — nothing recreates it. A **Deployment-managed pod** is owned by a ReplicaSet that maintains a desired count; deleting it makes the RS notice the shortfall and **create a replacement** within seconds. This is exactly why you almost never run bare Pods in production — controllers give you self-healing.

---

## StatefulSets, DaemonSets, Jobs, CronJobs

### 16. StatefulSet redis:7 (3 replicas) + headless Service; stable names & DNS
```yaml
apiVersion: v1
kind: Service
metadata: { name: redis }
spec:
  clusterIP: None            # headless
  selector: { app: redis }
  ports: [{ port: 6379 }]
---
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: redis }
spec:
  serviceName: redis         # must match the headless Service
  replicas: 3
  selector: { matchLabels: { app: redis } }
  template:
    metadata: { labels: { app: redis } }
    spec:
      containers: [{ name: redis, image: redis:7, ports: [{ containerPort: 6379 }] }]
```
```bash
kubectl get pods -l app=redis    # redis-0, redis-1, redis-2
```
**Explanation:** a StatefulSet gives pods **stable, ordinal names** `redis-0/1/2` (not random hashes). Paired with a **headless Service** (`clusterIP: None`), each pod gets a **stable DNS A record**: `redis-0.redis.<namespace>.svc.cluster.local`. This lets clients address a *specific* pod — essential for clustered stateful systems (primaries/replicas).

### 17. DaemonSet (busybox agent); why exactly one pod per node?
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata: { name: agent }
spec:
  selector: { matchLabels: { app: agent } }
  template:
    metadata: { labels: { app: agent } }
    spec:
      containers: [{ name: agent, image: busybox, command: ["sh","-c","while true; do sleep 3600; done"] }]
```
**Explanation:** a **DaemonSet** schedules **one pod on every (matching) node** and automatically adds a pod when a **new node** joins and removes it when a node leaves. There's no `replicas` field — the count equals the number of eligible nodes. Used for **node-level agents**: log collectors, monitoring agents, network/storage plugins.

### 18. Job (perl) that computes once; show COMPLETIONS and read the result
```bash
kubectl create job pi --image=perl:5.34 -- perl -Mbignum=bpi -wle 'print bpi(100)'
kubectl get job pi          # COMPLETIONS 1/1 when done
kubectl logs job/pi         # prints the value
```
**Explanation:** a **Job** runs one or more pods **to completion** (exit 0) rather than forever. `COMPLETIONS` shows succeeded/desired (`1/1`). Unlike a Deployment, a finished Job does **not** restart its pod. You read the result from the pod's **logs**. Jobs are for batch/one-shot work.

### 19. CronJob that prints the date every minute; suspend it; list its Jobs
```bash
kubectl create cronjob datecj --image=busybox --schedule="* * * * *" -- /bin/sh -c "date"
kubectl patch cronjob datecj -p '{"spec":{"suspend":true}}'   # stop scheduling new Jobs
kubectl get jobs                                              # datecj-<timestamp> entries
```
**Explanation:** a **CronJob** creates a **Job** on a cron **schedule** (`* * * * *` = every minute: minute hour day-of-month month day-of-week). `suspend: true` stops it from spawning **new** Jobs (already-running ones continue). Each run produces a Job named `datecj-<timestamp>`, visible via `kubectl get jobs`.

### 20. Manually run a Job from an existing CronJob
```bash
kubectl create job manual-run-1 --from=cronjob/datecj
```
**Explanation:** `--from=cronjob/...` copies the CronJob's Job template and runs it **immediately, once**, independent of the schedule. Perfect for **testing on demand** without waiting for the next tick or editing the schedule.

### 21. StatefulSet ordered create/terminate vs Deployment
```bash
kubectl get pods -l app=redis -w   # watch: redis-0 Ready, then -1, then -2
```
**Explanation:** a **StatefulSet** brings pods up **in order** `0 → 1 → 2`, waiting for each to be Ready before starting the next, and **tears them down in reverse** `2 → 1 → 0`. This guarantees, e.g., a primary is ready before replicas join. A **Deployment** creates and deletes pods **in parallel, in no particular order**, with random names — fine for stateless apps where every replica is interchangeable.

---

## Volumes, ConfigMaps & Secrets

### 22. Two containers sharing an `emptyDir`; one writes, one reads; prove it
```yaml
spec:
  volumes: [{ name: shared, emptyDir: {} }]
  containers:
    - name: writer
      image: busybox
      command: ["sh","-c","echo hello > /data/msg.txt && sleep 3600"]
      volumeMounts: [{ name: shared, mountPath: /data }]
    - name: reader
      image: busybox
      command: ["sh","-c","sleep 3600"]
      volumeMounts: [{ name: shared, mountPath: /data }]
```
```bash
kubectl exec -it <pod> -c reader -- cat /data/msg.txt    # prints "hello"
```
**Explanation:** an **`emptyDir`** is created empty when the pod starts and **lives for the pod's lifetime**, on the node. Because both containers mount the *same* volume, the file the **writer** creates is visible to the **reader** — proving shared storage. Deleting the pod deletes the emptyDir's data.

### 23. List StorageClasses, PVs and PVCs
```bash
kubectl get storageclass     # or: kubectl get sc
kubectl get pv               # PersistentVolumes (cluster-wide, not namespaced)
kubectl get pvc              # PersistentVolumeClaims (namespaced)
```
**Explanation:** a **StorageClass** describes a *type* of storage and how to **dynamically provision** it. A **PersistentVolume (PV)** is an actual piece of storage in the cluster. A **PersistentVolumeClaim (PVC)** is a user's *request* for storage that **binds** to a PV. PVs are cluster-scoped; PVCs are namespaced.

### 24. Mount a ConfigMap as a volume (each key → a file); verify
```bash
kubectl create configmap appcfg --from-literal=color=blue --from-literal=mode=dark
```
```yaml
volumes: [{ name: cfg, configMap: { name: appcfg } }]
# in the container:
volumeMounts: [{ name: cfg, mountPath: /etc/appcfg }]
```
```bash
kubectl exec -it <pod> -- sh -c 'ls /etc/appcfg && cat /etc/appcfg/color'   # files: color, mode
```
**Explanation:** when a ConfigMap is mounted as a volume, **each key becomes a file** (filename = key, contents = value) in the mount directory. Whole-volume ConfigMap mounts are **updated automatically** (eventually) when the ConfigMap changes — handy for config reloads.

### 25. Mount a single ConfigMap key with `subPath`; when needed + update caveat
```yaml
volumeMounts:
  - name: cfg
    mountPath: /etc/nginx/nginx.conf
    subPath: nginx.conf
volumes:
  - name: cfg
    configMap: { name: nginxcfg }   # has key nginx.conf
```
**Explanation:** **`subPath`** mounts **one file** into an **existing directory without hiding the directory's other files**. You need it when you must replace a single file like `/etc/nginx/nginx.conf` but keep the rest of `/etc/nginx`. **Caveat:** `subPath` mounts **do NOT receive automatic updates** when the ConfigMap/Secret changes (regular volume mounts do) — you must restart the pod to pick up new content.

### 26. Mount a Secret as a volume; verify decoded values + restrictive permissions
```yaml
volumes:
  - name: sec
    secret:
      secretName: dbsecret
      defaultMode: 0400          # owner read-only
volumeMounts: [{ name: sec, mountPath: /etc/secret, readOnly: true }]
```
```bash
kubectl exec -it <pod> -- sh -c 'ls -l /etc/secret && cat /etc/secret/password'
```
**Explanation:** a Secret mounted as a volume produces **one file per key containing the already-decoded (plaintext) value** — Kubernetes base64-*decodes* it for you on mount. `defaultMode: 0400` makes the files **owner-read-only**, limiting which processes can read them. Secret volumes are also stored in **tmpfs (RAM)**, not written to disk on the node.

### 27. Scaling a StatefulSet → one PVC per replica; what happens on delete?
```yaml
spec:
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: 1Gi } }
```
```bash
kubectl get pvc      # data-redis-0, data-redis-1, data-redis-2 ...
```
**Explanation:** a **`volumeClaimTemplate`** gives **each replica its own PVC** (`data-redis-0`, `data-redis-1`, …), so each pod keeps its **own persistent data**. Scaling up creates new PVCs. **Deleting the StatefulSet does NOT delete those PVCs by default** — they (and the data) are retained so you don't lose data accidentally; you must delete them manually. (Newer clusters can change this via `persistentVolumeClaimRetentionPolicy`.)

### 28. Use an initContainer to pre-populate a shared volume
```yaml
spec:
  volumes: [{ name: shared, emptyDir: {} }]
  initContainers:
    - name: seed
      image: busybox
      command: ["sh","-c","echo seeded > /work/data.txt"]
      volumeMounts: [{ name: shared, mountPath: /work }]
  containers:
    - name: app
      image: busybox
      command: ["sh","-c","cat /work/data.txt && sleep 3600"]
      volumeMounts: [{ name: shared, mountPath: /work }]
```
**Explanation:** **initContainers** run **to completion, in order, before** any app container starts. Here the init container writes data into the shared `emptyDir`; only after it finishes does the main container start and find the data ready. Common uses: fetching config/data, waiting for a dependency, running migrations.

### 29. `readOnly: true` on a volume mount; prove writes are rejected
```yaml
volumeMounts: [{ name: shared, mountPath: /data, readOnly: true }]
```
```bash
kubectl exec -it <pod> -- sh -c 'echo x > /data/test' 
# -> sh: can't create /data/test: Read-only file system
```
**Explanation:** `readOnly: true` mounts the volume so the container **cannot write** to it; any write attempt fails with **"Read-only file system."** Useful for injecting config/secrets the app must not modify, and for hardening.

### 30. Set a `sizeLimit` on an emptyDir; what if it's exceeded?
```yaml
volumes:
  - name: scratch
    emptyDir: { sizeLimit: 100Mi }
```
**Explanation:** `sizeLimit` caps how much the `emptyDir` may consume. If a container **writes beyond the limit**, the kubelet **evicts the pod** (ephemeral-storage limit exceeded) — it is removed and, if managed by a controller, rescheduled. This protects the node from a runaway pod filling its disk.

### 31. Generic Secret from literals; show values are base64-encoded, not encrypted
```bash
kubectl create secret generic dbsecret \
  --from-literal=username=admin --from-literal=password=s3cret
kubectl get secret dbsecret -o yaml          # values look like YWRtaW4=, czNjcmV0
echo 'czNjcmV0' | base64 -d                  # -> s3cret  (trivially reversible)
```
**Explanation:** Secret values are stored **base64-encoded**, which is **encoding, not encryption** — anyone who can `get` the Secret can decode it instantly. At rest in **etcd** it's plaintext unless the cluster has **encryption-at-rest** configured. So a Secret's protection comes from **RBAC** (who can read it), not from the encoding.

### 32. Secret from files (`--from-file`) holding a cert and key; resulting keys
```bash
kubectl create secret generic tls-files \
  --from-file=server.crt --from-file=server.key
kubectl get secret tls-files -o jsonpath='{.data}'   # keys: server.crt, server.key
```
**Explanation:** with `--from-file=<path>`, the **filename becomes the key** and the file's contents become the (base64-encoded) value. So you get keys `server.crt` and `server.key`. You can rename the key with `--from-file=tls.crt=server.crt`.

### 33. `kubernetes.io/tls` typed Secret from a cert/key pair; where is it consumed?
```bash
kubectl create secret tls my-tls --cert=server.crt --key=server.key
```
**Explanation:** this makes a Secret of **type `kubernetes.io/tls`** with the two fixed keys **`tls.crt`** and **`tls.key`**. It's consumed mainly by **Ingress** resources to **terminate HTTPS/TLS** (`spec.tls[].secretName`), and by applications/sidecars that need a server certificate. The typed form lets controllers validate that both keys are present.

### 34. Mount a Secret as a volume — security trade-offs vs env vars
**Explanation:**
- **Volume (files):** can have **restrictive file permissions** (`defaultMode`), is stored in **tmpfs (RAM)**, **updates propagate** to the file (except `subPath`), is **not shown** in `kubectl describe pod`, and is **not inherited by child processes' environment**. More secure, slightly more setup.
- **Env vars:** simpler, but the values appear in the **pod spec / `describe` output**, are **inherited by every child process**, can **leak into logs or crash dumps**, and **don't update** without a pod restart.
**Conclusion:** prefer **volume mounts** for sensitive data; use env vars for convenience with low-sensitivity config.

### 35. `envFrom` with `secretRef` to load all keys as env vars
```yaml
envFrom:
  - secretRef: { name: dbsecret }
```
**Explanation:** `envFrom` injects **every key** of the Secret as an environment variable in one shot (key → variable name, decoded value → value). Saves listing each key individually. (`configMapRef` does the same for ConfigMaps.) Beware key names that aren't valid env var identifiers — they're skipped.

### 36. docker-registry (image-pull) Secret + `imagePullSecrets`; when required
```bash
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=USER --docker-password=PASS
```
```yaml
spec:
  imagePullSecrets: [{ name: regcred }]
```
**Explanation:** required whenever the image lives in a **private registry** (or you want **authenticated DockerHub** pulls to dodge rate limits). The kubelet presents these credentials when pulling. Attach per-pod via `imagePullSecrets`, or add it to a **ServiceAccount** so all its pods inherit it.

### 37. Secret with several keys — selectively mount only one
```yaml
volumes:
  - name: sec
    secret:
      secretName: multi
      items:
        - { key: password, path: password.txt }   # only this key is mounted
volumeMounts: [{ name: sec, mountPath: /etc/secret, readOnly: true }]
```
**Explanation:** the **`items`** list controls **exactly which keys** are projected and **at what filename/path**. Keys not listed are **not** mounted. This implements least privilege — a pod sees only the secret data it needs.

---

## Services & networking

### 38. ClusterIP Service + DNS resolution from another pod
```bash
kubectl expose deployment web --port=80          # ClusterIP by default
kubectl run tmp --rm -it --image=busybox -- nslookup web.default.svc.cluster.local
```
**Explanation:** a **ClusterIP** Service gets a stable **virtual IP** reachable only inside the cluster, plus a DNS name **`<svc>.<namespace>.svc.cluster.local`**. CoreDNS resolves it; within the same namespace the **short name `web`** also works thanks to search domains.

### 39. NodePort Service + reach it on minikube
```bash
kubectl expose deployment web --type=NodePort --port=80
minikube service web --url           # prints a reachable URL
curl $(minikube ip):<nodePort>       # nodePort is in 30000–32767
```
**Explanation:** a **NodePort** opens the **same high port (30000–32767) on every node** and forwards it to the Service. On minikube, `minikube service web --url` gives a working URL, or hit `$(minikube ip):<nodePort>` directly. NodePort is the simplest way to reach a service from outside a bare cluster.

### 40. LoadBalancer Service — why `<pending>` on minikube, and `minikube tunnel`
```bash
kubectl expose deployment web --type=LoadBalancer --port=80
kubectl get svc web        # EXTERNAL-IP: <pending>
minikube tunnel            # run in a separate terminal; then EXTERNAL-IP appears
```
**Explanation:** `type: LoadBalancer` asks the **cloud provider** to provision an external load balancer and assign an `EXTERNAL-IP`. **minikube has no cloud LB**, so the field stays **`<pending>`**. **`minikube tunnel`** runs a local process that simulates the LB and routes an external IP to the Service, so it becomes reachable.

### 41. Headless Service (clusterIP: None) for a StatefulSet; per-pod A records
```yaml
apiVersion: v1
kind: Service
metadata: { name: redis }
spec:
  clusterIP: None
  selector: { app: redis }
  ports: [{ port: 6379 }]
```
```bash
kubectl run tmp --rm -it --image=busybox -- nslookup redis   # returns multiple pod IPs
```
**Explanation:** with **`clusterIP: None`** there is **no virtual IP**; DNS for the Service returns **one A record per backing pod** (the pod IPs), and StatefulSet pods additionally get `pod-N.<svc>...` names. This lets clients discover and address **individual pods** — necessary for stateful clusters that need stable peer identities.

### 42. Inspect Endpoints / EndpointSlice; how they're populated
```bash
kubectl get endpoints web
kubectl get endpointslices -l kubernetes.io/service-name=web -o wide
```
**Explanation:** the endpoints/EndpointSlice controller **watches pods that match the Service's selector and are Ready**, and records their **IP:port**. So Endpoints are derived **automatically from the selector + readiness**. If a pod isn't Ready it's excluded (listed under not-ready addresses). **Empty Endpoints** is the classic sign that the selector matches nothing or no pod is Ready.

### 43. Cross-namespace access via FQDN; short name fails
```bash
# from a pod in namespace A, reaching service "db" in namespace B:
wget -qO- db.B.svc.cluster.local      # works
wget -qO- db                          # fails (resolves only in the caller's namespace)
```
**Explanation:** the **fully-qualified name `<svc>.<namespace>.svc.cluster.local`** works from **any** namespace. The **short name** relies on the pod's DNS **search domains**, which include only the **pod's own namespace**, so `db` only resolves if the service is in the *same* namespace as the caller.

### 44. `kubectl port-forward` to a Service vs to a Pod
```bash
kubectl port-forward svc/web 8080:80      # forwards to one pod behind the Service
kubectl port-forward pod/web-abc 8080:80  # forwards to that specific pod
```
**Explanation:** both create a **local tunnel** from your machine into the cluster. Forwarding **to a Service** selects **one** backing pod (it does **not** load-balance) — convenient when you don't care which pod. Forwarding **to a Pod** targets an **exact** pod — what you want when **debugging a specific replica**. If the chosen pod dies, the forward breaks and must be restarted.

### 45. Service `port` vs `targetPort` vs `nodePort`
**Explanation:**
- **`port`** — the port the **Service** itself listens on (its ClusterIP:port that other pods call).
- **`targetPort`** — the **container** port the Service forwards traffic to (can be a number or a named port).
- **`nodePort`** — only for `type: NodePort`/`LoadBalancer`; the port opened on **every node** (range 30000–32767) for external access.
Traffic path: client → `nodePort` (external) → Service `port` → pod `targetPort`.

### 46. Verify connectivity to a ClusterIP from a throwaway debug pod
```bash
kubectl run tmp --rm -it --image=busybox -- sh
# inside the pod:
wget -qO- web            # or: wget -qO- web.default.svc.cluster.local
nc -zv web 80            # TCP connectivity check
```
**Explanation:** `kubectl run --rm -it` starts an **interactive, self-deleting** pod — a disposable network probe. From inside it you test the Service by **DNS name** with `wget`/`nc`. Success proves DNS, Endpoints, and the target port all work; failure tells you which layer to investigate (see item 48).

### 47. Two Deployments, one Service load-balancing across both via a shared label
```bash
# both deployments' pod templates carry the label app=shared
kubectl create deployment d1 --image=nginx:1.25
kubectl create deployment d2 --image=nginx:1.25
kubectl label deployment d1 d2 --all   # (or set the pod-template label in YAML: app: shared)
```
```yaml
# Service selects the common label, not the per-deployment label
apiVersion: v1
kind: Service
metadata: { name: shared }
spec:
  selector: { app: shared }
  ports: [{ port: 80, targetPort: 80 }]
```
```bash
# prove both are hit (use an image that returns its hostname, then loop):
kubectl run tmp --rm -it --image=busybox -- sh -c 'for i in $(seq 10); do wget -qO- shared; done'
```
**Explanation:** a Service targets pods purely by **label selector** — it doesn't care which Deployment owns them. If pods from **both** Deployments share the label `app=shared`, the Service's Endpoints include **all** of them and requests are **load-balanced across both**. Repeated requests returning different pod hostnames prove it. (Set `app: shared` in each pod template's labels.)

### 48. Systematically diagnose why a pod can't reach a Service
```bash
# 1) DNS resolves?
kubectl run tmp --rm -it --image=busybox -- nslookup <svc>
# 2) Endpoints populated?
kubectl get endpoints <svc>          # empty? -> selector/readiness problem
# 3) Selector labels actually match the pods?
kubectl get svc <svc> -o jsonpath='{.spec.selector}'; kubectl get pods --show-labels
# 4) Pods Ready? (failing readiness => excluded from Endpoints)
kubectl get pods
# 5) Right port? app listening on targetPort?
kubectl describe svc <svc>
```
**Explanation:** work outward in order: **DNS → Endpoints → selector labels → readiness → port**. DNS failure points at CoreDNS/name. Empty **Endpoints** means the **selector matches nothing** or no pod is **Ready**. If labels match but Endpoints are still empty, a **failing readiness probe** is excluding the pods. Finally confirm the **targetPort** matches the port the app actually listens on. This sequence isolates almost every "can't reach my service" problem.

### 49. Expose a service with NodePort and verify
```bash
kubectl expose deployment web --type=NodePort --port=80
kubectl get svc web -o wide                    # note the 80:3xxxx/TCP nodePort
curl $(minikube ip):<nodePort>                 # or: minikube service web --url
```
**Explanation:** same mechanism as item 39 — NodePort opens a high port on every node mapped to the Service. Verify by curling the node IP at that port (on minikube use `minikube ip` or `minikube service`).

---

## Health probes

### 50. HTTP livenessProbe on `/` port 80 for nginx; verify
```yaml
livenessProbe:
  httpGet: { path: /, port: 80 }
  initialDelaySeconds: 5
  periodSeconds: 10
```
```bash
kubectl describe pod <pod>     # shows: Liveness: http-get http://:80/ ...
```
**Explanation:** a **liveness probe** checks whether the container is still **healthy**; if it **fails repeatedly**, the kubelet **restarts the container**. The `httpGet` probe considers HTTP **200–399** a success. `initialDelaySeconds` waits before the first check (so a slow start isn't punished); `periodSeconds` is the check interval. `kubectl describe pod` shows the configured probe and any failures in Events.

### 51. tcpSocket readiness probe on redis port 6379; verify
```yaml
readinessProbe:
  tcpSocket: { port: 6379 }
  initialDelaySeconds: 5
  periodSeconds: 10
```
```bash
kubectl get pod <pod>          # READY 1/1 only once the probe passes
kubectl describe pod <pod>     # shows: Readiness: tcp-socket :6379 ...
```
**Explanation:** a **readiness probe** decides whether the pod should **receive traffic**; while it fails, the pod is **removed from the Service's Endpoints** (but **not** restarted). A **`tcpSocket`** probe simply checks that **a TCP connection to the port succeeds** — ideal for non-HTTP services like Redis. Until 6379 accepts connections, the pod stays `0/1 READY` and gets no traffic.

### 52. Startup probe with a ~2-minute budget; justify the numbers
```yaml
startupProbe:
  httpGet: { path: /, port: 80 }
  failureThreshold: 24
  periodSeconds: 5
```
**Explanation:** the **startup probe** protects **slow-booting** apps: while it runs, the **liveness and readiness probes are disabled**, so a long start can't trigger a premature restart. The total grace period is **`failureThreshold × periodSeconds` = 24 × 5 = 120 s (~2 minutes)**. We pick `periodSeconds: 5` to check reasonably often and `failureThreshold: 24` so the app has up to two minutes to come up; only after that budget is exhausted is the container considered failed and restarted. Once the startup probe **succeeds once**, liveness/readiness take over.

### 53. Default behaviour when no probes are defined
**Explanation:** with **no probes at all**:
- **No readiness probe** → the pod is treated as **Ready as soon as the container starts**, so it is **added to Service Endpoints immediately** — even if the app inside hasn't finished initializing (it can receive traffic too early).
- **No liveness probe** → the kubelet **only restarts the container if its process exits/crashes**. A container that is **hung or deadlocked but still running** is **never detected or restarted**.
So with no probes, Kubernetes assumes "running == healthy and ready," which is why explicit probes matter for real availability.

---

*End of LO4 answer key. Exam tip: most "why is it Pending / not Ready / not reachable" questions reduce to **scheduling (selector/affinity/resources)** or the **DNS → Endpoints → selector → readiness → port** chain. Know those two and you can reason through almost anything here.*
