# Intro to DevOps — Exam Study Guide (LO4 · LO5 · LO6)

### Red Hat Academy track (DO180 / DO188) · Kubernetes + OpenShift + Podman

---

## How to use this document

This is your single source of truth for the three exam outcomes. Each practical item follows the same four-line rhythm so you can scan fast under time pressure:

| Tag | Meaning |
|---|---|
| **Q** | The question, in the wording the exam tends to use. |
| **DO** | The exact commands to type. `oc` and `kubectl` are shown **side by side** — pick whichever your terminal has. |
| **VERIFY** | What to run (and screenshot) to prove it worked. |
| **WHY** | The one-paragraph explanation the grader wants to hear. |

**Search trick:** every item has a tag like `{L4.7}`. CTRL-F the tag or a phrase from the question to jump straight to it.

**The golden rule of `oc` vs `kubectl`:** On OpenShift, `oc` *is* `kubectl` with extra verbs bolted on. Anything `kubectl X` does, `oc X` does identically. The only real differences are the OpenShift-only conveniences: **projects** (`oc new-project`), **Routes** (`oc expose` → a public URL), **`oc new-app`** (build+deploy in one shot), **`oc rsh`/`oc debug`**, and **SCC** security. Those are called out wherever they matter.

> **Environment reminder (the lab cluster):** registry is `registry.ocp4.example.com:8443`, API is `https://api.ocp4.example.com:6443`. Logins: `developer`/`developer`, `admin`/`redhatocp`. On the node terminal, `chroot /host` to get host binaries.

---

## The three outcomes at a glance

- **LO4 — Operate Kubernetes/OpenShift workloads (practical).** Create and manage Deployments, ReplicaSets, StatefulSets, DaemonSets, Jobs/CronJobs; wire up Services and probes; mount ConfigMaps, Secrets, and volumes. *You will be asked to make the cluster do something and prove it.*
- **LO5 — Ship applications: find and fix the mistake (practical).** A container, image build, or manifest is broken on purpose. Diagnose from logs/events/describe, name the root cause, fix it. *You will be asked to repair something and explain what was wrong.*
- **LO6 — Evaluate orchestration systems (theory).** Compare Kubernetes, OpenShift, Podman, Docker, Swarm, k3s. Take a position and defend it with trade-offs. *You will be asked to argue a recommendation in writing.*

---

# CHEATSHEETS

Keep this section open in a second window during the exam.

## 1 · The 10 commands that solve most LO4 questions

```bash
# --- kubectl form ---                          # --- oc form (OpenShift) ---
kubectl get pods -o wide -l app=web             oc get pods -o wide -l app=web
kubectl get pods,rs,deploy,svc                  oc get pods,rs,deploy,svc      # oc get all
kubectl describe pod <pod>                      oc describe pod <pod>          # events at bottom
kubectl logs <pod> [-c CTR] [--previous] [-f]   oc logs <pod> [-c CTR] [-p] [-f]
kubectl exec -it <pod> [-c CTR] -- sh           oc rsh <pod>                   # oc exec also works
kubectl apply -f file.yaml                      oc apply -f file.yaml
kubectl delete -f file.yaml                     oc delete -f file.yaml
kubectl rollout status|history|undo deploy/web  oc rollout status|history|undo deploy/web
kubectl get <obj> <name> -o yaml                oc get <obj> <name> -o yaml
kubectl get <obj> <name> -o jsonpath='{.path}'  oc get <obj> <name> -o jsonpath='{.path}'
```

**The single most useful exam trick — generate clean YAML without touching the cluster:**

```bash
kubectl create deployment web --image=nginx --dry-run=client -o yaml > web.yaml
oc      create deployment web --image=nginx --dry-run=client -o yaml > web.yaml
kubectl run p --image=nginx --dry-run=client -o yaml > pod.yaml      # same for a bare Pod
```

Edit the generated file, then `apply -f`. Faster and less error-prone than writing YAML from scratch.

## 2 · kubectl ↔ oc translation table

| Task | `kubectl` | `oc` |
|---|---|---|
| Switch working namespace/project | `config set-context --current --namespace=x` | `oc project x` |
| Create namespace/project | `create namespace x` | `oc new-project x` |
| Build **and** deploy from source/image | *(not built-in)* | `oc new-app IMAGE~GITURL` |
| Expose a Deployment internally | `expose deploy/web` → **Service** | `oc expose deploy/web` → **Service** |
| Expose externally (public URL) | `create ingress …` (needs controller) | `oc expose svc/web` → **Route** (URL) |
| List external URLs | `get ingress` | `oc get routes` |
| Shell into a pod | `exec -it P -- sh` | `oc rsh P` |
| Debug a workload | `kubectl debug` | `oc debug deploy/web` |
| Inject env from a Secret/ConfigMap | *(edit YAML / set env)* | `oc set env deploy/web --from=secret/s` |
| Mount a volume onto a Deployment | *(edit YAML)* | `oc set volume deploy/web --add …` |
| Grant a role to a user | `create rolebinding` | `oc adm policy add-role-to-user` |
| Allow a pod to run as root | *(PodSecurity labels)* | `oc adm policy add-scc-to-user anyuid -z SA` |

## 3 · Resource short-names (stop typing the long form)

| Short | Full | What it is |
|---|---|---|
| `po` | pods | The container cages — smallest deployable unit. |
| `deploy` | deployments | Manager that keeps **stateless** pods alive. |
| `rs` | replicasets | The replica-count math engine a Deployment owns. |
| `sts` | statefulsets | Manager for **numbered, disk-locking** pods (databases). |
| `ds` | daemonsets | Forces exactly **1 pod onto every node**. |
| `cj` | cronjobs | Cron clock that spawns Jobs on a schedule. |
| *(none)* | jobs | Run-to-completion task. |
| `svc` | services | Internal stable virtual-IP load balancer. |
| `ing` | ingress | Layer-7 HTTP router (OpenShift uses **Routes**). |
| `ep` | endpoints | Live list of pod IPs glued to a Service. |
| `netpol` | networkpolicies | Pod-to-pod firewall rules. |
| `cm` | configmaps | Non-secret config (files / env vars). |
| *(none)* | secrets | Base64-encoded sensitive data. |
| `pvc` | persistentvolumeclaims | A pod's *ticket* requesting disk. |
| `pv` | persistentvolumes | The actual disk slice. |
| `sc` | storageclasses | Automated disk provisioner (standard vs fast-nvme). |
| `no` | nodes | Worker machines. |
| `ns` | namespaces | Virtual cluster rooms (OpenShift: **projects**). |
| `sa` | serviceaccounts | The identity a pod uses to call the K8s API. |

> **Forgot a short-name?** `kubectl api-resources` (or `oc api-resources`) prints every object, its API group, and its official short-name. This is your in-exam panic button.

**Add-on flags that change the camera angle:**

```bash
-o wide          # reveal hidden columns: pod IP + which NODE it landed on
--show-labels    # show every label (find selector mismatches)
-l key=value     # filter by label
-w               # --watch: stream changes live (great for ordered StatefulSet scaling)
--sort-by=.metadata.creationTimestamp   # order events/objects by time
-o jsonpath='{.path}'                   # pull one exact field
```

## 4 · Pod diagnosis decision tree (memorize this)

`kubectl describe pod <pod>` → read the **Events** at the bottom, then map the symptom:

```
Pending                       → Insufficient cpu/mem | nodeSelector/affinity unmet | taint | unbound PVC
ImagePullBackOff / ErrImagePull → image name/tag typo | private registry needs imagePullSecrets
CrashLoopBackOff              → app starts then exits → kubectl logs <pod> --previous
Completed (then restarts)     → no long-running process → give it a real command
OOMKilled                     → lastState.reason → raise limits.memory
CreateContainerConfigError    → missing ConfigMap/Secret key referenced by the pod
Init:0/1 (stuck)              → initContainer failing → kubectl logs <pod> -c <init>
Terminating (stuck)          → finalizer or long grace period → --grace-period=0 --force (risky)
```

**Service returns nothing?** → `kubectl get endpoints <svc>`. **Empty list = label mismatch or no Ready pods** (the #1 Service bug).

## 5 · Podman quick reference (LO5 container half)

```bash
podman run -d --name web -p HOST:CTR --net NET -e K=V IMAGE [cmd]   # detached, named, mapped
podman ps -a                         # list ALL containers incl. exited (find the dead one)
podman logs [-f] <ctr>               # why did it die?
podman inspect <ctr> --format '{{.State.Status}}'        # one field
podman inspect <ctr> --format '{{.NetworkSettings.Networks}}'   # which networks?
podman exec [-it] <ctr> <cmd>        # run inside a running container
podman cp <ctr>:/path ./local        # copy out (or local → ctr)
podman network create NET            # custom network = automatic DNS by container name
podman build -f Containerfile -t NAME:TAG .
podman push NAME registry/.../name:tag
```

- **Port map** is always `-p HOST:CONTAINER`. Getting this backwards is the #1 LO5 trap.
- **DNS by name only works on a *custom* network**, never on the default one.
- **Bind-mount on SELinux hosts needs `:Z`** (`-v ./html:/usr/share/nginx/html:Z`) or the container is denied access.

---
# LO4 — Operate Kubernetes / OpenShift workloads (practical)

> Every command works the same with `oc` or `kubectl` unless noted. Replace `<pod>` with the real name from `get pods`. Namespace/project assumed current; add `-n NS` (kubectl) / `oc project NS` otherwise.

## Part 1 — Deployments, ReplicaSets & rollouts

### {L4.1} Create a Deployment imperatively and export its YAML
**Q:** Create a Deployment named `web` running `nginx:1.25` with 3 replicas using a single imperative create command, then export the generated YAML to a file.

**DO**
```bash
# kubectl                                        # oc
kubectl create deployment web \                  oc create deployment web \
  --image=nginx:1.25 --replicas=3                  --image=nginx:1.25 --replicas=3

# export the live object to a file:
kubectl get deploy web -o yaml > web.yaml        oc get deploy web -o yaml > web.yaml
# OR generate a CLEAN manifest without creating it:
kubectl create deployment web --image=nginx:1.25 --replicas=3 --dry-run=client -o yaml > web.yaml
```
**VERIFY**
```bash
kubectl get deploy web ; kubectl get pods -l app=web ; cat web.yaml
```
**WHY** One imperative `create deployment` builds the object. `-o yaml` dumps the **live** object (with status/defaults); `--dry-run=client -o yaml` produces a **clean** manifest without touching the cluster. (Old kubectl lacks `--replicas` on create → `scale` afterward.)

### {L4.2} Authenticated image pulls (DockerHub rate limit)
**Q:** Enable authenticated pulls from DockerHub to avoid the anonymous pull limit. What resource do you create?

**DO** — a **docker-registry Secret** (type `kubernetes.io/dockerconfigjson`):
```bash
kubectl create secret docker-registry dockerhub \      # identical with: oc create secret ...
  --docker-server=docker.io \
  --docker-username=YOURUSER --docker-password=YOURPASS --docker-email=you@example.com

# attach it to the default ServiceAccount (covers every pod in the namespace):
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets":[{"name":"dockerhub"}]}'
# oc shortcut for the same link:
oc secrets link default dockerhub --for=pull
```
Per-deployment alternative: `spec.template.spec.imagePullSecrets: [{name: dockerhub}]`.
**VERIFY**
```bash
kubectl get secret dockerhub -o jsonpath='{.type}'; echo     # kubernetes.io/dockerconfigjson
kubectl get sa default -o jsonpath='{.imagePullSecrets}'; echo
```
**WHY** The resource is a **Secret of type docker-registry**. Authenticated pulls get a higher rate limit than anonymous. Same pattern with `--docker-server=quay.io`.

### {L4.3} Scale two ways and show the ReplicaSet
**Q:** Scale `web` from 3 to 5 replicas two different ways — once with `scale`, once by editing the manifest — and show the resulting ReplicaSet.

**DO**
```bash
kubectl scale deployment web --replicas=5        # way 1: imperative   (oc scale ... identical)
kubectl edit deployment web                      # way 2: change spec.replicas to 5, save
# (or edit web.yaml → spec.replicas: 5 → kubectl apply -f web.yaml)
```
**VERIFY**
```bash
kubectl get deploy web
kubectl get rs -l app=web      # SAME ReplicaSet, DESIRED now 5
kubectl get pods -l app=web
```
**WHY** Both just change `spec.replicas`. **No new ReplicaSet** appears because the pod *template* is unchanged — a new RS is created only when the template changes (image/env/etc.).

### {L4.4} Rolling update and watch it
**Q:** Roll `web` from `nginx:1.25` to `nginx:1.27` and watch with `rollout status`.

**DO**
```bash
kubectl set image deployment/web web=nginx:1.27   # 'web' = the CONTAINER name (oc set image identical)
kubectl rollout status deployment/web
```
**VERIFY**
```bash
kubectl get rs -l app=web        # new RS scaled up, old RS scaled to 0
kubectl describe deployment web | grep -i image
```
**WHY** `set image` patches the pod template → new RS, pods replaced gradually. Confirm the container name: `kubectl get deploy web -o jsonpath='{.spec.template.spec.containers[*].name}'`.

### {L4.5} Rollout history & rollback; the CHANGE-CAUSE column
**Q:** View rollout history and roll back to the previous revision. Explain what CHANGE-CAUSE shows and how to populate it.

**DO**
```bash
kubectl rollout history deployment/web
kubectl rollout history deployment/web --revision=2     # detail of one revision
kubectl rollout undo deployment/web                     # back to previous
kubectl rollout undo deployment/web --to-revision=1     # to a specific revision
# populate CHANGE-CAUSE:
kubectl annotate deployment/web kubernetes.io/change-cause="update to nginx:1.27" --overwrite
```
**VERIFY** `kubectl rollout history deployment/web` → CHANGE-CAUSE column now filled.
**WHY** CHANGE-CAUSE shows the `kubernetes.io/change-cause` annotation per revision; it's `<none>` until you set it. (Legacy `--record` is deprecated.)

### {L4.6} Set RollingUpdate maxSurge/maxUnavailable
**Q:** Set strategy to RollingUpdate with `maxSurge: 1`, `maxUnavailable: 0`; explain the availability effect.

**DO** (patch or edit `spec.strategy`):
```bash
kubectl patch deployment web -p '{"spec":{"strategy":{"type":"RollingUpdate","rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'
```
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # may run 1 EXTRA pod above desired during the update
      maxUnavailable: 0    # never drop below desired (zero downtime)
```
**VERIFY** `kubectl get deploy web -o jsonpath='{.spec.strategy}'; echo`
**WHY** `maxUnavailable: 0` guarantees the full replica count stays Ready throughout; `maxSurge: 1` spins one new pod first, then retires an old one — **zero-downtime, slightly more resource use**.

### {L4.7} Switch to Recreate strategy
**Q:** Switch `web` to the Recreate strategy and give a concrete scenario where it's required instead of RollingUpdate.

**DO**
```bash
kubectl patch deployment web -p '{"spec":{"strategy":{"type":"Recreate"}}}'
```
**VERIFY** `kubectl get deploy web -o jsonpath='{.spec.strategy.type}'; echo` → `Recreate`
**WHY** Recreate **kills all old pods, then starts new ones** → brief downtime. Required when old and new versions **cannot run at the same time** — e.g. a DB schema migration where two versions writing concurrently would corrupt data, or a single-writer `ReadWriteOnce` volume that only one pod may mount.

### {L4.8} Add CPU/memory requests and limits
**Q:** Add CPU/memory requests and limits to a container and verify they appear in the running pod spec.

**DO** (edit `spec.template.spec.containers[].resources`):
```yaml
resources:
  requests: { cpu: "100m", memory: "128Mi" }   # scheduler reserves at least this
  limits:   { cpu: "500m", memory: "256Mi" }   # hard ceiling; over memory → OOMKilled
```
`oc set resources deployment/web --requests=cpu=100m,memory=128Mi --limits=cpu=500m,memory=256Mi`
**VERIFY** `kubectl get pod <pod> -o jsonpath='{.spec.containers[0].resources}'; echo`
**WHY** **Requests** drive scheduling (a node must have that much free) and **limits** cap usage (exceed memory → **OOMKilled**; exceed CPU → throttled, not killed).

### {L4.9} revisionHistoryLimit
**Q:** Set `revisionHistoryLimit: 3` and explain its effect on rollback and old ReplicaSets.

**DO** `kubectl patch deployment web -p '{"spec":{"revisionHistoryLimit":3}}'`
**VERIFY** `kubectl get rs -l app=web` → at most 3 old (scaled-to-0) ReplicaSets retained.
**WHY** Kubernetes keeps old ReplicaSets so you can roll back. `revisionHistoryLimit: 3` keeps the **3 most recent** old RSes; older ones are garbage-collected, so you can only roll back as far as the kept history.

### {L4.10} Stuck rollout on a bad tag
**Q:** Set the image to a non-existent tag, watch the rollout get stuck, and explain why old pods keep serving.

**DO**
```bash
kubectl set image deployment/web web=nginx:doesnotexist
kubectl rollout status deployment/web        # hangs; new pods ImagePullBackOff
kubectl get pods -l app=web
# recover:
kubectl rollout undo deployment/web          # or set a valid tag again
```
**WHY** The Deployment controller **won't scale down the old RS until new pods become Ready**. The new pods are stuck in `ImagePullBackOff`, so the old, healthy pods keep all the traffic — a rolling update is **safe by design**.

### {L4.11} Label selector & the Deployment→RS→Pod chain
**Q:** List only the pods of one Deployment with a label selector, and explain the Deployment → ReplicaSet → Pod selector relationship.

**DO** `kubectl get pods -l app=web` (add `--show-labels` to see them all)
**WHY** A Deployment's `spec.selector.matchLabels` selects its ReplicaSet; the RS's selector selects Pods carrying those labels (`app=web` by default). The chain is **label-driven**: Deployment manages RS, RS manages Pods, all matched by labels — which is why a selector/template-label mismatch is fatal (see {L5.26}).

### {L4.12} Expose a Deployment; what object is created
**Q:** Expose a Deployment with `expose`, then explain what object was created and how its selector was derived.

**DO**
```bash
kubectl expose deployment web --port=80 --target-port=80     # creates a Service (ClusterIP)
oc expose deployment web --port=80 --target-port=80          # same: a Service
oc expose svc/web                                            # OpenShift extra: Service → ROUTE (public URL)
```
**VERIFY** `kubectl get svc web -o wide` (SELECTOR column = `app=web`); `oc get routes`
**WHY** `expose deploy` creates a **Service** whose **selector is copied from the Deployment's pod labels** (`app=web`), so it automatically tracks those pods' IPs as Endpoints. On OpenShift, `oc expose svc/web` then creates a **Route** giving an external hostname (Ingress equivalent).

## Part 2 — Pods, multi-container & scheduling

### {L4.13} Add a sidecar container
**Q:** Add a second (sidecar) container to a Deployment's pod template; explain how the two share network and volumes.

**DO** (add to `spec.template.spec.containers`):
```yaml
containers:
  - name: web
    image: nginx:1.25
  - name: sidecar
    image: busybox:1.36
    command: ["sh","-c","tail -f /var/log/nginx/access.log"]
    volumeMounts: [{ name: logs, mountPath: /var/log/nginx }]
```
**WHY** Containers in one pod **share the pod's network namespace** (same IP, reach each other on `localhost`) and any **shared volume** (e.g. an `emptyDir`). That's the sidecar pattern: log shippers, proxies, adapters running beside the main app.

### {L4.14} Deployment manifest with named port + nodeSelector
**Q:** Write a Deployment for `httpd:2.4`, 2 replicas, a named container port, and a `nodeSelector`; explain why it stays Pending if no node matches.

**DO**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: web }
spec:
  replicas: 2
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      nodeSelector: { disktype: ssd }       # pod only schedules on nodes with this label
      containers:
        - name: httpd
          image: httpd:2.4
          ports: [{ name: http, containerPort: 80 }]   # named port
```
```bash
# fix by labelling a node:
kubectl label node <node> disktype=ssd
```
**WHY** The scheduler can only place a pod on a node that satisfies `nodeSelector`. If **no node has `disktype=ssd`**, the pod has nowhere to go → stays **Pending** forever (describe shows "0/N nodes available: node(s) didn't match node selector").

### {L4.15} Bare Pod vs Deployment-managed pod
**Q:** Create a bare Pod (no controller) running busybox that sleeps, then explain what happens when you delete it vs a Deployment-managed pod.

**DO**
```bash
kubectl run sleeper --image=busybox --restart=Never -- sleep 3600   # --restart=Never → bare Pod
```
**WHY** A **bare Pod** has no controller — delete it and it's **gone for good**. A Deployment-managed pod is owned by a ReplicaSet that **recreates** it to maintain the desired replica count. Bare pods are for one-off debugging only.

### {L4.22} Multi-container pod sharing an emptyDir
**Q:** Create a pod where two containers share an `emptyDir` — one writes a file, the other reads it. Prove it's shared.

**DO**
```yaml
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh","-c","echo hello > /data/msg; sleep 3600"]
      volumeMounts: [{ name: shared, mountPath: /data }]
    - name: reader
      image: busybox
      command: ["sh","-c","sleep 5; cat /data/msg; sleep 3600"]
      volumeMounts: [{ name: shared, mountPath: /data }]
  volumes: [{ name: shared, emptyDir: {} }]
```
**VERIFY** `kubectl logs <pod> -c reader` → prints `hello`
**WHY** An `emptyDir` is created with the pod and mounted into **both** containers, so the writer's file is visible to the reader. It lives and dies with the pod (cleared on pod deletion).

## Part 3 — StatefulSets, DaemonSets, Jobs & CronJobs

### {L4.16} StatefulSet + headless Service (stable names & DNS)
**Q:** Write a StatefulSet for `redis:7`, 3 replicas, plus a headless Service; show stable pod names and per-pod DNS.

**DO**
```yaml
apiVersion: v1
kind: Service
metadata: { name: redis }
spec:
  clusterIP: None             # headless → no VIP, returns per-pod A records
  selector: { app: redis }
  ports: [{ port: 6379 }]
---
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: redis }
spec:
  serviceName: redis          # MUST match the headless Service name
  replicas: 3
  selector: { matchLabels: { app: redis } }
  template:
    metadata: { labels: { app: redis } }
    spec:
      containers: [{ name: redis, image: redis:7, ports: [{ containerPort: 6379 }] }]
```
**VERIFY**
```bash
kubectl get pods -l app=redis          # redis-0, redis-1, redis-2 (stable, ordinal names)
kubectl run tmp --rm -it --image=busybox -- nslookup redis-0.redis   # per-pod DNS
```
**WHY** A StatefulSet gives **stable, ordinal identities** (`redis-0/1/2`) and a **headless Service** (`clusterIP: None`) returns each pod's own A record at `<pod>.<svc>.<ns>.svc.cluster.local` — essential for clustered databases that must address each member individually.

### {L4.17} DaemonSet — one pod per node
**Q:** Create a DaemonSet running a busybox agent; explain why exactly one pod runs per node.

**DO**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata: { name: agent }
spec:
  selector: { matchLabels: { app: agent } }
  template:
    metadata: { labels: { app: agent } }
    spec:
      containers: [{ name: agent, image: busybox, command: ["sleep","infinity"] }]
```
**VERIFY** `kubectl get pods -o wide -l app=agent` → one pod, one per distinct NODE.
**WHY** A DaemonSet has **no `replicas` field**. Its controller schedules **exactly one pod on every (eligible) node**, and automatically adds a pod when a new node joins. Used for per-node agents: log collectors, monitoring, CNI.

### {L4.18} Job that computes once and completes
**Q:** Create a Job (busybox/perl) that computes something once and completes; show COMPLETIONS and read the result.

**DO**
```bash
kubectl create job calc --image=busybox -- sh -c 'echo $((6*7))'
# perl variant (classic Red Hat example):
# kubectl create job pi --image=perl:5.34 -- perl -Mbignum=bpi -wle 'print bpi(200)'
```
**VERIFY** `kubectl get jobs` → `COMPLETIONS 1/1`; `kubectl logs job/calc` → `42`
**WHY** A **Job** runs a pod to completion (exit 0) and records success in COMPLETIONS. Unlike a Deployment it is **not** restarted once finished — it's for batch / run-once work. (Batch pods use `restartPolicy: OnFailure` or `Never`, never `Always`.)

### {L4.19} CronJob — schedule, suspend, list spawned Jobs
**Q:** Create a CronJob that prints the date every minute; show how to suspend it and list the Jobs it spawns.

**DO**
```bash
kubectl create cronjob datecj --image=busybox --schedule="*/1 * * * *" -- date
kubectl patch cronjob datecj -p '{"spec":{"suspend":true}}'    # stop creating new Jobs
```
**VERIFY** `kubectl get jobs` (one per minute while active); `kubectl get cronjob datecj` (SUSPEND column).
**WHY** A **CronJob** spawns a **Job** on each cron tick. `suspend: true` keeps the schedule defined but **stops creating new Jobs** (running ones finish). Standard 5-field cron: `min hr dom mon dow`.

### {L4.20} Manually trigger a Job from a CronJob
**Q:** Manually run a Job from an existing CronJob to test it on demand.

**DO** `kubectl create job manual-run --from=cronjob/datecj`
**VERIFY** `kubectl get jobs` (the manual Job appears immediately); `kubectl logs job/manual-run`
**WHY** `--from=cronjob/...` copies the CronJob's jobTemplate into a one-off Job so you can test it **without waiting for the schedule**.

### {L4.21} Ordered StatefulSet scaling vs Deployment
**Q:** Show a StatefulSet creates/terminates pods in order; contrast with a Deployment.

**DO**
```bash
kubectl scale statefulset redis --replicas=3 ; kubectl get pods -l app=redis -w   # up: 0 → 1 → 2
kubectl scale statefulset redis --replicas=0 ; kubectl get pods -l app=redis -w   # down: 2 → 1 → 0
# contrast:
kubectl scale deployment web --replicas=5    ; kubectl get pods -l app=web -w      # all at once, random names
```
**WHY** A StatefulSet **creates pods in ascending order and deletes in descending order**, waiting for each to be Ready/terminated before the next — so clustered stateful apps initialize and shut down predictably. A Deployment changes pods **in parallel with random name suffixes** because order doesn't matter for stateless apps.

## Part 4 — Storage & volumes

### {L4.23} List StorageClasses, PVs, PVCs
**Q:** List the StorageClasses, PVs and PVCs.

**DO** `kubectl get sc,pv,pvc` (add `-A` for all namespaces)
**WHY** **StorageClass** = the provisioner/profile (standard, fast-nvme). **PV** = the actual provisioned disk (cluster-scoped). **PVC** = a namespaced *request* that binds to a PV. A PVC stuck `Pending` usually means no matching StorageClass/PV.

### {L4.24} Mount a ConfigMap as a volume (key → file)
**Q:** Mount a ConfigMap as a volume so each key becomes a file; verify the contents inside the pod.

**DO**
```bash
kubectl create configmap app-config \
  --from-literal=app.properties=$'environment=production\ncache_ttl=3600'
```
```yaml
spec:
  containers:
    - name: web
      image: nginx
      volumeMounts: [{ name: cfg, mountPath: /etc/app }]
  volumes: [{ name: cfg, configMap: { name: app-config } }]
```
**VERIFY** `kubectl exec <pod> -- ls /etc/app` → `app.properties`; `kubectl exec <pod> -- cat /etc/app/app.properties`
**WHY** Mounting a ConfigMap as a volume turns **each key into a file** (filename = key, contents = value) under the mount path. Volume-mounted ConfigMaps **auto-update** in the pod when the ConfigMap changes (unlike env vars and `subPath`).

### {L4.25} subPath mount of a single key
**Q:** Mount a single ConfigMap key to a specific path using `subPath`; explain when it's needed and its update caveat.

**DO**
```yaml
volumeMounts:
  - name: cfg
    mountPath: /etc/app/special.txt    # a single FILE, not a directory
    subPath: special_key.txt           # only this key
volumes: [{ name: cfg, configMap: { name: app-config } }]
```
**WHY** `subPath` mounts **one file into an existing directory without hiding the directory's other files** (a normal volume mount would mask everything else under the mount path). **Caveat:** `subPath` mounts **do NOT auto-update** when the ConfigMap changes — you must restart the pod.

### {L4.26} Mount a Secret as a volume (decoded, restrictive perms)
**Q:** Mount a Secret as a volume; verify files contain decoded values with restrictive permissions.

**DO**
```bash
kubectl create secret generic vault --from-literal=DB_PASSWORD='S3cret!'
```
```yaml
volumeMounts: [{ name: sec, mountPath: /etc/secret, readOnly: true }]
volumes: [{ name: sec, secret: { secretName: vault, defaultMode: 0400 } }]
```
**VERIFY** `kubectl exec <pod> -- cat /etc/secret/DB_PASSWORD` → `S3cret!` (decoded); `ls -l` shows `0400`.
**WHY** Secret volumes write each key as a file containing the **decoded** value (you never see base64 inside the pod), are backed by **tmpfs (RAM)**, and `defaultMode: 0400` makes them read-only to the owner only.

### {L4.27} StatefulSet volumeClaimTemplates → one PVC per replica
**Q:** Show scaling a StatefulSet creates one PVC per replica; explain what happens to those PVCs when the StatefulSet is deleted.

**DO** (add to the StatefulSet):
```yaml
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: 1Gi } }
```
**VERIFY** `kubectl get pvc` → `data-redis-0`, `data-redis-1`, `data-redis-2`
**WHY** `volumeClaimTemplates` creates a **dedicated PVC per pod ordinal**, so each replica keeps its own persistent disk across restarts/rescheduling. **Deleting the StatefulSet does NOT delete the PVCs** — they're retained on purpose to protect data; you must delete them manually.

### {L4.28} initContainer pre-populates a shared volume
**Q:** Use an initContainer to pre-populate data into a shared volume before the main container reads it.

**DO**
```yaml
spec:
  initContainers:
    - name: seed
      image: busybox
      command: ["sh","-c","echo seeded > /data/file"]
      volumeMounts: [{ name: shared, mountPath: /data }]
  containers:
    - name: app
      image: busybox
      command: ["sh","-c","cat /data/file; sleep 3600"]
      volumeMounts: [{ name: shared, mountPath: /data }]
  volumes: [{ name: shared, emptyDir: {} }]
```
**WHY** **initContainers run to completion, in order, before any app container starts.** They share the pod's volumes, so they're ideal for seeding data, running migrations, or waiting on a dependency before the main app boots.

### {L4.29} readOnly volume mount
**Q:** Set `readOnly: true` on a volume mount and prove writes are rejected.

**DO** `volumeMounts: [{ name: cfg, mountPath: /etc/app, readOnly: true }]`
**VERIFY** `kubectl exec <pod> -- sh -c 'echo x > /etc/app/test'` → `Read-only file system`
**WHY** `readOnly: true` mounts the volume read-only inside the container; any write fails at the filesystem level. Use it to enforce immutability of config/secret mounts.

### {L4.30} emptyDir sizeLimit
**Q:** Set a `sizeLimit` on an emptyDir and explain what happens if the container exceeds it.

**DO** `volumes: [{ name: scratch, emptyDir: { sizeLimit: 500Mi } }]`
**WHY** When usage of the emptyDir exceeds `sizeLimit`, the kubelet **evicts the pod** (it's over its ephemeral-storage budget). Bounds runaway logs/temp files from filling the node.
## Part 5 — Secrets & ConfigMaps

### {L4.31} Generic Secret from literals (base64, not encrypted)
**Q:** Create a generic Secret for a DB username/password from literals; show the stored values are base64-encoded, not encrypted.

**DO**
```bash
kubectl create secret generic dbcreds \
  --from-literal=username=admin --from-literal=password=redhat123
```
**VERIFY**
```bash
kubectl get secret dbcreds -o jsonpath='{.data.password}'; echo    # cmVkaGF0MTIz  (base64)
echo cmVkaGF0MTIz | base64 -d; echo                                 # redhat123  (trivially reversed)
```
**WHY** Secret `data` values are **base64-encoded, NOT encrypted** — anyone with read access decodes them instantly. base64 is for safe transport of binary, not for confidentiality. Real protection comes from **RBAC**, **encryption-at-rest in etcd**, and not committing secrets to Git.

### {L4.32} Secret from files (--from-file)
**Q:** Create a Secret from files holding a certificate and key; identify the resulting keys.

**DO** `kubectl create secret generic certs --from-file=tls.crt=./server.crt --from-file=tls.key=./server.key`
**VERIFY** `kubectl get secret certs -o jsonpath='{.data}'; echo` → keys `tls.crt`, `tls.key`
**WHY** With `--from-file`, the **key name defaults to the filename** (or you set it as `key=path`), and the value is the base64 of the file contents. One Secret can hold many file-keys.

### {L4.33} kubernetes.io/tls typed Secret
**Q:** Create a `kubernetes.io/tls` typed Secret from a cert/key pair and explain where it's consumed.

**DO** `kubectl create secret tls web-tls --cert=./server.crt --key=./server.key`
**VERIFY** `kubectl get secret web-tls -o jsonpath='{.type}'; echo` → `kubernetes.io/tls`
**WHY** A `tls` Secret has the fixed keys `tls.crt`/`tls.key` and is consumed by **Ingress/Route TLS termination** and by apps needing HTTPS certs. The typed form lets controllers validate the pair.

### {L4.34} Secret as volume vs env var (trade-offs)
**Q:** Mount a Secret as a volume; explain the security trade-offs versus env vars.

**DO** mount as in {L4.26}, or as env: `kubectl set env deployment/web --from=secret/dbcreds`
**WHY** **Volume** mounts are safer: backed by tmpfs, support file permissions, and **update live** when the Secret changes; they're not exposed by `printenv` or leaked into child-process environments or crash dumps. **Env vars** are simpler but visible in `kubectl describe`/`/proc/<pid>/environ` and don't refresh until pod restart. Prefer volumes for sensitive data.

### {L4.35} envFrom secretRef — load all keys at once
**Q:** Use `envFrom` with a `secretRef` to load every key of a Secret as env vars.

**DO**
```yaml
containers:
  - name: web
    image: nginx
    envFrom: [{ secretRef: { name: dbcreds } }]   # every key → an env var of the same name
```
**VERIFY** `kubectl exec <pod> -- printenv | grep -E 'username|password'`
**WHY** `envFrom` injects **all keys** of the Secret (or ConfigMap with `configMapRef`) as env vars in one line, instead of mapping each key individually with `valueFrom`. Use `--prefix=` (e.g. `MYSQL_`) to namespace them (see the MT2 lab).

### {L4.36} docker-registry Secret + imagePullSecrets
**Q:** Create a docker-registry (image-pull) Secret and reference it via `imagePullSecrets`; explain when it's required.

**DO** create as in {L4.2}, then:
```yaml
spec:
  template:
    spec:
      imagePullSecrets: [{ name: dockerhub }]
```
**WHY** Required whenever a pod pulls from a **private registry** (or to use authenticated DockerHub for higher rate limits). Without it the kubelet pulls anonymously and gets `ErrImagePull`/`401`. Attach to the ServiceAccount to cover all pods, or per-pod via `imagePullSecrets`.

### {L4.37} Selectively mount one Secret key
**Q:** Create a Secret with several keys and mount only one of them into a pod.

**DO**
```yaml
volumes:
  - name: sec
    secret:
      secretName: vault
      items: [{ key: api_token.key, path: token }]    # only this key, renamed to 'token'
volumeMounts: [{ name: sec, mountPath: /etc/secret, readOnly: true }]
```
**VERIFY** `kubectl exec <pod> -- ls /etc/secret` → only `token`
**WHY** The `items` list filters which keys are projected and lets you rename the resulting file `path`. Without `items`, **all** keys are mounted.

## Part 6 — Services & networking

### {L4.38} ClusterIP Service + DNS resolution
**Q:** Create a ClusterIP Service for a Deployment and resolve it by DNS from another pod.

**DO**
```bash
kubectl expose deployment web --port=80 --target-port=80    # default type = ClusterIP
kubectl run tmp --rm -it --image=busybox -- nslookup web
```
**VERIFY** `nslookup web` resolves to the Service's ClusterIP; `wget -qO- web:80` returns the page.
**WHY** A **ClusterIP** Service gets a stable virtual IP and a DNS name `web.<ns>.svc.cluster.local` (short `web` within the same namespace), load-balancing across the pods its selector matches. It's reachable **only inside the cluster**.

### {L4.39} NodePort Service
**Q:** Create a NodePort Service and reach it via `minikube service <svc> --url` and `$(minikube ip):<nodePort>`.

**DO**
```bash
kubectl expose deployment web --type=NodePort --port=80 --target-port=80
minikube service web --url            # prints the reachable URL
curl $(minikube ip):$(kubectl get svc web -o jsonpath='{.spec.ports[0].nodePort}')
```
**WHY** A **NodePort** opens the same high port (30000–32767) on **every node** and forwards it to the Service → pods. It's the simplest way to reach a service from outside a bare cluster without a load balancer.

### {L4.40} LoadBalancer Service on minikube
**Q:** Create a LoadBalancer Service, explain why EXTERNAL-IP is `<pending>` on minikube, and access it with `minikube tunnel`.

**DO**
```bash
kubectl expose deployment web --type=LoadBalancer --port=80 --target-port=80
# in a SEPARATE terminal (needs sudo):
minikube tunnel
# back in the first terminal:
kubectl get svc web        # EXTERNAL-IP now populated
```
**WHY** `LoadBalancer` asks the **cloud provider** for an external IP. minikube has no cloud, so EXTERNAL-IP stays `<pending>` until `minikube tunnel` simulates a load balancer by creating a route on your host.

### {L4.41} Headless Service for a StatefulSet
**Q:** Create a headless Service (`clusterIP: None`) and show it returns per-pod A records instead of one VIP.

**DO** see {L4.16}; then `kubectl run tmp --rm -it --image=busybox -- nslookup redis`
**VERIFY** returns A records for `redis-0/1/2` pod IPs — **not** a single VIP.
**WHY** Setting `clusterIP: None` makes the Service **headless**: DNS returns the **individual pod IPs** rather than load-balancing through one virtual IP, so clients (or clustered DBs) can address each pod directly.

### {L4.42} Endpoints / EndpointSlice
**Q:** Inspect a Service's Endpoints and explain how they're populated from the pod selector.

**DO** `kubectl get endpoints web` / `kubectl get endpointslices -l kubernetes.io/service-name=web`
**WHY** The endpoints controller watches pods matching the Service's **selector** and lists the IPs of those that are **Ready**. An **empty Endpoints list** = selector matches nothing, or no pod is Ready — the root cause of "Service returns nothing" (see {L5.25}).

### {L4.43} Cross-namespace access via FQDN
**Q:** Demonstrate cross-namespace access using the FQDN, and show the short name fails from another namespace.

**DO**
```bash
# from a pod in namespace A, reaching service 'web' in namespace B:
wget -qO- web.B.svc.cluster.local:80      # works (FQDN)
wget -qO- web:80                           # fails from namespace A (resolves only within same ns)
```
**WHY** The short name `web` resolves only **within the same namespace**. To reach a Service in another namespace use the FQDN `web.<namespace>.svc.cluster.local` — DNS search domains only cover the caller's own namespace.

### {L4.44} port-forward to a Service vs a Pod
**Q:** Compare `port-forward` to a Service vs to a Pod — the difference and when each fits.

**DO**
```bash
kubectl port-forward svc/web 8080:80      # forwards to ONE pod behind the service
kubectl port-forward pod/<pod> 8080:80    # forwards to that EXACT pod
```
**WHY** Both tunnel a local port into the cluster for debugging. `svc/...` picks **one** backing pod (no real load balancing); `pod/...` targets a **specific** pod — use it to inspect one instance (e.g. a crashing replica). Neither is for production traffic.

### {L4.45} port vs targetPort vs nodePort
**Q:** Explain the difference between a Service's `port`, `targetPort`, and `nodePort`.

**ANSWER**
- **`port`** — the port the **Service** listens on (what clients hit: `web:PORT`).
- **`targetPort`** — the **container** port traffic is forwarded to (what the pod listens on).
- **`nodePort`** — the port opened on **every node** (NodePort/LoadBalancer types only), range 30000–32767.

Flow: client → `nodePort` (external) → `port` (Service VIP) → `targetPort` (pod).

### {L4.46} Verify connectivity from a debug pod
**Q:** Verify connectivity to a ClusterIP Service from a throwaway debug pod with wget/nc.

**DO**
```bash
kubectl run tmp --rm -it --image=busybox -- sh
# inside the pod:
wget -qO- web:80
nc -zv web 80
exit
```
**WHY** A disposable in-cluster pod is the correct vantage point to test a **ClusterIP** Service (unreachable from your laptop). `--rm` auto-deletes it on exit.

### {L4.47} One Service load-balancing two Deployments
**Q:** Create two Deployments and one Service that load-balances across both via a shared label; prove requests hit both.

**DO**
```bash
kubectl create deployment v1 --image=hashicorp/http-echo -- /http-echo -text=from-v1
kubectl create deployment v2 --image=hashicorp/http-echo -- /http-echo -text=from-v2
kubectl label deployment v1 v2 tier=web --overwrite           # (label the pods)
# Service selects the SHARED label, not app=v1/app=v2:
kubectl create service clusterip web --tcp=80:5678 -o yaml --dry-run=client \
  | sed 's/app: web/tier: web/' | kubectl apply -f -
```
Then loop `wget -qO- web:80` from a debug pod → output mixes `from-v1` and `from-v2`.
**WHY** A Service routes to **every pod its selector matches**, regardless of which Deployment owns them. Give both Deployments' pods a common label (`tier=web`) and point the Service selector at it → traffic spreads across both (a poor-man's blue/green).

### {L4.48} Diagnose pod → Service unreachability (ordered)
**Q:** Systematically diagnose why a pod can't reach a Service: DNS → endpoints → selector → readiness → port.

**DO**
```bash
kubectl exec <client> -- nslookup web          # 1 DNS resolves?
kubectl get endpoints web                       # 2 endpoints populated?
kubectl get pods -l <svc-selector> --show-labels  # 3 selector matches pod labels?
kubectl get pods -l app=web                      # 4 pods Ready? (readiness probe)
kubectl get svc web -o jsonpath='{.spec.ports}'  # 5 correct port/targetPort?
```
**WHY** Walk the chain in order; the **first failing check is the broken link**: NXDOMAIN → DNS/typo; empty endpoints → selector/label mismatch or no Ready pod; wrong port → connection refused.

### {L4.49} Expose with NodePort and verify
**Q:** Expose a service with a NodePort and verify it works.

**DO**
```bash
kubectl expose deployment web --type=NodePort --port=80
kubectl get svc web                              # note the 80:3XXXX/TCP nodePort
curl $(minikube ip):<nodePort>                   # or: minikube service web --url
```
**WHY** Same as {L4.39}: NodePort is the simplest external reach on a bare cluster — confirm by curling `nodeIP:nodePort`.

## Part 7 — Health probes

### {L4.50} HTTP livenessProbe
**Q:** Add an HTTP livenessProbe hitting `/` on port 80 to an nginx Deployment and verify via `describe pod`.

**DO**
```yaml
livenessProbe:
  httpGet: { path: /, port: 80 }
  initialDelaySeconds: 5
  periodSeconds: 10
```
**VERIFY** `kubectl describe pod <pod>` → `Liveness: http-get ... delay=5s period=10s`
**WHY** A **liveness** probe detects a **hung/deadlocked** container that's still "running" but not serving. On failure the **kubelet restarts the container**. (It does NOT remove the pod from the Service — that's readiness.)

### {L4.51} TCP readinessProbe
**Q:** Add a `tcpSocket` readiness probe to a redis container on port 6379 and verify.

**DO**
```yaml
readinessProbe:
  tcpSocket: { port: 6379 }
  periodSeconds: 5
```
**VERIFY** `kubectl describe pod <pod>` shows the readiness probe; `kubectl get endpoints redis` only lists pods once the probe passes.
**WHY** A **readiness** probe decides whether a pod receives traffic. While it fails, the pod's IP is **pulled from the Service Endpoints** (but the container is **not** restarted) — so half-started pods don't get requests.

### {L4.52} startupProbe budget for slow boot
**Q:** Configure a startup probe with a `failureThreshold × periodSeconds` budget for a ~2-minute boot; justify the numbers.

**DO**
```yaml
startupProbe:
  httpGet: { path: /, port: 80 }
  failureThreshold: 12
  periodSeconds: 10        # 12 × 10s = 120s grace before liveness takes over
```
**WHY** A **startup** probe protects **slow-starting** apps: liveness/readiness are suspended until startup succeeds, so the app gets its full boot budget (`failureThreshold × periodSeconds = 120s`) without being killed prematurely. After it passes once, normal probes resume.

### {L4.53} Default behavior with no probes
**Q:** Explain the default behavior when no probes are defined.

**ANSWER** With **no probes**: **liveness** defaults to "the container's main process is alive" — Kubernetes only restarts it if the process **exits/crashes**, never for a hang. **Readiness** defaults to "Ready as soon as the container starts," so the pod receives traffic **immediately**, even before the app can actually serve — which can route requests into a not-yet-ready app. That's why explicit probes matter for real workloads.
# LO5 — Ship applications: find & fix the mistake (practical)

> The exam hands you something broken on purpose. Method is always the same: **reproduce → read the evidence (`logs` / `describe` / `inspect`) → name the root cause → fix → verify**. State the root cause out loud — the grader wants the diagnosis, not just a working result.

## Part 1 — Podman runtime mistakes

### {L5.1} Reversed port mapping
**Q:** `podman run -d --name web -p 80:8080 docker.io/library/nginx` — nginx listens on 80, but the site isn't reachable on the host port you expected. What's reversed?

**FIX**
```bash
podman run -d --name web -p 8080:80 docker.io/library/nginx   # -p HOST:CONTAINER
```
**WHY** `-p` is **`HOST:CONTAINER`**. `80:8080` maps host **80** → container **8080**, but nginx listens on **80** inside, so nothing answers. You wanted host 8080 → container 80. **Reversing `-p` is the #1 trap.**

### {L5.2} Missing required env var (MySQL init)
**Q:** `podman run -d --name db -e MYSQL_ROOT_PASSWORD docker.io/library/mysql` — the container exits during init.

**DO** `podman logs db` → *"Database is uninitialized and password option is not specified."*
**FIX** `podman run -d --name db -e MYSQL_ROOT_PASSWORD=redhat123 docker.io/library/mysql`
**WHY** `-e MYSQL_ROOT_PASSWORD` with **no value** passes an empty variable. MySQL's entrypoint refuses to initialize without a root password (or `MYSQL_ALLOW_EMPTY_PASSWORD=1`) and exits. Always give the variable a value.

### {L5.3} `-p` ignored under host networking
**Q:** `podman run -d --name app --network host -p 8080:80 docker.io/library/nginx` — why is `-p` effectively ignored?

**FIX**
```bash
# want port mapping? drop host networking:
podman run -d --name app -p 8080:80 docker.io/library/nginx
# or keep host net and hit the real port directly:
podman run -d --name app --network host docker.io/library/nginx ; curl localhost:80
```
**WHY** With `--network host` the container **shares the host's network namespace** — there's no separate namespace to map ports into, so `-p` is meaningless (Podman warns and ignores it). The app is reachable directly on the host port it binds.

### {L5.4} busybox exits immediately (Exited 0)
**Q:** `podman run -d --name c1 docker.io/library/busybox` — goes straight to `Exited (0)`. Why, and how to keep it running?

**FIX** `podman run -d --name c1 docker.io/library/busybox sleep infinity`
**WHY** A container runs **only as long as its main process (PID 1)**. busybox's default command exits immediately, so the container stops with code 0 (success, not error). Give it a long-running process (`sleep infinity`, a server) to keep it alive.

### {L5.5} `--rm` + `-d` then `logs` fails
**Q:** `podman run --rm -d --name job docker.io/library/alpine echo hello` — then `podman logs job` fails. Explain the gotcha.

**FIX**
```bash
# see output live without detaching:
podman run --rm --name job docker.io/library/alpine echo hello
```
**WHY** `--rm` **auto-removes the container the instant it exits**. `echo hello` finishes immediately, so by the time you run `logs job` the container (and its logs) are already gone. Drop `-d`, or drop `--rm` if you need to inspect afterward.

### {L5.6} Memory limit too low for MySQL
**Q:** `podman run -d -p 8080:80 --memory 8m docker.io/library/mysql` — the DB never becomes healthy. Which limit is the problem?

**FIX** `podman run -d --memory 512m -e MYSQL_ROOT_PASSWORD=redhat docker.io/library/mysql`
**WHY** `--memory 8m` caps the container at **8 MB**, far below MySQL's needs; it gets **OOM-killed during init** and never starts. Raise the memory limit (hundreds of MB). (Also note the `-p 8080:80` is wrong for MySQL's 3306, but the killer here is memory.)

### {L5.7} DNS by name fails on the default network
**Q:** Two alpine containers, `podman exec a ping b` — name resolution fails on the default network. Why, and how to fix?

**FIX**
```bash
podman network create appnet
podman run -d --name a --network appnet alpine sleep infinity
podman run -d --name b --network appnet alpine sleep infinity
podman exec a ping -c1 b      # now resolves
```
**WHY** Podman's **default** network has **no built-in DNS** for container-name resolution. Create a **user-defined network** — its embedded DNS lets containers resolve each other by name. (Same rule as the DO188 troubleshooting lab: a missing custom network breaks `host not found in upstream`.)

### {L5.8} Bind-mount missing `:Z` (SELinux denies access)
**Q:** `podman run -d --name web -v ./html:/usr/share/nginx/html docker.io/library/nginx` — files aren't visible / SELinux denies access. What flag is missing?

**FIX**
```bash
podman run -d --name web -v "$(pwd)/html:/usr/share/nginx/html:Z" docker.io/library/nginx
```
**WHY** On SELinux-enforcing hosts (RHEL), a bind-mounted host dir isn't labeled for container access, so the container is **denied**. The **`:Z`** suffix relabels the content with a private container label (`:z` for shared). Use an **absolute path** if the relative one isn't resolved.

### {L5.9} (Volume/permissions variants)
**Q:** A mounted volume's files are owned by the wrong UID so the container app can't write.
**FIX** match ownership: build the image with the right `USER`/`--chown`, or run with the user the volume expects; for named volumes let the image initialize them. **WHY** Rootless Podman maps container UIDs to subordinate host UIDs; a host-owned bind mount may be unreadable/unwritable to the in-container user. Align UID/GID or use `:Z` + correct `chown`.

## Part 2 — Containerfile / Dockerfile mistakes

### {L5.10} apt-get install without update
**Q:** `FROM debian:12` / `RUN apt-get install -y nginx` — the build can't find the package. What's missing?

**FIX**
```dockerfile
RUN apt-get update && apt-get install -y nginx
```
**WHY** A base image ships with an **empty/stale package index**. `apt-get install` has no package lists to resolve against until `apt-get update` populates them. Combine them in **one `RUN`** so the cache is consistent.

### {L5.11} CMD shell form can't stop cleanly
**Q:** `CMD python app.py` — the app can't be stopped cleanly. Shell vs exec form?

**FIX**
```dockerfile
CMD ["python", "app.py"]
```
**WHY** **Shell form** (`CMD python app.py`) runs the process as a child of `/bin/sh -c`, so **SIGTERM hits the shell, not python** → no graceful shutdown, slow forced kills. **Exec form** (JSON array) makes the app **PID 1**, so it receives signals directly.

### {L5.12} npm install cache busting (layer order)
**Q:** `FROM node:20` / `COPY . /app` / `WORKDIR /app` / `RUN npm install` — every source change re-runs a full `npm install`. Reorder for caching.

**FIX**
```dockerfile
WORKDIR /app
COPY package*.json ./        # copy ONLY manifests first
RUN npm install              # cached unless deps change
COPY . .                     # source last
```
**WHY** Docker caches layers by their inputs. `COPY . .` before install means **any** file change invalidates the `npm install` layer. Copy `package*.json` first so the install layer is reused until **dependencies** actually change.

### {L5.13} Overwriting PATH breaks the shell
**Q:** `FROM alpine:3.20` / `ENV PATH=/app/bin` / `RUN apk add --no-cache curl` — curl/sh aren't found. What broke?

**FIX**
```dockerfile
ENV PATH=/app/bin:$PATH      # PREPEND, don't replace
```
**WHY** Setting `ENV PATH=/app/bin` **replaces** the entire PATH, so `/bin`, `/usr/bin`, etc. vanish and the shell can't find `apk`, `curl`, or even `sh`. Always **append/prepend** to `$PATH`, never overwrite it.

### {L5.14/15} EXPOSE vs actual listen port mismatch
**Q:** `FROM ubuntu:24.04` / `EXPOSE 8080` / `CMD ["python3","-m","http.server","3000"]` — you map `-p 8080:8080` but nothing answers. What's the mismatch, and what does EXPOSE actually do?

**FIX**
```bash
# make the app listen on 8080:
CMD ["python3","-m","http.server","8080"]   # then -p 8080:8080
# (alternatively keep 3000 and map -p 8080:3000)
```
**WHY** The app listens on **3000**, but you published **8080:8080** → traffic hits a port nothing serves. **`EXPOSE` is documentation only** — it does **not** publish or open a port; only `-p`/`-P` at run time does. Match the published container port to the **actual listen port**.

### {L5.16} Image bloat — clean apt cache in the same layer
**Q:** `FROM debian:12` / `RUN apt-get update && apt-get install -y build-essential` — the image is huge. Cut the size in the same layer.

**FIX**
```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends build-essential \
 && rm -rf /var/lib/apt/lists/*
```
**WHY** Each `RUN` is a **layer**; deleting files in a *later* layer doesn't shrink earlier ones. Clean the apt lists **in the same `RUN`** (and use `--no-install-recommends`) so the cache is never committed to a layer.

### {L5.17} USER before COPY → permission denied
**Q:** `FROM python:3.11` / `USER appuser` / `COPY requirements.txt …` / `RUN pip install …` — permission denied. What's wrong with the ordering/ownership?

**FIX**
```dockerfile
WORKDIR /app
COPY --chown=appuser:appuser requirements.txt .
RUN pip install -r requirements.txt
USER appuser                 # switch user AFTER privileged steps
```
**WHY** After `USER appuser`, subsequent `COPY`/`RUN` run as that unprivileged user and **can't write** root-owned dirs. Do privileged installs first, use `COPY --chown` for ownership, and **drop to the non-root user last** (it's still good practice to run the *final* process as non-root).

### {L5.18} 1 GB Go image → multi-stage build
**Q:** `FROM golang:1.22 … RUN go build … CMD ["/src/app"]` — the final image is ~1 GB. How does multi-stage fix it?

**FIX**
```dockerfile
# build stage
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN go build -o /app

# runtime stage — tiny base, only the binary
FROM gcr.io/distroless/static
COPY --from=build /app /app
ENTRYPOINT ["/app"]
```
**WHY** The full Go toolchain (compiler, source, modules) bloats the image. A **multi-stage build** compiles in a fat builder stage, then `COPY --from=build` only the **final binary** into a minimal runtime base (distroless/alpine/scratch) → image drops from ~1 GB to tens of MB. (Same pattern as the DO188 Node/nginx and OpenJDK examples.)
## Part 3 — Kubernetes / OpenShift troubleshooting

> All commands work with `oc` or `kubectl`. The reflex for every one of these: **`describe` → read Events bottom-up**, then `logs` (add `--previous`/`-p` for a crashed instance).

### {L5.19} Pod stuck Pending
**Q:** A pod is stuck `Pending`. Diagnose with `describe pod` and identify the reason from the events.

**DO** `kubectl describe pod <pod>` → read Events.
**Likely causes:** insufficient CPU/memory on any node · unsatisfiable `nodeSelector`/affinity · a taint with no toleration · an unbound PVC.
**WHY** `Pending` means the scheduler **can't place the pod**. The Events line ("0/N nodes are available: …") names exactly which constraint failed. Fix the constraint (add capacity, label a node, create the PVC) and it schedules.

### {L5.20} YAML indentation broke the manifest
**Q:** Remove a couple of spaces from a deployment and try to deploy it. Identify the cause from the error and fix it.

**DO** `kubectl apply -f bad.yaml`
```
error: error converting YAML to JSON: did not find expected key
# or: mapping values are not allowed in this context
```
**FIX** Restore correct **2-space** indentation (YAML forbids tabs; nesting is significant).
**WHY** YAML is whitespace-structured: a missing/extra space reparents or orphans a key. The "did not find expected key" / "mapping values not allowed" errors are classic indentation faults. Validate with `--dry-run=client` before applying.

### {L5.21} ImagePullBackOff
**Q:** A pod is in `ImagePullBackOff`. Diagnose and fix.

**DO** `kubectl describe pod <pod>` → "Failed to pull image … not found / 401 unauthorized".
**FIX** Correct the image name/tag, **or** add an `imagePullSecrets` for a private registry (see {L4.36}).
**WHY** `ImagePullBackOff` = the kubelet can't pull the image: a **typo/bad tag**, or a **private registry** with no credentials. The describe Events distinguish "not found" (typo) from "unauthorized" (auth).

### {L5.22} CrashLoopBackOff
**Q:** A pod is in `CrashLoopBackOff`. Use `logs --previous` to read the last crash and fix the root cause.

**DO**
```bash
kubectl logs <pod> --previous     # oc logs <pod> -p
kubectl describe pod <pod>
```
**WHY** `CrashLoopBackOff` = the container **starts then exits repeatedly**, so the kubelet backs off restarting it. The **previous** instance's logs (current one may be too young) reveal the real error — a config bug, missing dependency, bad command. Fix that, not the symptom.

### {L5.23} OOMKilled
**Q:** A pod's last state shows `OOMKilled`. Confirm it, then fix it.

**DO**
```bash
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'; echo   # OOMKilled
```
**FIX** Raise `resources.limits.memory` (and/or fix the app's memory leak).
**WHY** `OOMKilled` = the container exceeded its **memory limit** and the kernel killed it. Confirm via `lastState.reason`, then either raise the limit or reduce the app's footprint.

### {L5.24} Pods never become Ready (failing readiness probe)
**Q:** A Deployment's pods never become Ready. Trace it to a failing readiness probe and correct it.

**DO** `kubectl describe pod <pod>` → "Readiness probe failed: …".
**FIX** Correct the probe's `path`/`port`/scheme, or fix the app so the endpoint responds; loosen `initialDelaySeconds` if it's just slow to boot.
**WHY** A failing **readiness** probe keeps the pod out of the Service Endpoints (no traffic) but does **not** restart it — so it sits Ready 0/1 forever. The probe is usually pointing at the wrong path/port.

### {L5.25} Service returns nothing — label mismatch
**Q:** A Service returns nothing. Diagnose a Service/pod label mismatch using `get endpoints`.

**DO** `kubectl get endpoints <svc>` → **empty**.
**FIX** Make the Service `spec.selector` match the pods' labels (compare `kubectl get pods --show-labels` with `kubectl get svc <svc> -o jsonpath='{.spec.selector}'`).
**WHY** **Empty Endpoints = the Service selector matches no Ready pod.** A Service is just a label selector over pods; if the labels don't line up, there are no backends and requests get nothing. (The single most common Service bug.)

### {L5.26} selector ≠ template labels (invalid Deployment)
**Q:** `spec.selector.matchLabels` doesn't match `spec.template.metadata.labels` — explain the error and fix it.

**DO** `kubectl apply -f bad.yaml`
```
error: Deployment.apps "x" is invalid: spec.template.metadata.labels:
       Invalid value ...: `selector` does not match template `labels`
```
**FIX** Make the two label sets **identical**.
**WHY** A Deployment's selector must match the labels it stamps on pods — otherwise it would create pods it can't manage. Kubernetes rejects the object at validation time. (selector is also **immutable** after creation.)

### {L5.27} Requests exceed node capacity
**Q:** `resources.requests` exceed any node's capacity — explain why the pod is Pending and fix it.

**DO** `kubectl describe pod <pod>` → "0/N nodes available: Insufficient cpu/memory".
**FIX** Lower the requests, or add a bigger node.
**WHY** The scheduler needs a node with **at least** the requested CPU/memory free. If no node can satisfy the request, the pod stays **Pending** — requests are a guaranteed reservation, not a best-effort hint.

### {L5.28} PVC doesn't exist (FailedMount)
**Q:** A pod mounts a PVC that doesn't exist — diagnose the FailedMount/Pending and create the PVC.

**DO** `kubectl describe pod <pod>` → "persistentvolumeclaim "x" not found".
**FIX**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: x }
spec:
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: 1Gi } }
  storageClassName: standard      # match a real StorageClass (lvms-vg1 on the lab cluster)
```
**WHY** A pod referencing a missing PVC can't mount its volume, so it never starts (Pending/FailedMount). Create the PVC (with a valid `storageClassName`) and the pod proceeds once it Binds.

### {L5.29} Container with no long-running command
**Q:** An ubuntu container with no long-running command shows `Completed`/`CrashLoopBackOff`. Explain and add a proper command.

**FIX**
```yaml
command: ["sleep", "infinity"]    # or run a real server
```
**WHY** Same root cause as {L5.4} but inside Kubernetes: the container's command **exits immediately**, so the pod goes `Completed`; with `restartPolicy: Always` (the Deployment default) the kubelet keeps restarting it → `CrashLoopBackOff`. Give it a process that stays running.

### {L5.30} Triage everything that happened to a pod
**Q:** Use `get events --sort-by=.lastTimestamp` to triage everything that happened to a pod over time.

**DO**
```bash
kubectl get events --sort-by=.lastTimestamp
kubectl get events --field-selector involvedObject.name=<pod> --sort-by=.lastTimestamp
```
**WHY** Events are the cluster's timeline (scheduling, pulls, probe failures, OOM, evictions). Sorting by `lastTimestamp` shows the **sequence** so you can see cause→effect (e.g. FailedScheduling → Scheduled → Pulling → Failed).

### {L5.31} Debug DNS from inside a pod
**Q:** Use `exec -it` to get a shell and debug DNS with `nslookup` and `/etc/resolv.conf`.

**DO**
```bash
kubectl exec -it <pod> -- sh          # oc rsh <pod>
nslookup kubernetes.default
cat /etc/resolv.conf                   # should point at the cluster DNS (CoreDNS) + search domains
```
**WHY** From inside the pod you see exactly what the app sees: whether **CoreDNS** resolves service names and whether `resolv.conf` has the right nameserver and `*.svc.cluster.local` search domains. NXDOMAIN here = DNS/networking issue, not an app bug.

### {L5.32} Ephemeral debug container (no-shell/distroless)
**Q:** Use an ephemeral debug container to troubleshoot a no-shell/distroless container.

**DO**
```bash
kubectl debug -it <pod> --image=busybox --target=<container> -- sh
# OpenShift: oc debug deploy/web   (debug copy of the workload)
```
**WHY** Distroless/scratch images have **no shell** to `exec` into. `kubectl debug` attaches an **ephemeral container** that shares the target's process/network namespaces, giving you tools (`sh`, `nc`, `ps`) **without modifying or restarting** the pod.

### {L5.33} Throwaway client pod for connectivity
**Q:** Spin up a `kubectl run tmp` pod to test connectivity to a Service from inside the cluster.

**DO**
```bash
kubectl run tmp --rm -it --image=busybox -- sh
wget -qO- <svc>:80 ; nc -zv <svc> 80 ; exit
```
**WHY** A disposable in-cluster pod is the right place to verify Service reachability and DNS (a ClusterIP isn't reachable from your laptop). `--rm -it` deletes it on exit.

### {L5.34} Node NotReady
**Q:** A node shows `NotReady`. List what you'd check and inspect on minikube.

**DO**
```bash
kubectl get nodes
kubectl describe node <node> | grep -A8 Conditions     # Ready / MemoryPressure / DiskPressure
minikube ssh -- 'systemctl status kubelet'             # on OCP: chroot /host; systemctl status kubelet
```
**WHY** `NotReady` usually means the **kubelet is down**, **disk/memory pressure**, or a broken **CNI/network plugin**. Node Conditions and kubelet logs point at which. (On the OCP lab: node terminal → `chroot /host` → `systemctl status kubelet`/`crio`.)

### {L5.35} Pod stuck Terminating — finalizers & grace
**Q:** A pod is stuck `Terminating`. Explain finalizers and the grace period, then force-delete and state the risks.

**DO**
```bash
kubectl get pod <pod> -o jsonpath='{.metadata.finalizers}'; echo
kubectl delete pod <pod> --grace-period=0 --force
```
**WHY** **Finalizers** block deletion until their controller finishes cleanup; the **grace period** is the SIGTERM→SIGKILL window. Force-delete removes the API object immediately **but the container may still run on the node** → orphaned process and, for stateful apps, **data corruption / split-brain**. Last resort only.

### {L5.36} Wrong apiVersion/kind pairing
**Q:** Wrong `apiVersion`/`kind` pairing (e.g. `apps/v1` + `Pod`) — explain the validation error and correct it.

**DO** `kubectl apply -f bad.yaml` → `error: no matches for kind "Pod" in version "apps/v1"`
**FIX** A **Pod** is `apiVersion: v1`; **Deployment/ReplicaSet/StatefulSet/DaemonSet** are `apps/v1`; **Job/CronJob** are `batch/v1`.
**WHY** Each kind lives in a specific API group/version. Mismatch them and the API server can't find the kind. `kubectl explain <kind>` shows the correct `apiVersion`.

### {L5.37} CreateContainerConfigError (missing ConfigMap key)
**Q:** A pod fails with `CreateContainerConfigError` because a `configMapKeyRef` key is missing. Diagnose and fix.

**DO** `kubectl describe pod <pod>` → "couldn't find key <X> in ConfigMap <cm>".
**FIX** Add the key (`kubectl edit cm <cm>`) or correct the `configMapKeyRef.key` in the pod spec.
**WHY** The container can't be configured because it references a ConfigMap/Secret key that doesn't exist → it never starts. The describe Events name the exact missing key.

### {L5.38} Secret in a different namespace
**Q:** A pod can't mount a Secret that lives in a different namespace. Explain namespace scoping and fix it.

**DO / FIX**
```bash
kubectl get secret <s> -n <src-ns> -o yaml \
  | sed 's/namespace: <src-ns>/namespace: <pod-ns>/' \
  | kubectl apply -n <pod-ns> -f -
```
**WHY** Secrets (and ConfigMaps, PVCs) are **namespace-scoped**; a pod can only reference objects in its **own** namespace. There is no cross-namespace mount — the Secret must be **copied into** the pod's namespace.

### {L5.39} ProgressDeadlineExceeded
**Q:** A rollout is stuck with `ProgressDeadlineExceeded`. Find the cause and remediate.

**DO**
```bash
kubectl rollout status deployment/web
kubectl describe deployment web | grep -A3 Conditions    # ProgressDeadlineExceeded
kubectl describe pod <new-pod>                            # the real reason
```
**FIX** Fix the root cause (bad image, failing probe, insufficient resources) — the rollout resumes automatically — or `kubectl rollout undo deployment/web`.
**WHY** The new ReplicaSet didn't reach availability within `progressDeadlineSeconds` (default 600s). The Deployment surfaces this condition, but the **failing new pod** holds the actual reason.

### {L5.40} Catch field typos with dry-run / validate
**Q:** Catch indentation/field typos before applying with `--dry-run=server` / `--validate=true`, then fix them.

**DO** `kubectl apply -f f.yaml --dry-run=server` → `error: unknown field "spec.template.spec.containers[0].imagePullpolicy"`
**FIX** Correct the field (`imagePullPolicy`, camelCase) / nesting, then apply.
**WHY** **Server-side dry-run validates against the real schema** and rejects unknown fields and mis-nesting **before** anything is created — far safer than discovering it after a partial apply.

### {L5.41} App can't reach its DB Service (ordered)
**Q:** An app can't reach its database Service. Verify in order and identify the broken link.

**DO (in order)**
```bash
kubectl get pod <app>                                  # 1 Running?
kubectl get svc db                                     # 2 Service exists?
kubectl get endpoints db                               # 3 endpoints populated?
kubectl exec <app> -- nslookup db                      # 4 DNS resolves?
kubectl get svc db -o jsonpath='{.spec.ports}'; echo   # 5 correct port?
```
**WHY** Walk pod → Service → endpoints → DNS → port; the **first failing check is the broken link** (empty endpoints → selector/label mismatch; NXDOMAIN → DNS; wrong port → connection refused).

### {L5.42} Read state/lastState/restartCount
**Q:** Read a container's `state`/`lastState` and `restartCount` with jsonpath to find the cause of repeated restarts.

**DO**
```bash
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].restartCount}'; echo
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'; echo
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'; echo
```
**WHY** A high `restartCount` plus `lastState.terminated.reason` (e.g. **OOMKilled**, **Error**) and the **exit code** pinpoint *why* a container keeps restarting — without scrolling through describe.

### {L5.43} OpenShift Route vs Ingress
**Q:** Compare OpenShift **Route** to **Ingress**.

**ANSWER** Both expose HTTP(S) services outside the cluster. **Ingress** is the Kubernetes-native, portable resource — you write Ingress rules, but you must install a separate **ingress controller** (nginx, Traefik, HAProxy) for them to do anything. **Route** is OpenShift's built-in object, backed by the platform's integrated **HAProxy router** — no controller to install, it auto-generates a hostname, and offers simple TLS modes (**edge / passthrough / re-encrypt**) out of the box. OpenShift supports both and reconciles Ingress objects into Routes. In short: **Route** = simpler, native, non-portable; **Ingress** = standard, portable, needs a controller.
# LO6 — Evaluate orchestration systems (theory)

> These are written answers. The grader wants a **clear position** plus **trade-offs that justify it**. Each item below gives you a one-line **Position** to commit to memory, then the **Why** to write out. Pattern for any "would you choose X?" question: *state the verdict → orchestration → resilience → ecosystem/cost → name the caveat.*

## Core comparisons

### {L6.1} Kubernetes vs Docker networking
**Position:** Docker = simple, host-local with NAT; Kubernetes = flat, cluster-wide, every pod routable.
**Why:** Docker's default is host-centric — a **bridge** network with **NAT**, private container IPs behind the host, manual `-p` port publishing, and name resolution only on user-defined networks. Kubernetes mandates a **flat network**: every pod gets its own IP and **any pod reaches any pod across nodes without NAT**, implemented by **CNI plugins** (Calico, Flannel, Cilium). On top, **Services** give stable virtual IPs + load balancing, and **CoreDNS** gives discovery by name. Docker networking is local and simple; Kubernetes is built for multi-node scale, discovery, and self-healing.

### {L6.2} K8s storage abstraction vs Podman storage
**Position:** Kubernetes is far more flexible for stateful workloads.
**Why:** Kubernetes abstracts storage via **CSI drivers**, **StorageClasses**, and **dynamic provisioning** — a pod's **PVC** requests storage and a PV is provisioned on demand from any backend (cloud disk, NFS, Ceph), and the volume **follows the pod** when rescheduled. Podman volumes/plugins are essentially **host-local**. For databases and other stateful services, Kubernetes provides portable, dynamic, multi-node, lifecycle-managed storage with access modes and reclaim policies; Podman storage is tied to one host.

### {L6.3} Operator pattern (CRDs + controllers) vs plain manifests
**Position:** Operators trade upfront effort for automated day-2 control; worth it for complex stateful services.
**Why:** Plain manifests describe desired state but leave **day-2 ops** (backups, failover, safe upgrades, scaling a clustered DB) to humans/scripts. An **Operator** encodes that operational knowledge into a **controller** that continuously reconciles a **CRD**, automating those tasks. Cost: building/adopting and maintaining the controller (learning curve). Benefit: far more automated control and much less operational toil. Manifests suffice for simple/stateless apps; operators pay off for production databases.

### {L6.4} Plain K8s (kubectl + manifests) vs OpenShift S2I + console
**Position:** kubectl+manifests optimize for control; S2I+console optimize for developer velocity.
**Why:** Plain Kubernetes means authoring Dockerfiles + YAML and managing everything explicitly — powerful, but steep and boilerplate-heavy. OpenShift's **S2I** + **developer console** let a developer point at source code and get an image **built and deployed without writing a Dockerfile or manifests**, through a guided UI. Maximum control vs speed, guardrails, and gentler onboarding.

### {L6.5} Migrating a workload OpenShift → Kubernetes
**Position:** Substantial effort; worth it only if portability/cost outweigh OpenShift's built-ins.
**Why:** You must rebuild OpenShift's integrated extras: **Routes → Ingress + controller**, **DeploymentConfig → Deployment**, **S2I/BuildConfig → external CI**, lose the **integrated registry**, reproduce stricter **SCC** defaults with PodSecurity/admission, and stand up your own **monitoring/logging/console**. Upside: **portability, no vendor coupling, lower licensing**. It's a trade of integrated, supported tooling for flexibility and self-managed effort.

### {L6.6} CI/CD build agents in Kubernetes vs dedicated VMs
**Position:** Kubernetes wins on elasticity/cost; VMs win on hard isolation.
**Why:** Agents **in Kubernetes** (Jenkins agents, GitLab runners, Tekton) are ephemeral and **autoscaling** — a pod per build, gone after — with good bin-packing and low idle cost. Downsides: weaker **isolation** (shared node kernel) and **noisy-neighbor** risk, mitigated with requests/limits, quotas, node pools. **Dedicated VMs** give strong isolation and predictable performance but waste idle resources and scale slowly. Choose VMs for untrusted/security-sensitive builds; Kubernetes otherwise.

### {L6.7} K8s integration with cloud + dynamic capacity
**Position:** Deep — cloud-controller-manager + autoscalers grow/shrink infra on demand.
**Why:** The **cloud-controller-manager** provisions cloud **LoadBalancers** and attaches cloud **block storage** to PVCs via CSI. Scaling happens at two levels: the **Horizontal Pod Autoscaler** (and VPA) scales workloads by metrics; the **Cluster Autoscaler** (or **Karpenter** on AWS) adds/removes **nodes** based on unschedulable-pod pressure. With managed services (EKS/GKE/AKS), capacity matches load automatically.

### {L6.8} OpenShift vs vanilla K8s security/hardening
**Position:** OpenShift is secure-by-default; regulated industries pick it for a certified, audited baseline.
**Why:** OpenShift ships **Security Context Constraints (SCC)** that force **non-root** unless explicitly granted, built-in **OAuth/RBAC**, an integrated registry, image provenance/scanning, and Red Hat-signed supported images. Vanilla Kubernetes is **permissive by default** — you add PodSecurity admission, RBAC, NetworkPolicies, scanning yourself, and mistakes are easy. Finance/healthcare/government choose OpenShift for a **hardened, vendor-supported baseline** with a ready compliance/audit story.

### {L6.9} Learning curve & skill set — OpenShift vs plain K8s
**Position:** OpenShift helps newcomers early, adds complexity to master later.
**Why:** Plain Kubernetes is already **steep** (manifests, networking, RBAC, controllers, storage). OpenShift layers more concepts — **SCC, Routes, BuildConfig/ImageStream, `oc`** — so there's *more* to learn for deep ops. But its **console + S2I lower the barrier** for the common case: build/deploy without mastering Dockerfiles/YAML. Net: helpful for initial productivity, with OpenShift-specific machinery to learn for deep debugging/operations.

### {L6.10} Full Kubernetes vs k3s footprint (small on-prem)
**Position:** For small/edge/single-team, full Kubernetes is overkill; k3s gives the same API at a fraction of the cost.
**Why:** Full Kubernetes runs a heavyweight control plane (**etcd + multiple components**), needs more CPU/RAM and ongoing care, built for large-scale HA. **k3s** packs it into a single <100 MB binary, can use **SQLite** instead of etcd, strips legacy cloud drivers, runs on a Raspberry Pi — exposing the **same kubectl/API**. Reach for full Kubernetes only when you need large-scale HA, many nodes, or deep cloud integration.

### {L6.11} Docker Compose (single host) vs Kubernetes — small team
**Position:** Compose is the defensible choice until you genuinely need HA / elastic scaling / multi-host resilience.
**Why:** **Compose on one host** is trivial to learn/operate, fine for dev and small stable single-host prod — but **no multi-node scheduling, no cross-host self-healing, no autoscaling**, and a single point of failure. **Kubernetes** brings resilience, horizontal scaling, rolling updates, at significant complexity. For modest, predictable load with no HA requirement, Compose ships fastest; move to Kubernetes when its limits become the bottleneck.

### {L6.12} Vendor lock-in — Kubernetes (CNCF) vs OpenShift (Red Hat)
**Position:** Kubernetes = low lock-in/high portability; OpenShift = convenience + support at the price of some lock-in.
**Why:** Vanilla Kubernetes is **CNCF-governed and open**, portable across clouds/distros — lock-in is **low**. **OpenShift** adds value (support, integrated tooling, hardening) but also **Red-Hat-specific constructs** (Routes, DeploymentConfig, SCC, subscriptions) and licensing cost → **some lock-in**. OpenShift buys convenience and vendor accountability now; plain Kubernetes preserves flexibility but you assemble and support the platform yourself.

## "Would you choose…" recommendation questions

### {L6.13} Recommend OpenShift for an integrated, supported platform
**Position:** Yes — it's exactly the turnkey, accountable platform such a company is asking for.
**Why:** One vendor-supported product with **built-in CI/CD (S2I/Tekton)**, integrated **registry**, **monitoring/logging**, a **developer console**, hardened **secure-by-default** settings, and **enterprise SLAs**. Far less integration work than assembling open-source pieces, faster onboarding, a single accountable vendor for support and patches. The accepted trade-offs are **licensing cost and some lock-in** — which is what this company chose to pay for turnkey tooling.

### {L6.14} K8s vs VM storage provisioning
**Position:** Kubernetes = declarative, dynamic, self-service; VMs = manual, static, admin-driven.
**Why:** With **VMs**, an admin manually attaches disks/LUNs tied to the VM's lifecycle. **Kubernetes** makes storage **declarative**: a **PVC** requests capacity from a **StorageClass**, **CSI** provisions a PV automatically, and the volume **follows the pod** on reschedule. Kubernetes decouples the storage *request* from *provisioning* and adds self-service, portability, and automated lifecycle.

### {L6.15} Startup with two engineers, ship fast & cheap
**Position:** Managed PaaS / managed container service (or Compose/k3s self-hosted) — **not** self-managed full Kubernetes.
**Why:** With two engineers the priority is shipping, not running a control plane. A managed platform (Cloud Run, Render, Fly, App Runner, small managed cluster) removes cluster upgrades, patching, control-plane HA — you pay mainly for what you run and spend near-zero on ops. Self-managed Kubernetes imposes overhead that **dwarfs a two-person team**. Self-hosting? Compose or k3s on one host keeps cost/overhead minimal.

### {L6.16} Docker vs Podman — architecture & rootless security
**Position:** Security-conscious teams prefer Podman: daemonless + rootless shrinks the attack surface.
**Why:** Docker uses a central **daemon (dockerd) traditionally running as root** — a single point of failure and broad attack surface (the Docker socket ≈ root). **Podman is daemonless**: it forks containers directly (conmon/runc) with no privileged daemon, and runs **rootless by default**, mapping container root to an unprivileged host user. It's CLI-compatible with Docker and integrates with **systemd + SELinux**. No privileged background process to compromise = better isolation.

### {L6.17} Day-2 overhead — self-managed vs managed Kubernetes
**Position:** Managed offloads the control plane; usually the better choice barring strong reasons to self-manage.
**Why:** **Self-managed** puts all day-2 work on you: control-plane and node **upgrades**, **etcd backups**/restore testing, **node scaling**, OS/security **patching**, control-plane **HA** — specialized, ongoing effort. A **managed service** (EKS/GKE/AKS/ARO) runs/upgrades/keeps the control plane available and often automates node scaling/backups, leaving you your workloads and worker nodes. Trade some control + a fee for **dramatically lower overhead**.

### {L6.18} Media-streaming, spiky global traffic
**Position:** Managed Kubernetes with autoscaling, multi-region, fronted by a CDN.
**Why:** Spiky global traffic needs **elastic scaling**: **HPA** scales pods to demand, **Cluster Autoscaler/Karpenter** adds/removes nodes to control cost. Kubernetes' **self-healing** + rolling updates keep the service up during spikes and deploys; **multi-region** cuts latency and adds resilience; a **CDN** offloads heavy media delivery. A managed control plane keeps ops manageable while the platform absorbs unpredictable load.

### {L6.19} Outgrown Swarm/Compose — migrate to Kubernetes?
**Position:** Yes, if genuinely outgrown — the one-time migration pays off for lasting scaling/resilience needs.
**Why:** Effort is real: rewrite Compose/Swarm defs into manifests, learn the platform, rebuild CI/CD, networking, storage. Benefits are durable: huge **ecosystem** (Helm, operators, mesh, observability), **autoscaling**, **self-healing**, multi-cloud portability, the largest community/talent pool. Swarm is effectively **maintenance mode** and Compose can't scale across hosts — staying put caps growth. If needs are modest, it may not be worth it yet.

### {L6.20} Kubernetes for a microservices architecture?
**Position:** Yes — the de facto microservices platform.
**Why:** **Orchestration:** independent deploy/scale/rolling-update per service, with built-in **service discovery + load balancing**. **Resilience:** self-heals (restarts failed pods, reschedules off dead nodes), health probes, zero-downtime rollouts/rollbacks. **Ecosystem:** unmatched — Helm, **service meshes** (Istio/Linkerd) for mTLS/traffic, rich observability. Caveat: for a mere handful of services it can be overkill, but for real microservices the orchestration/resilience/ecosystem make it the strong choice.

### {L6.21} Kubernetes for enterprise-grade applications?
**Position:** Yes — scalable, community-backed, feature-rich; invest in ops skills.
**Why:** **Scalability:** HPA + cluster autoscaling to thousands of nodes/pods. **Community:** CNCF governance, the largest contributor/vendor ecosystem, deep docs and talent pool — de-risks long-term adoption. **Features:** RBAC, namespaces + quotas for multi-tenancy, rolling updates, autoscaling, extensibility via **CRDs/operators**; managed offerings add SLAs. A robust, future-proof default for enterprise workloads.

### {L6.22} OpenShift for a microservices architecture?
**Position:** Yes — especially for enterprises wanting microservices **plus** integrated tooling.
**Why:** OpenShift is Kubernetes underneath, inheriting the same **orchestration** and **resilience**. On top it adds a microservices-tuned **ecosystem**: built-in **CI/CD (Tekton/S2I)**, **OpenShift Service Mesh** (Istio), integrated registry, developer console — less DIY wiring. Trade-offs: licensing cost + some lock-in. Strong choice for orgs valuing an integrated, supported platform over DIY flexibility.

### {L6.23} OpenShift for enterprise-grade applications?
**Position:** Yes — when the enterprise values vendor support and a hardened, integrated platform.
**Why:** Kubernetes' **scalability** plus enterprise features: **secure-by-default SCC**, integrated **CI/CD, registry, monitoring/logging**, developer console, **Red Hat SLAs** and certified signed images. "Community support" comes twice — the broad Kubernetes/CNCF community **and** Red Hat's commercial support/lifecycle guarantees — exactly what regulated, support-dependent enterprises want. **Feature richness** cuts integration effort and strengthens compliance. Cost: licensing + some lock-in.

---

## LO6 EXTRA — Swarm, k3s, runtimes & broader talking points

**Docker Swarm (prof-flagged).** Docker's built-in orchestrator: `docker swarm init`, **services** (`docker service create`), **stacks** (deploy a Compose file), an **overlay network** for multi-host, **Raft** consensus among managers, built-in routing mesh + secrets. *Pros:* trivial setup, Compose-native, gentle curve, fine for small clusters. *Cons:* small ecosystem, limited autoscaling/extensibility, largely in **maintenance mode**. **vs Kubernetes:** Swarm = simplicity at small scale; Kubernetes = power, ecosystem, resilience at higher complexity.

**k3s (prof-flagged).** CNCF-certified lightweight Kubernetes (Rancher/SUSE) in a single <100 MB binary bundling API server, controller-manager, scheduler, kubelet, containerd. Defaults to **SQLite** (or embedded etcd for HA), has **server**/**agent** roles, bundles Traefik ingress, a service load balancer, local-path storage. **Same kubectl/API** as upstream. Ideal for **edge, IoT, dev, small on-prem**; peers are k0s and microk8s. Choose it when full Kubernetes is overkill; full Kubernetes when you need large-scale HA + deep cloud integration.

**Container runtimes & OCI.** Kubernetes talks to runtimes via the **CRI**; the **dockershim was removed in v1.24**, so clusters use **containerd** or **CRI-O** (OpenShift's default) directly, both built on **runc**. Docker images still run everywhere because they follow the **OCI** image/runtime specs — that standardization keeps images portable across Docker, Podman, containerd, CRI-O.

**Other ready talking points.**
- **Packaging:** **Helm** (templated charts) vs **Kustomize** (overlays) vs raw manifests.
- **GitOps:** **ArgoCD/Flux** reconcile the cluster to Git → auditability + easy rollback.
- **Service mesh:** mTLS + traffic control at many-service scale.
- **Autoscaling:** **HPA** (pods) vs **Cluster Autoscaler** (nodes).
- **State of truth:** **etcd** — must be backed up.
- **Security:** **RBAC**, **NetworkPolicy** (default-allow → zero-trust), **Pod Security Standards** (privileged/baseline/restricted; OpenShift uses **SCC**).
- **When NOT to use Kubernetes:** a single app, a tiny team, or stable load → Compose, a PaaS, or serverless is the better fit.
# APPENDIX

The practical labs below are the **actual DO180/DO188 graded exercises** from your course materials, rewritten as clean step-by-step walkthroughs. They're the best rehearsal for the LO4/LO5 practical — run each once so the commands are muscle memory.

---

## A · Podman command reference (DO188)

```bash
# Login
podman login registry.redhat.io
podman login -u developer -p developer registry.ocp4.example.com:8443
podman login -u $(oc whoami) -p $(oc whoami -t) default-route-openshift-image-registry.apps.ocp4.example.com
podman logout --all

# Images
podman pull <registry>/<image>:<tag>
podman images                                   # list local images
podman image tag LOCAL:TAG NEW:TAG
podman build -f Containerfile -t NAME:TAG .     # --squash-all to flatten layers
podman push NAME:TAG quay.io/USER/IMAGE:TAG
podman image inspect IMAGE --format '{{.Config.Cmd}}'
podman image rm IMAGE  [--all] [-f]
podman image prune -af                          # delete dangling + unused

# Run / lifecycle
podman run -d --name N -p HOST:CTR --net NET -e K=V -v HOSTDIR:CTRDIR:Z IMAGE [cmd]
podman ps [-a]                                  # -a includes stopped (find the dead container)
podman logs [-f] N
podman exec [-it] N <cmd>                        # --latest / -l = last-created container
podman cp [N:]src [N:]dst
podman inspect N --format '{{.State.Status}}'
podman stop|restart|kill|rm [-f] N              # rm --all --force = nuke everything
podman pause|unpause N
podman port N                                    # show port mappings

# Networking (custom net = DNS by name)
podman network create NET                        # -o isolate=true to block cross-network traffic
podman network ls | inspect NET | rm NET | prune
podman network connect|disconnect NET N

# Secrets
printf "R3d4ht123" | podman secret create dbsecret -
podman run --secret dbsecret,type=env,target=DB_SECRET --name app IMAGE   # as env var
podman secret rm NAME

# Volumes
podman volume create VOL
podman volume import VOL archive.tar.gz          # export VOL --output archive.tar.gz
podman run --mount type=volume,src=VOL,dst=/data IMAGE
podman run --mount type=tmpfs,tmpfs-size=512M,destination=/data IMAGE   # ephemeral, no COW

# Inspect ports actually in use inside a container
podman exec N ss -pant                            # p=process a=all n=numeric t=tcp

# Skopeo (registry ops without pulling)
skopeo inspect docker://<image>
skopeo copy docker://FROM docker://TO
skopeo list-tags docker://<image>

# Compose
podman-compose up -d        # --force-recreate, --remove-orphans
podman-compose down [-v]    # -v removes anonymous volumes
podman-compose stop
```

**Quadlet / systemd (run containers as services):**
```bash
systemctl --user daemon-reload
systemctl --user start|stop|status <name>.service
loginctl enable-linger        # keep user services running after logout
```

---

## B · `oc` / `kubectl` command reference (DO180)

```bash
# Login & projects
oc login -u developer -p developer https://api.ocp4.example.com:6443
oc whoami [--show-console] [-t]            # -t prints the token
oc new-project NAME            # = kubectl create namespace + switch
oc project NAME                # switch current project   (kubectl: config set-context --current --namespace=NAME)
oc status [--suggest]          # project overview + issue hints

# Discover the API (panic buttons)
oc api-resources               # every kind + short-name + API group
oc explain pod.spec.containers # field docs   (--recursive for all fields, no descriptions)
oc api-versions

# Get / inspect
oc get RESOURCE [NAME] [-o wide|yaml|json] [-n NS] [--show-labels] [-l k=v]
oc get all                     # pods, svc, deploy, rs, routes...
oc describe RESOURCE NAME       # events at the bottom
oc get events --sort-by=.metadata.creationTimestamp

# Create / run / deploy
oc create -f file.yaml | oc apply -f file.yaml
oc run NAME --image IMG [--restart Always|OnFailure|Never] [--env K=V] [-it] [--rm] [--command -- cmd args]
oc new-app IMAGE~GITURL         # build + deploy from source/image (OpenShift-only)
oc create deployment web --image=nginx --replicas=3 [--dry-run=client -o yaml]

# Operate
oc scale --replicas=5 deployment/web
oc set image deployment/web web=nginx:1.27
oc set env deployment/web --from=secret/dbparams --prefix=MYSQL_
oc set env deployment/web KEY=VALUE
oc set volume deployment/web --add --type secret --secret-name s --mount-path /app-secrets
oc set resources deployment/web --requests=cpu=100m,memory=128Mi --limits=memory=256Mi
oc rollout status|history|undo|pause|resume deployment/web

# Expose
oc expose deployment/web --port=80 --target-port=80      # → Service
oc expose svc/web [--hostname host.apps.ocp4.example.com] # → Route (public URL)
oc create ingress NAME --rule="HOST/*=svc:port"          # Ingress (vanilla-K8s style)
oc get routes

# Config & secrets
oc create cm my-config --from-literal k1=v1 --from-file k2=./file
oc create secret generic s --from-literal user=demo --from-literal password=zT1k
oc create secret generic ssh --from-file id_rsa=/path/id_rsa
oc create secret tls s-tls --cert /path.crt --key /path.key
oc create secret docker-registry reg --docker-server=... --docker-username=... --docker-password=...

# Debug & access
oc exec POD [-c CTR] -- CMD
oc rsh POD                      # interactive shell (= kubectl exec -it -- sh)
oc logs POD [-c CTR] [-p] [-f] [--tail=N] [--selector=k=v]
oc cp [POD:]src [POD:]dst
oc port-forward svc/web 8080:80
oc debug deploy/web | node/NODE

# Node / cluster ops (admin)
oc get nodes ; oc adm top nodes|pods [-A] [--containers]
oc debug node/NODE   →  chroot /host  →  systemctl status kubelet|crio
oc get clusteroperators
oc adm policy add-role-to-user ROLE USER
oc adm policy add-scc-to-user anyuid -z SERVICEACCOUNT     # allow root in a pod

# Node-local container tools (inside chroot /host)
crictl ps | pods | images | logs | inspect | exec
```

---

## C · Containerfile (Dockerfile) instruction reference

| Instruction | What it does | Exam gotcha |
|---|---|---|
| `FROM image[:tag] [AS name]` | Base image; start a build stage. | Name stages for multi-stage builds ({L5.18}). |
| `WORKDIR /path` | Set working dir for later instructions. | Created if missing; use it instead of `cd`. |
| `COPY src dst` | Copy build-context files in. | `COPY --chown=u:g` to fix ownership ({L5.17}). |
| `ADD src dst` | COPY **plus** URL fetch + tar auto-extract. | Prefer `COPY` for plain local files (clarity). |
| `RUN cmd` | Run a command, commit a new layer. | Combine `update && install && clean` in one `RUN` ({L5.10},{L5.16}). |
| `ENV K=V` | Runtime env var. | Don't overwrite `PATH` — prepend ({L5.13}). |
| `ARG K=V` | Build-time variable. | Not present at runtime unless copied into `ENV`. |
| `EXPOSE port` | **Documentation only** — does NOT publish. | Publishing is `-p` at run time ({L5.14/15}). |
| `USER name` | Switch user for later steps + final process. | Put privileged installs **before** it ({L5.17}). |
| `VOLUME /path` | Mark a path as a volume mount point. | Anonymous volume created on run. |
| `LABEL k="v"` | Image metadata. | e.g. `maintainer=`. |
| `ENTRYPOINT ["exe"]` | The executable; survives extra args. | Use **exec (JSON) form** for clean signals ({L5.11}). |
| `CMD ["arg"]` | Default args to ENTRYPOINT (or the command). | Overridden by run-time args; exec form preferred. |

**Multi-stage skeleton (the size fix):**
```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN go build -o /app

FROM gcr.io/distroless/static
COPY --from=build /app /app
ENTRYPOINT ["/app"]
```
## D · DO188 lab walkthroughs (step by step)

These are the graded Podman labs from the course, condensed to the exact commands. Registry is `registry.ocp4.example.com:8443`.

### D.1 — Podman basics (copy files, run a server, custom network)

**Goal:** copy a file out of a container, run an httpd server on a custom network, copy a file in.

```bash
# 1. Confirm the seed container is running
podman ps --format='{{.Names}}'                       # → basics-podman-secret

# 2. Verify the file exists inside it
podman exec basics-podman-secret ls /etc/secret-file  # → /etc/secret-file

# 3. Copy the secret file OUT to the local solution file
cd ~/DO188/labs/basics-podman
podman cp basics-podman-secret:/etc/secret-file solution

# 4. Create a custom network (gives name-based DNS)
podman network create lab-net

# 5. Run the web server: host 8080 → container 8080, on lab-net, detached
podman run -d --name basics-podman-server \
  --net lab-net -p 8080:8080 \
  registry.ocp4.example.com:8443/ubi9/httpd-24

# 6. Copy the site file INTO the running server
podman cp index.html basics-podman-server:/var/www/html/
# verify in a browser: http://localhost:8080  → "Hello from Podman Basics"

# 7. Run a second container (client) on the same network
podman run -d --name basics-podman-client \
  --net lab-net registry.ocp4.example.com:8443/ubi9/httpd-24
```
**Key points:** `podman cp` works both directions; a **custom network** is what enables container-name DNS; `-p HOST:CONTAINER`.

### D.2 — Build & push a container image

```bash
# 1. Authenticate to the registry
podman login -u developer -p developer registry.ocp4.example.com:8443

# 2. Build from the provided Containerfile, tagging for the registry
cd ~/DO188/labs/images-lab
podman build --file Containerfile \
  --tag registry.ocp4.example.com:8443/developer/images-lab .

# 3. Push it
podman push registry.ocp4.example.com:8443/developer/images-lab

# 4. Add a second tag and push that too
podman tag registry.ocp4.example.com:8443/developer/images-lab \
           registry.ocp4.example.com:8443/developer/images-lab:grue
podman push registry.ocp4.example.com:8443/developer/images-lab:grue

# 5. Run from the tagged image, map 8080, verify
podman run -d --name images-lab -p 8080:8080 images-lab:grue
curl localhost:8080
```
**Key points:** the **image name must include the registry host** to push there; tags are cheap labels on the same image.

### D.3 — Custom image build (multi-stage, ENV/WORKDIR/USER/ENTRYPOINT)

**Goal:** build a Node QR-code app image with a certificate-generation build stage, correct env, non-root user, and a default command. This is the canonical "build a proper Containerfile" exercise.

```dockerfile
# Stage 1: generate TLS certs in a throwaway builder
FROM registry.ocp4.example.com:8443/redhattraining/podman-certificate-generator AS certs
RUN ./gen_certificates.sh

# Stage 2: the runtime image
FROM registry.ocp4.example.com:8443/ubi9/nodejs-22:1
USER root
RUN groupadd -r student && useradd -r -m -g student student && \
    npm config set cache /tmp/.npm --global

COPY --from=certs --chown=student:student /app/*.pem /etc/pki/tls/private/certs/
COPY --chown=student:student . /app/

ENV TLS_PORT=8443 \
    HTTP_PORT=8080 \
    CERTS_PATH="/etc/pki/tls/private/certs"

WORKDIR /app
USER student                      # drop privileges AFTER root setup
RUN npm install --omit=dev        # production deps only
ENTRYPOINT npm start              # default command, not overridable by args
```
```bash
podman build -t localhost/podman-qr-app .
podman run --name custom-lab -p 8080:8080 -p 8443:8443 podman-qr-app
# → "TLS Server running on port 8443" / "Server running on port 8080"
```
**Key points:** `COPY --from=certs` pulls the build-stage output (multi-stage); `--chown` fixes ownership; set `USER` to non-root **after** the privileged steps; `--omit=dev` trims the image; `ENTRYPOINT` makes the default command sticky.

### D.4 — Persisting data (named volume + import, DB + frontend)

```bash
# 1. Create a named volume and import seed data into it
podman volume create postgres-vol
podman volume import postgres-vol ~/DO188/labs/persisting-lab/postgres-vol.tar.gz

# 2. Custom network for the two tiers
podman network create persisting-net

# 3. Database container — env vars + the volume mounted at the data dir
podman run --name persisting-db -d --net persisting-net \
  -e POSTGRESQL_USER=user -e POSTGRESQL_PASSWORD=pass -e POSTGRESQL_DATABASE=db \
  --mount='type=volume,src=postgres-vol,dst=/var/lib/pgsql/data' \
  registry.ocp4.example.com:8443/rhel9/postgresql-13:1

# 4. Frontend — host 3000 → container 8080, same network
podman run --name persisting-frontend -d --net persisting-net -p 3000:8080 \
  registry.ocp4.example.com:8443/redhattraining/podman-urlshortener-frontend
# verify: http://localhost:3000  and the imported short URL
```
**Key points:** **named volumes** persist data beyond the container; mounting at the DB's data path (`/var/lib/pgsql/data`) is what makes the data survive; `volume import` preloads a tarball.

### D.5 — Troubleshooting (the find-and-fix lab — pure LO5)

**Symptom:** `quotes-ui` container is `Exited`, the others (`quotes-api-v1/v2`) are up.

```bash
podman ps -a                                  # see the exited quotes-ui
podman logs quotes-ui
# → nginx: [emerg] host not found in upstream "quotes-api-v1"
```
**Diagnosis 1 — wrong network:** the proxy can't resolve the API hostnames.
```bash
podman inspect quotes-ui --format='{{.NetworkSettings.Networks}}'      # → pasta (default!)
podman inspect quotes-api-v1 --format='{{.NetworkSettings.Networks}}'  # → troubleshooting-lab
# fix: recreate quotes-ui ON the right network
podman rm quotes-ui
podman run -d --name quotes-ui -p 3000:8080 -e QUOTES_API_VERSION=v2 \
  --net troubleshooting-lab \
  registry.ocp4.example.com:8443/redhattraining/quotes-ui-versioning:1.0
```
**Diagnosis 2 — wrong upstream port:** v1 works, v2 returns `502 Bad Gateway`.
```bash
podman exec quotes-ui curl -s http://localhost:8080/api/v2/quotes   # 502
podman logs quotes-api-v2 | grep port                                # → 8081 (not 8080!)
podman exec quotes-ui cat /etc/nginx/nginx.conf                      # proxy_pass ...:8080  ← wrong
# fix: bind-mount the corrected nginx.conf (which points to :8081) with :Z
podman rm -f quotes-ui
podman run -d --name quotes-ui -p 3000:8080 -e QUOTES_API_VERSION=v2 \
  --net troubleshooting-lab \
  -v ~/DO188/labs/troubleshooting-lab/nginx.conf:/etc/nginx/nginx.conf:Z \
  registry.ocp4.example.com:8443/redhattraining/quotes-ui-versioning:1.0
# verify: http://localhost:3000
```
**Key points:** **read the logs first** (`host not found in upstream` = network/DNS; `502` = reached upstream, wrong port/no response). Custom network for name resolution; bind-mount with **`:Z`** to override a config file.

### D.6 — Compose (multi-service app with networks + env)

Final `compose.yaml` for the quotes app — built up by the lab one requirement at a time:

```yaml
name: compose-lab
services:
  wiremock:
    container_name: "quotes-provider"
    image: "registry.ocp4.example.com:8443/redhattraining/wiremock"
    volumes:
      - ~/DO188/labs/compose-lab/wiremock/stubs:/home/wiremock:Z   # mock config, :Z for SELinux
    networks:
      - backend-net
  quotes-api:
    container_name: "quotes-api"
    image: "registry.ocp4.example.com:8443/redhattraining/podman-quotesapi-compose"
    networks:
      - backend-net          # talks to provider
      - frontend-net         # talks to UI
    environment:
      QUOTES_SERVICE: "http://quotes-provider:8080"   # DNS name : port
  quotes-ui:
    container_name: "quotes-ui"
    image: "registry.ocp4.example.com:8443/redhattraining/podman-quotes-ui"
    networks:
      - frontend-net
    ports:
      - "3000:8080"          # host 3000 → container 8080

networks:
  backend-net: {}
  frontend-net: {}
```
```bash
cd ~/DO188/labs/compose-lab
podman-compose up -d
# apply changes by re-running:
podman-compose down && podman-compose up -d
# verify: http://localhost:3000
```
**Key points:** the two **networks isolate tiers** (provider not exposed to UI); `quotes-api` joins **both** networks to bridge them; service-to-service URLs use the **container name + internal port** (`quotes-provider:8080`); only the UI publishes a host port.
## E · Capstone integrated lab — 3-tier app on Kubernetes/OpenShift (MT2 CH8.2)

**Scenario:** deploy a "famous quotes" app — a MySQL database tier wired to a frontend web tier via a Secret, persistent storage, internal Services, and an external Route/Ingress. This single lab exercises Secrets, PVCs, Deployments, `set env --from`, Services, Ingress, and image rollout — i.e. most of LO4 at once.

```bash
# Phase 1 — project
oc new-project review                       # kubectl: create namespace review + switch
# (kubectl) kubectl create namespace review && kubectl config set-context --current --namespace=review

# Phase 2 — Database tier
# 1. Secret holding DB params
kubectl create secret generic dbparams \
  --from-literal=user=operator1 \
  --from-literal=password=redhat123 \
  --from-literal=database=quotesdb

# 2. PVC (no one-line command attaches storage to a Deployment, so declare it)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: mysql-pvc }
spec:
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: 2Gi } }
  storageClassName: lvms-vg1
EOF

# 3. DB Deployment at 0 replicas, pre-wired to the PVC
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: { name: quotesdb }
spec:
  replicas: 0
  selector: { matchLabels: { app: quotesdb } }
  template:
    metadata: { labels: { app: quotesdb } }
    spec:
      containers:
        - name: mysql8
          image: registry.ocp4.example.com:8443/rhel9/mysql-80:1-228
          volumeMounts:
            - { name: db-data, mountPath: /var/lib/mysql }
      volumes:
        - name: db-data
          persistentVolumeClaim: { claimName: mysql-pvc }
EOF

# 4. Inject the secret as env, prefixed MYSQL_  (→ MYSQL_USER, MYSQL_PASSWORD, MYSQL_DATABASE)
kubectl set env deployment/quotesdb --from=secret/dbparams --prefix=MYSQL_

# 5. Turn the DB on, expose it internally on 3306
kubectl scale deployment/quotesdb --replicas=1
kubectl expose deployment quotesdb --port=3306 --target-port=3306

# Phase 3 — Web UI tier
kubectl create deployment frontend \
  --image=registry.ocp4.example.com:8443/redhattraining/famous-quotes:2-42 --replicas=0

# inject DB secret with QUOTES_ prefix, then the hostname pointing at the DB Service
kubectl set env deployment/frontend --from=secret/dbparams --prefix=QUOTES_
kubectl set env deployment/frontend QUOTES_HOSTNAME=quotesdb     # = the Service name (DNS)

kubectl scale deployment/frontend --replicas=1
kubectl expose deployment frontend --port=8000 --target-port=8000

# external access: Ingress (vanilla K8s) — on OpenShift you'd use: oc expose svc/frontend
kubectl create ingress frontend \
  --rule="frontend-review.apps.ocp4.example.com/*=frontend:8000"

# Phase 4 — verify + test an image rollout
kubectl set image deployment/quotesdb mysql8=registry.ocp4.example.com:8443/rhel9/mysql-80:1-237
curl http://frontend-review.apps.ocp4.example.com
```

**What this lab teaches (and likely tests):**
- **`set env --from=secret/... --prefix=`** is the idiomatic way to feed a DB secret into both tiers without retyping values.
- The frontend reaches the DB by the **Service name** (`QUOTES_HOSTNAME=quotesdb`) — that's cluster DNS in action.
- Start Deployments at **`replicas: 0`**, wire everything, then **scale to 1** — a clean way to avoid pods crash-looping before their config exists.
- A **PVC** keeps MySQL data across pod restarts; `storageClassName` must match a real class (`lvms-vg1` on the lab cluster).
- `set image` performs a rolling image upgrade ({L4.4}).

---

## F · The "kitchen-sink" manifest (annotated reference)

One file demonstrating many LO4 objects together. Use it as a copy-paste pattern bank — each comment maps to a numbered task.

```yaml
# ── ConfigMap: each key becomes a file when mounted as a volume ({L4.24},{L4.25}) ──
apiVersion: v1
kind: ConfigMap
metadata: { name: app-config }
data:
  app.properties: |
    environment=production
    cache_ttl=3600
  special_key.txt: "Strictly isolated string data"     # targeted later via subPath
---
# ── Secret: stringData is auto-base64'd by the API server ({L4.26},{L4.31},{L4.35},{L4.37}) ──
apiVersion: v1
kind: Secret
metadata: { name: vault-secret }
type: Opaque
stringData:                       # plain text in; stored base64-encoded
  DB_USERNAME: "master_admin"
  DB_PASSWORD: "SuperComplexPassword123!"
  api_token.key: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
---
# ── Headless Service: clusterIP None → per-pod A records, no VIP ({L4.16},{L4.41}) ──
apiVersion: v1
kind: Service
metadata: { name: redis-mesh }
spec:
  clusterIP: None
  selector: { app: redis-node }
  ports: [{ port: 6379, targetPort: 6379 }]
---
# ── Deployment with named port, nodeSelector, sidecar, probes, volumes ──
apiVersion: apps/v1
kind: Deployment
metadata: { name: web-app }
spec:
  replicas: 2
  selector: { matchLabels: { app: web-app } }
  template:
    metadata: { labels: { app: web-app } }
    spec:
      nodeSelector: { disktype: super-fast-nvme }       # Pending forever if no node matches ({L4.14})
      containers:
        - name: web-server
          image: nginx:1.25
          ports:
            - { name: http-port, containerPort: 80 }     # named port ({L4.14})
          envFrom:
            - secretRef: { name: vault-secret }           # all keys as env vars ({L4.35})
          startupProbe:                                   # 12 × 10s = 120s boot budget ({L4.52})
            httpGet: { path: /, port: http-port }
            failureThreshold: 12
            periodSeconds: 10
          livenessProbe:                                  # restart on hang ({L4.50})
            httpGet: { path: /, port: http-port }
            periodSeconds: 5
          readinessProbe:                                 # gate traffic ({L4.51})
            tcpSocket: { port: 80 }
            periodSeconds: 3
          volumeMounts:
            - { name: shared-scratchpad, mountPath: /var/log/nginx }
            - name: config-volume                          # subPath: one file, dir intact ({L4.25})
              mountPath: /etc/app/special.txt
              subPath: special_key.txt                     # NOTE: subPath does not auto-update
        - name: log-shipper-sidecar                        # sidecar shares net + volume ({L4.13},{L4.22})
          image: busybox:1.36
          command: ['sh','-c','tail -f /logs/access.log']
          volumeMounts:
            - { name: shared-scratchpad, mountPath: /logs }
      volumes:
        - name: shared-scratchpad
          emptyDir: { sizeLimit: 500Mi }                   # exceed → pod evicted ({L4.30})
        - name: config-volume
          configMap: { name: app-config }
---
# ── DaemonSet: NO replicas field → exactly 1 pod per node ({L4.17}) ──
apiVersion: apps/v1
kind: DaemonSet
metadata: { name: fluentd-logger }
spec:
  selector: { matchLabels: { app: log-collector } }
  template:
    metadata: { labels: { app: log-collector } }
    spec:
      containers:
        - { name: collector, image: busybox:1.36, command: ["sleep","infinity"] }
---
# ── CronJob: suspendable schedule; Jobs need restartPolicy OnFailure/Never ({L4.18},{L4.19},{L4.20}) ──
apiVersion: batch/v1
kind: CronJob
metadata: { name: hourly-batch-job }
spec:
  schedule: "0 * * * *"
  suspend: false                       # true = keep schedule, stop spawning Jobs ({L4.19})
  jobTemplate:
    spec:
      completions: 2                    # must exit 0 twice to be Complete ({L4.18})
      parallelism: 1
      template:
        spec:
          restartPolicy: OnFailure      # batch pods can't use Always
          containers:
            - { name: calculator, image: perl:5.34, command: ["perl","-e","print 'done\n'"] }
```

---

## G · Last-night checklist

- [ ] `--dry-run=client -o yaml` to scaffold every manifest — don't hand-write YAML.
- [ ] `describe pod` → **read Events bottom-up** before touching anything.
- [ ] `get endpoints <svc>` whenever a Service "returns nothing."
- [ ] `logs --previous` (`-p`) for any CrashLoopBackOff.
- [ ] `-p HOST:CONTAINER` — never reverse it.
- [ ] Custom Podman network = DNS by name; bind-mount on RHEL = `:Z`.
- [ ] `oc` = `kubectl` + (projects, Routes, `new-app`, `rsh`, `debug`, SCC).
- [ ] LO6: state a **position first**, then defend with trade-offs.
- [ ] Run the DO188 labs (Appendix D) once tonight so the commands are reflex.

*Sources: your full exam notes, the DO188 lab PDFs, the MT2 command/solution references, and the kitchen-sink manifest. Verify exact image tags / hostnames against the live exam wording.*
