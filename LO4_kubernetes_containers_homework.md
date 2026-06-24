# LO4 — Accelerated Delivery of Multilayer Applications Using Containers

This document answers the LO4 Kubernetes homework tasks.  
Assumptions:

- `kubectl` is installed and configured for your cluster.
- Examples use the `default` namespace unless another namespace is created.
- Minikube examples assume `minikube` is running.
- Replace fake credentials, node labels, certificate files, and generated pod names with your real values.
- Commands are written for a Linux/macOS shell. On Windows PowerShell, quote escaping may need small changes.

---

## 1. Create a Deployment named `web` running `nginx:1.25` with 3 replicas using one imperative command, then export YAML

Create the Deployment:

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3 --port=80
```

Export the generated YAML:

```bash
kubectl get deployment web -o yaml > web-generated.yaml
```

Verify:

```bash
kubectl get deployment web
kubectl get pods -l app=web
```

Expected idea:

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    3/3     3            3           1m
```

If the teacher wants the YAML generated before creating the object, use client dry-run:

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3 --port=80 \
  --dry-run=client -o yaml > web-generated.yaml

kubectl apply -f web-generated.yaml
```

---

## 2. Enable authenticated image pulling from DockerHub

You need a Kubernetes **Secret** of type `kubernetes.io/dockerconfigjson`. The imperative helper command creates it as a `docker-registry` Secret.

```bash
kubectl create secret docker-registry dockerhub-creds \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<dockerhub-username> \
  --docker-password=<dockerhub-password-or-access-token> \
  --docker-email=<email>
```

Then reference it from a Pod or Deployment using `imagePullSecrets`:

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: dockerhub-creds
```

This is required for private images and useful for authenticated DockerHub pulls to reduce the chance of anonymous pull-rate limiting.

---

## 3. Scale `web` from 3 to 5 replicas two different ways and show the ReplicaSet

### Way 1: `kubectl scale`

```bash
kubectl scale deployment/web --replicas=5
kubectl get deployment web
kubectl get rs -l app=web -o wide
kubectl get pods -l app=web
```

Expected ReplicaSet idea:

```text
NAME              DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES
web-xxxxxxxxxx    5         5         5       5m    nginx        nginx:1.25
```

Scaling only changes replica count. It does **not** create a new ReplicaSet because the Pod template did not change.

### Way 2: edit the manifest

Edit `web-generated.yaml`:

```yaml
spec:
  replicas: 5
```

Apply it:

```bash
kubectl apply -f web-generated.yaml
kubectl get rs -l app=web -o wide
```

Again, the same ReplicaSet should normally show `DESIRED=5` because only the replica count changed.

---

## 4. Perform a rolling update from `nginx:1.25` to `nginx:1.27`

First check the container name:

```bash
kubectl get deployment web -o jsonpath='{.spec.template.spec.containers[*].name}{"\n"}'
```

For a Deployment created from `nginx`, the container name is usually `nginx`.

Run the update:

```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
```

Watch ReplicaSets:

```bash
kubectl get rs -l app=web -w
```

Verify the image:

```bash
kubectl get deployment web -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl get pods -l app=web -o wide
```

A new ReplicaSet is created because the Pod template changed.

---

## 5. View rollout history and roll back to the previous revision

View rollout history:

```bash
kubectl rollout history deployment/web
```

Roll back to the previous revision:

```bash
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
```

Roll back to a specific revision:

```bash
kubectl rollout undo deployment/web --to-revision=1
```

### What `CHANGE-CAUSE` shows

`CHANGE-CAUSE` shows the value of the annotation:

```text
kubernetes.io/change-cause
```

It is meant to explain why a rollout happened.

Populate it before making a rollout:

```bash
kubectl annotate deployment/web \
  kubernetes.io/change-cause="Upgrade nginx from 1.25 to 1.27" \
  --overwrite

kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout history deployment/web
```

Older tutorials use `kubectl --record`, but that workflow is deprecated. Using the annotation directly is clearer.

---

## 6. Set `RollingUpdate` with `maxSurge: 1` and `maxUnavailable: 0`

Patch it:

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

Manifest form:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

Effect:

- `maxSurge: 1` allows Kubernetes to create 1 extra Pod above the desired replica count during the update.
- `maxUnavailable: 0` means Kubernetes should not intentionally take any available replica offline during the update.
- With 5 replicas, Kubernetes may temporarily run 6 Pods, but it keeps 5 available while replacing old Pods with new Pods.
- This improves availability but requires enough cluster capacity for the extra surge Pod.

---

## 7. Switch strategy to `Recreate` and describe when it is required

Patch:

```bash
kubectl patch deployment web --type=merge -p '{
  "spec": {
    "strategy": {
      "type": "Recreate"
    }
  }
}'
```

Manifest form:

```yaml
spec:
  strategy:
    type: Recreate
```

`Recreate` stops all old Pods before creating new Pods.

Concrete scenario:

A database-backed application performs a schema migration that is not backward-compatible. Version 1 and version 2 of the app cannot safely run at the same time because they expect different database schemas. In that case, a rolling update could break data or produce errors, so `Recreate` is safer even though it causes downtime.

---

## 8. Add CPU/memory requests and limits and verify in the running Pod spec

Patch the Deployment:

```bash
kubectl patch deployment web --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/resources",
    "value": {
      "requests": {
        "cpu": "100m",
        "memory": "128Mi"
      },
      "limits": {
        "cpu": "500m",
        "memory": "256Mi"
      }
    }
  }
]'
```

Verify on the Deployment:

```bash
kubectl get deployment web -o yaml | grep -A10 resources:
```

Verify on a running Pod:

```bash
POD=$(kubectl get pod -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl get pod "$POD" -o jsonpath='{.spec.containers[0].resources}{"\n"}'
```

Expected idea:

```text
{"limits":{"cpu":"500m","memory":"256Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}
```

Requests are used by the scheduler to place Pods. Limits are enforced by the container runtime through resource controls.

---

## 9. Set `revisionHistoryLimit: 3`

Patch:

```bash
kubectl patch deployment web --type=merge -p '{
  "spec": {
    "revisionHistoryLimit": 3
  }
}'
```

Manifest:

```yaml
spec:
  revisionHistoryLimit: 3
```

Explanation:

- Kubernetes stores Deployment revision history in old ReplicaSets.
- With `revisionHistoryLimit: 3`, Kubernetes keeps up to 3 old ReplicaSets for rollback.
- You can roll back only to revisions that still have retained ReplicaSets.
- Older ReplicaSets beyond the limit are cleaned up.
- The current ReplicaSet still exists; the limit is about old rollback history.

Verify:

```bash
kubectl get rs -l app=web
kubectl rollout history deployment/web
```

---

## 10. Set a non-existent image tag and observe a stuck rollout

Set a bad image:

```bash
kubectl set image deployment/web nginx=nginx:no-such-tag-hopefully
kubectl rollout status deployment/web --timeout=60s
```

Observe:

```bash
kubectl get pods -l app=web
kubectl describe pod -l app=web | grep -A5 -E "ErrImagePull|ImagePullBackOff|Failed"
kubectl get rs -l app=web -o wide
```

Expected behavior:

- New Pods fail with `ErrImagePull` or `ImagePullBackOff`.
- The rollout does not complete.
- Old Pods keep serving traffic because a rolling update does not remove healthy old Pods until replacement Pods become available.
- With `maxUnavailable: 0`, Kubernetes is especially conservative and keeps the old available replicas.

Recover:

```bash
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
```

---

## 11. Use a label selector to list only Pods belonging to one Deployment

List only `web` Pods:

```bash
kubectl get pods -l app=web
```

Show selectors:

```bash
kubectl get deployment web -o jsonpath='{.spec.selector.matchLabels}{"\n"}'
kubectl get rs -l app=web
kubectl get pods -l app=web --show-labels
```

Relationship:

```text
Deployment → ReplicaSet → Pod
```

- The Deployment has `.spec.selector`.
- The Deployment creates ReplicaSets using a Pod template.
- ReplicaSets also have selectors.
- ReplicaSets create and own Pods whose labels match those selectors.
- The selector must match the Pod template labels, otherwise Kubernetes rejects the object or the controller cannot manage the Pods correctly.

---

## 12. Expose a Deployment with `kubectl expose`

Expose `web` internally:

```bash
kubectl expose deployment web \
  --name=web-svc \
  --port=80 \
  --target-port=80 \
  --type=ClusterIP
```

Inspect:

```bash
kubectl get service web-svc
kubectl get service web-svc -o yaml
```

Object created:

```text
Service
```

How the selector was derived:

- `kubectl expose deployment web` reads the Deployment selector.
- The Service gets a selector such as:

```yaml
selector:
  app: web
```

The Service then routes traffic to Pods matching that selector.

---

## 13. Add a sidecar container to a Deployment

Example Pod template snippet:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-sidecar
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-sidecar
  template:
    metadata:
      labels:
        app: web-sidecar
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          volumeMounts:
            - name: shared-logs
              mountPath: /shared
        - name: sidecar
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              while true; do
                date >> /shared/sidecar.log
                sleep 5
              done
          volumeMounts:
            - name: shared-logs
              mountPath: /shared
      volumes:
        - name: shared-logs
          emptyDir: {}
```

Apply:

```bash
kubectl apply -f web-sidecar.yaml
```

Explanation:

- Both containers share the same Pod network namespace.
- They have the same Pod IP.
- They can talk to each other using `localhost`.
- They can share files by mounting the same volume, such as `emptyDir`.

Verify:

```bash
POD=$(kubectl get pod -l app=web-sidecar -o jsonpath='{.items[0].metadata.name}')
kubectl exec "$POD" -c nginx -- cat /shared/sidecar.log
```

---

## 14. Deployment manifest for `httpd:2.4` with 2 replicas, named port, and `nodeSelector`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-node
spec:
  replicas: 2
  selector:
    matchLabels:
      app: httpd-node
  template:
    metadata:
      labels:
        app: httpd-node
    spec:
      nodeSelector:
        disktype: ssd
      containers:
        - name: httpd
          image: httpd:2.4
          ports:
            - name: http
              containerPort: 80
```

Apply:

```bash
kubectl apply -f httpd-node.yaml
kubectl get pods -l app=httpd-node -o wide
```

If no node has the label `disktype=ssd`, the Pods stay `Pending`.

Check node labels:

```bash
kubectl get nodes --show-labels
```

Add the label to a node if you want the Pods to schedule:

```bash
kubectl label node <node-name> disktype=ssd
```

Reason:

The scheduler can only place the Pods on nodes matching:

```yaml
nodeSelector:
  disktype: ssd
```

If no node matches, there is no valid placement.

---

## 15. Create a bare Pod running BusyBox, then delete it

Create a bare Pod:

```bash
kubectl run sleeper \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sh -c "sleep 3600"
```

Verify:

```bash
kubectl get pod sleeper
```

Delete:

```bash
kubectl delete pod sleeper
kubectl get pod sleeper
```

Result:

- The bare Pod is deleted and does not come back.
- There is no controller watching it.

Compare with a Deployment-managed Pod:

```bash
POD=$(kubectl get pod -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod "$POD"
kubectl get pods -l app=web
```

Result:

- The deleted Deployment-managed Pod is replaced.
- The ReplicaSet notices the actual replica count is too low and creates a new Pod.

---

## 16. StatefulSet for `redis:7` with 3 replicas plus a headless Service

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
      targetPort: 6379
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

Apply:

```bash
kubectl apply -f redis-sts.yaml
kubectl get pods -l app=redis
```

Stable Pod names:

```text
redis-0
redis-1
redis-2
```

Per-Pod DNS records:

```text
redis-0.redis.default.svc.cluster.local
redis-1.redis.default.svc.cluster.local
redis-2.redis.default.svc.cluster.local
```

Verify DNS from a debug Pod:

```bash
kubectl run dns-test --rm -it --restart=Never --image=busybox:1.36 -- sh

nslookup redis.default.svc.cluster.local
nslookup redis-0.redis.default.svc.cluster.local
nslookup redis-1.redis.default.svc.cluster.local
nslookup redis-2.redis.default.svc.cluster.local
exit
```

A StatefulSet gives Pods stable ordinal names and stable network identities.

---

## 17. Create a DaemonSet running a BusyBox agent

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
          command:
            - sh
            - -c
            - |
              while true; do
                echo "agent running on $(hostname)"
                sleep 30
              done
```

Apply:

```bash
kubectl apply -f busybox-agent-ds.yaml
kubectl get daemonset busybox-agent
kubectl get pods -l app=busybox-agent -o wide
```

Explanation:

A DaemonSet ensures one Pod runs on each eligible node. If the cluster has 3 eligible nodes, the DaemonSet creates 3 Pods. If a node is added, the DaemonSet creates a Pod for that node too.

---

## 18. Create a Job that computes once and completes

Using a Perl image for an actual Perl computation:

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
          command:
            - perl
            - -e
            - |
              $sum = 0;
              $sum += $_ for (1..100);
              print "sum(1..100)=$sum\n";
```

Apply and watch:

```bash
kubectl apply -f calc-job.yaml
kubectl get jobs
kubectl get pods -l job-name=calc-once
```

Expected `COMPLETIONS`:

```text
NAME        STATUS     COMPLETIONS   DURATION   AGE
calc-once   Complete   1/1           5s         10s
```

Read result:

```bash
kubectl logs job/calc-once
```

Expected:

```text
sum(1..100)=5050
```

BusyBox alternative if your assignment strictly requires BusyBox:

```bash
kubectl create job calc-busybox --image=busybox:1.36 -- sh -c 'echo "6*7=$((6*7))"'
kubectl logs job/calc-busybox
```

---

## 19. Create a CronJob that prints the date every minute; suspend it and list spawned Jobs

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
              command:
                - sh
                - -c
                - date
```

Apply:

```bash
kubectl apply -f date-cronjob.yaml
kubectl get cronjob date-every-minute
```

Wait at least one minute, then list Jobs:

```bash
kubectl get jobs --sort-by=.metadata.creationTimestamp
```

A useful owner view:

```bash
kubectl get jobs -o custom-columns=NAME:.metadata.name,OWNER:.metadata.ownerReferences[0].name
```

Suspend:

```bash
kubectl patch cronjob date-every-minute -p '{"spec":{"suspend":true}}'
kubectl get cronjob date-every-minute
```

Unsuspend:

```bash
kubectl patch cronjob date-every-minute -p '{"spec":{"suspend":false}}'
```

---

## 20. Manually run a Job from an existing CronJob

```bash
kubectl create job date-manual --from=cronjob/date-every-minute
kubectl get job date-manual
kubectl logs job/date-manual
```

This is useful for testing a CronJob template immediately without waiting for the schedule.

---

## 21. Show StatefulSet ordered creation/termination and contrast with Deployment

Scale StatefulSet down:

```bash
kubectl scale statefulset redis --replicas=0
kubectl get pods -l app=redis -w
```

Expected termination order:

```text
redis-2
redis-1
redis-0
```

Scale up:

```bash
kubectl scale statefulset redis --replicas=3
kubectl get pods -l app=redis -w
```

Expected creation order:

```text
redis-0
redis-1
redis-2
```

Explanation:

- StatefulSet Pods have stable ordinal identities.
- By default, StatefulSet creates Pods in order and waits for each Pod to be ready before continuing.
- During scale down, it terminates higher ordinals first.

Deployment contrast:

```bash
kubectl scale deployment web --replicas=0
kubectl scale deployment web --replicas=3
kubectl get pods -l app=web -w
```

Deployment Pods have random suffixes, no stable ordinal identity, and are usually created or deleted in parallel depending on controller behavior.

---

## 22. Multi-container Pod sharing an `emptyDir`

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
      command:
        - sh
        - -c
        - |
          echo "hello from writer" > /data/message.txt
          sleep 3600
      volumeMounts:
        - name: shared
          mountPath: /data

    - name: reader
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          while [ ! -f /data/message.txt ]; do sleep 1; done
          cat /data/message.txt
          sleep 3600
      volumeMounts:
        - name: shared
          mountPath: /data

  volumes:
    - name: shared
      emptyDir: {}
```

Apply:

```bash
kubectl apply -f shared-emptydir.yaml
```

Prove it is shared:

```bash
kubectl logs shared-emptydir -c reader
kubectl exec shared-emptydir -c reader -- cat /data/message.txt
kubectl exec shared-emptydir -c writer -- cat /data/message.txt
```

Expected:

```text
hello from writer
```

---

## 23. List StorageClasses, PVs and PVCs

```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc
kubectl get pvc -A
```

Explanation:

- `StorageClass` describes dynamic storage provisioning options.
- `PersistentVolume` is cluster storage.
- `PersistentVolumeClaim` is a namespace-scoped request for storage.

---

## 24. Mount a ConfigMap as a volume so each key becomes a file

Create ConfigMap:

```bash
kubectl create configmap app-config \
  --from-literal=message="hello from configmap" \
  --from-literal=color="blue"
```

Pod:

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

Apply and verify:

```bash
kubectl apply -f config-volume-pod.yaml
kubectl exec config-volume-pod -- ls -l /etc/config
kubectl exec config-volume-pod -- cat /etc/config/message
kubectl exec config-volume-pod -- cat /etc/config/color
```

Expected:

```text
hello from configmap
blue
```

Each ConfigMap key becomes a file.

---

## 25. Mount one ConfigMap key to a specific path using `subPath`

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

Apply and verify:

```bash
kubectl apply -f config-subpath-pod.yaml
kubectl exec config-subpath-pod -- cat /app/message.txt
```

When needed:

Use `subPath` when you need to mount one file into an existing directory without replacing the whole directory.

Example: mounting only one config file into `/etc/nginx/conf.d/default.conf`.

Update caveat:

A ConfigMap mounted using `subPath` does **not** receive automatic live updates. You normally need to restart the Pod.

---

## 26. Mount a Secret as a volume and verify decoded values with restrictive permissions

Create Secret:

```bash
kubectl create secret generic app-secret \
  --from-literal=username=admin \
  --from-literal=password=s3cr3t
```

Pod:

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
        secretName: app-secret
        defaultMode: 0400
```

Apply and verify:

```bash
kubectl apply -f secret-volume-pod.yaml
kubectl exec secret-volume-pod -- ls -l /etc/secret
kubectl exec secret-volume-pod -- cat /etc/secret/username
kubectl exec secret-volume-pod -- cat /etc/secret/password
```

Expected:

```text
admin
s3cr3t
```

The files contain decoded values. The `defaultMode: 0400` makes permissions restrictive.

---

## 27. Scaling a StatefulSet creates one PVC per replica; what happens when the StatefulSet is deleted

StatefulSet with a PVC template:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-pvc
spec:
  clusterIP: None
  selector:
    app: redis-pvc
  ports:
    - name: redis
      port: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-pvc
spec:
  serviceName: redis-pvc
  replicas: 3
  selector:
    matchLabels:
      app: redis-pvc
  template:
    metadata:
      labels:
        app: redis-pvc
    spec:
      containers:
        - name: redis
          image: redis:7
          ports:
            - name: redis
              containerPort: 6379
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

Apply:

```bash
kubectl apply -f redis-pvc-sts.yaml
kubectl get pvc
```

Expected PVCs:

```text
data-redis-pvc-0
data-redis-pvc-1
data-redis-pvc-2
```

Scale:

```bash
kubectl scale statefulset redis-pvc --replicas=5
kubectl get pvc
```

Expected new PVCs:

```text
data-redis-pvc-3
data-redis-pvc-4
```

Delete StatefulSet:

```bash
kubectl delete statefulset redis-pvc
kubectl get pvc
```

PVC behavior:

- PVCs are not deleted automatically when the StatefulSet is deleted.
- Kubernetes keeps them to protect data.
- Delete them manually if you no longer need the data:

```bash
kubectl delete pvc data-redis-pvc-0 data-redis-pvc-1 data-redis-pvc-2
```

---

## 28. Use an initContainer to pre-populate data into a shared volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-shared-volume
spec:
  initContainers:
    - name: init-writer
      image: busybox:1.36
      command:
        - sh
        - -c
        - echo "created by init container" > /work/index.html
      volumeMounts:
        - name: work
          mountPath: /work

  containers:
    - name: main
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          cat /work/index.html
          sleep 3600
      volumeMounts:
        - name: work
          mountPath: /work

  volumes:
    - name: work
      emptyDir: {}
```

Apply:

```bash
kubectl apply -f init-shared-volume.yaml
kubectl logs init-shared-volume -c main
```

Expected:

```text
created by init container
```

Explanation:

- Init containers run before app containers.
- The init container writes to the shared `emptyDir`.
- The main container starts later and reads the pre-populated file.

---

## 29. Set `readOnly: true` on a volume mount and prove writes are rejected

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readonly-volume
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
          readOnly: true
  volumes:
    - name: data
      emptyDir: {}
```

Apply:

```bash
kubectl apply -f readonly-volume.yaml
```

Try to write:

```bash
kubectl exec readonly-volume -- sh -c 'echo test > /data/file.txt'
```

Expected error:

```text
can't create /data/file.txt: Read-only file system
```

---

## 30. Set a `sizeLimit` on an `emptyDir`

Use a memory-backed `emptyDir` for a clear demo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limited-emptydir
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: cache
          mountPath: /cache
  volumes:
    - name: cache
      emptyDir:
        medium: Memory
        sizeLimit: 10Mi
```

Apply:

```bash
kubectl apply -f limited-emptydir.yaml
```

Try to exceed it:

```bash
kubectl exec limited-emptydir -- sh -c 'dd if=/dev/zero of=/cache/bigfile bs=1M count=20'
```

Expected idea:

```text
No space left on device
```

Explanation:

- `sizeLimit` limits the capacity of the `emptyDir`.
- With `medium: Memory`, it is backed by tmpfs and the limit is easy to observe.
- If a container exceeds available ephemeral storage, writes can fail and the Pod can be evicted under storage pressure.

---

## 31. Create a generic Secret from literals and show base64 encoding

Create:

```bash
kubectl create secret generic db-creds \
  --from-literal=username=dbuser \
  --from-literal=password='p@ssw0rd'
```

View YAML:

```bash
kubectl get secret db-creds -o yaml
```

Expected idea:

```yaml
data:
  password: cEBzc3cwcmQ=
  username: ZGJ1c2Vy
```

Decode manually:

```bash
echo 'ZGJ1c2Vy' | base64 -d
echo
echo 'cEBzc3cwcmQ=' | base64 -d
echo
```

Expected:

```text
dbuser
p@ssw0rd
```

Important:

Kubernetes Secret data is base64-encoded, not automatically encrypted in the object output. Encryption at rest requires separate cluster configuration.

---

## 32. Create a Secret from files holding a certificate and key

Create sample files for testing:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=example.local"
```

Create generic Secret from files:

```bash
kubectl create secret generic cert-files \
  --from-file=tls.crt \
  --from-file=tls.key
```

Identify resulting keys:

```bash
kubectl get secret cert-files -o jsonpath='{.data}' | jq
```

If `jq` is not installed:

```bash
kubectl get secret cert-files -o yaml
```

Resulting keys are the file basenames:

```text
tls.crt
tls.key
```

---

## 33. Create a `kubernetes.io/tls` typed Secret and explain where it is consumed

Create TLS Secret:

```bash
kubectl create secret tls web-tls \
  --cert=tls.crt \
  --key=tls.key
```

Verify type:

```bash
kubectl get secret web-tls
kubectl get secret web-tls -o jsonpath='{.type}{"\n"}'
```

Expected:

```text
kubernetes.io/tls
```

Where consumed:

- Most commonly by an Ingress for HTTPS termination.
- It can also be mounted into Pods that need certificate/key files.

Ingress example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  tls:
    - hosts:
        - example.local
      secretName: web-tls
  rules:
    - host: example.local
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

---

## 34. Mount a Secret as a volume and explain trade-offs versus environment variables

Volume mount example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-tradeoff
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: db-secret
          mountPath: /run/secrets/db
          readOnly: true
  volumes:
    - name: db-secret
      secret:
        secretName: db-creds
```

Trade-offs:

Secret as volume:

- Can expose only selected keys.
- Can use file permissions such as `0400`.
- Secret volume updates can be projected into running Pods eventually.
- Better for applications that can read credentials from files.

Secret as environment variables:

- Easy for apps expecting env vars.
- Values are visible to the process environment.
- They do not update in a running container when the Secret changes; restart is needed.
- Env vars can be accidentally printed in logs or debug output.

Both methods require RBAC and node security to be handled carefully.

---

## 35. Use `envFrom` with `secretRef`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-envfrom
spec:
  restartPolicy: Never
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "username=$username"
          echo "password=$password"
      envFrom:
        - secretRef:
            name: db-creds
```

Apply and view:

```bash
kubectl apply -f secret-envfrom.yaml
kubectl logs secret-envfrom
```

Every key in the Secret becomes an environment variable.

---

## 36. Create a docker-registry image-pull Secret and reference it with `imagePullSecrets`

Create Secret:

```bash
kubectl create secret docker-registry dockerhub-creds \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<dockerhub-username> \
  --docker-password=<dockerhub-password-or-token> \
  --docker-email=<email>
```

Reference it in a Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: private-image-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: private-image-app
  template:
    metadata:
      labels:
        app: private-image-app
    spec:
      imagePullSecrets:
        - name: dockerhub-creds
      containers:
        - name: app
          image: <dockerhub-username>/<private-repo>:<tag>
```

Required when:

- Pulling private registry images.
- Pulling from registries that require authentication.
- Authenticating to DockerHub to reduce anonymous pull-limit problems.

---

## 37. Create a Secret with several keys and selectively mount only one

Create:

```bash
kubectl create secret generic multi-secret \
  --from-literal=username=appuser \
  --from-literal=password=apppass \
  --from-literal=token=abc123
```

Pod that mounts only `password`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: selective-secret
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: secret
          mountPath: /etc/only-secret
          readOnly: true
  volumes:
    - name: secret
      secret:
        secretName: multi-secret
        items:
          - key: password
            path: password.txt
```

Verify:

```bash
kubectl apply -f selective-secret.yaml
kubectl exec selective-secret -- ls -l /etc/only-secret
kubectl exec selective-secret -- cat /etc/only-secret/password.txt
```

Only `password.txt` should appear.

---

## 38. Create a ClusterIP Service and resolve it by DNS from another Pod

Create Service:

```bash
kubectl expose deployment web \
  --name=web-clusterip \
  --port=80 \
  --target-port=80 \
  --type=ClusterIP
```

Check:

```bash
kubectl get svc web-clusterip
```

DNS test:

```bash
kubectl run dns-client --rm -it --restart=Never --image=busybox:1.36 -- sh
```

Inside the debug Pod:

```sh
nslookup web-clusterip.default.svc.cluster.local
wget -qO- http://web-clusterip.default.svc.cluster.local
exit
```

Explanation:

Kubernetes creates DNS records for Services. The full service name is:

```text
<service-name>.<namespace>.svc.cluster.local
```

---

## 39. Create a NodePort Service and reach it through Minikube

Create:

```bash
kubectl expose deployment web \
  --name=web-nodeport \
  --port=80 \
  --target-port=80 \
  --type=NodePort
```

Check:

```bash
kubectl get svc web-nodeport
```

Access with Minikube helper:

```bash
minikube service web-nodeport --url
```

Access using node IP and nodePort:

```bash
NODE_PORT=$(kubectl get svc web-nodeport -o jsonpath='{.spec.ports[0].nodePort}')
MINIKUBE_IP=$(minikube ip)

echo "http://$MINIKUBE_IP:$NODE_PORT"
curl "http://$MINIKUBE_IP:$NODE_PORT"
```

---

## 40. Create a LoadBalancer Service and explain `<pending>` on Minikube

Create:

```bash
kubectl expose deployment web \
  --name=web-lb \
  --port=80 \
  --target-port=80 \
  --type=LoadBalancer
```

Check:

```bash
kubectl get svc web-lb
```

On Minikube, `EXTERNAL-IP` often shows:

```text
<pending>
```

Reason:

Minikube is local and does not automatically have a cloud provider load balancer. In cloud Kubernetes, a cloud load balancer would normally be provisioned.

Use Minikube tunnel:

```bash
minikube tunnel
```

In another terminal:

```bash
kubectl get svc web-lb
curl http://<external-ip>
```

---

## 41. Create a headless Service for StatefulSet and show per-Pod A records

Headless Service:

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
      targetPort: 6379
```

Apply:

```bash
kubectl apply -f redis-headless-svc.yaml
```

DNS test:

```bash
kubectl run dns-headless --rm -it --restart=Never --image=busybox:1.36 -- sh
```

Inside:

```sh
nslookup redis.default.svc.cluster.local
nslookup redis-0.redis.default.svc.cluster.local
nslookup redis-1.redis.default.svc.cluster.local
nslookup redis-2.redis.default.svc.cluster.local
exit
```

Explanation:

- A normal ClusterIP Service returns one virtual IP.
- A headless Service has `clusterIP: None`.
- With StatefulSet Pods, DNS can return per-Pod records such as:

```text
redis-0.redis.default.svc.cluster.local
```

---

## 42. Inspect Endpoints/EndpointSlice and explain selector population

Inspect:

```bash
kubectl get endpoints web-clusterip
kubectl get endpointslice -l kubernetes.io/service-name=web-clusterip
kubectl describe endpointslice -l kubernetes.io/service-name=web-clusterip
```

Show selector:

```bash
kubectl get svc web-clusterip -o yaml
kubectl get pods -l app=web -o wide
```

Explanation:

- The Service has a selector, for example `app: web`.
- Kubernetes finds Pods matching that selector.
- Matching ready Pods become backend endpoints.
- EndpointSlices are the scalable modern API for storing Service backend endpoint information.
- If there are no endpoints, check selector labels, Pod readiness, and target ports.

---

## 43. Demonstrate cross-namespace access with FQDN

Create another namespace:

```bash
kubectl create namespace other
```

From namespace `other`, test full DNS name:

```bash
kubectl run dns-other \
  -n other \
  --rm -it \
  --restart=Never \
  --image=busybox:1.36 \
  -- nslookup web-clusterip.default.svc.cluster.local
```

This should work.

Test short name from another namespace:

```bash
kubectl run dns-other-short \
  -n other \
  --rm -it \
  --restart=Never \
  --image=busybox:1.36 \
  -- nslookup web-clusterip
```

This usually fails because the short name is searched inside the current namespace first:

```text
web-clusterip.other.svc.cluster.local
```

Use one of these from another namespace:

```text
web-clusterip.default
web-clusterip.default.svc
web-clusterip.default.svc.cluster.local
```

---

## 44. Compare `kubectl port-forward` to a Service versus to a Pod

Port-forward to Service:

```bash
kubectl port-forward svc/web-clusterip 8080:80
```

Then:

```bash
curl http://localhost:8080
```

Port-forward to Pod:

```bash
POD=$(kubectl get pod -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward pod/$POD 8081:80
```

Then:

```bash
curl http://localhost:8081
```

Difference:

- `svc/web-clusterip` forwards to one backend selected by the Service. It is convenient when you care about the application as a Service.
- `pod/<pod-name>` forwards directly to one specific Pod. It is best for debugging a specific Pod.
- Service port-forwarding avoids needing to know a Pod name, but an active connection can still break if the selected backend disappears.
- Pod port-forwarding is precise but tied to that Pod's lifecycle.

---

## 45. Difference between Service `port`, `targetPort`, and `nodePort`

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Definitions:

- `port`: the port exposed by the Service inside the cluster.
- `targetPort`: the port on the selected backend Pods.
- `nodePort`: the port opened on every node for external access when using `type: NodePort` or a LoadBalancer that allocates node ports.

Example flow:

```text
client → nodeIP:30080 → Service port 80 → Pod targetPort 80
```

`targetPort` can also be a named container port, such as `targetPort: http`.

---

## 46. Verify connectivity to ClusterIP Service from a throwaway debug Pod

Start debug Pod:

```bash
kubectl run tmp --rm -it --restart=Never --image=busybox:1.36 -- sh
```

Inside:

```sh
nslookup web-clusterip
wget -qO- http://web-clusterip:80
nc -vz web-clusterip 80
exit
```

If `nc -vz` is not supported by the BusyBox build, use `wget`.

This proves the Service is reachable from inside the cluster.

---

## 47. Create two Deployments and one Service that load-balances across both

Deployment A:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-a
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shared-echo
      version: a
  template:
    metadata:
      labels:
        app: shared-echo
        version: a
    spec:
      containers:
        - name: echo
          image: hashicorp/http-echo:0.2.3
          args:
            - "-text=hello from A"
          ports:
            - containerPort: 5678
```

Deployment B:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-b
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shared-echo
      version: b
  template:
    metadata:
      labels:
        app: shared-echo
        version: b
    spec:
      containers:
        - name: echo
          image: hashicorp/http-echo:0.2.3
          args:
            - "-text=hello from B"
          ports:
            - containerPort: 5678
```

One Service selecting both:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: shared-echo
spec:
  selector:
    app: shared-echo
  ports:
    - port: 80
      targetPort: 5678
```

Apply:

```bash
kubectl apply -f echo-a.yaml
kubectl apply -f echo-b.yaml
kubectl apply -f shared-echo-svc.yaml
```

Verify endpoints include Pods from both Deployments:

```bash
kubectl get pods -l app=shared-echo --show-labels
kubectl get endpoints shared-echo
```

Prove requests hit both:

```bash
kubectl run echo-client --rm -it --restart=Never --image=busybox:1.36 -- sh
```

Inside:

```sh
for i in $(seq 1 20); do wget -qO- http://shared-echo; echo; done
exit
```

Expected output should include both:

```text
hello from A
hello from B
```

Explanation:

The Service selector is only:

```yaml
app: shared-echo
```

Both Deployments create Pods with that label, so the Service load-balances across both groups.

---

## 48. Diagnose why a Pod cannot reach a Service

Use this systematic checklist.

### 1. DNS

From a debug Pod:

```bash
kubectl run debug --rm -it --restart=Never --image=busybox:1.36 -- sh
```

Inside:

```sh
nslookup web-clusterip
nslookup web-clusterip.default.svc.cluster.local
exit
```

If DNS fails, check CoreDNS:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

### 2. Service exists and has correct port

```bash
kubectl get svc web-clusterip
kubectl describe svc web-clusterip
```

Check:

- `port`
- `targetPort`
- selector

### 3. Endpoints / EndpointSlices

```bash
kubectl get endpoints web-clusterip
kubectl get endpointslice -l kubernetes.io/service-name=web-clusterip
```

If there are no endpoints, the Service has no matching ready Pods.

### 4. Selector labels

```bash
kubectl get svc web-clusterip -o jsonpath='{.spec.selector}{"\n"}'
kubectl get pods --show-labels
kubectl get pods -l app=web
```

The Service selector must match Pod labels.

### 5. Readiness

```bash
kubectl get pods -l app=web
kubectl describe pod -l app=web
```

If Pods are not Ready, they may not be added as ready endpoints.

### 6. Ports

Check container port and actual listening port:

```bash
kubectl get deployment web -o yaml | grep -A5 ports:
kubectl exec deploy/web -- sh -c 'nginx -T 2>/dev/null | grep listen || true'
```

Test directly against Pod IP:

```bash
kubectl get pods -l app=web -o wide
```

From debug Pod:

```sh
wget -qO- http://<pod-ip>:80
```

If Pod IP works but Service fails, the issue is likely selector, endpoints, or Service port mapping.

---

## 49. Expose a Service with NodePort and verify it works

Create NodePort Service:

```bash
kubectl expose deployment web \
  --name=web-nodeport-verify \
  --port=80 \
  --target-port=80 \
  --type=NodePort
```

Verify object:

```bash
kubectl get svc web-nodeport-verify
```

Get nodePort:

```bash
NODE_PORT=$(kubectl get svc web-nodeport-verify -o jsonpath='{.spec.ports[0].nodePort}')
echo "$NODE_PORT"
```

Minikube verification:

```bash
minikube service web-nodeport-verify --url
curl "$(minikube service web-nodeport-verify --url)"
```

Alternative:

```bash
curl "http://$(minikube ip):$NODE_PORT"
```

If using Docker driver on some systems, `minikube service --url` is often more reliable than direct node IP access.

---

## 50. Add HTTP livenessProbe to nginx and verify with `describe pod`

Patch:

```bash
kubectl patch deployment web --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/livenessProbe",
    "value": {
      "httpGet": {
        "path": "/",
        "port": 80
      },
      "initialDelaySeconds": 5,
      "periodSeconds": 10
    }
  }
]'
```

Wait for new Pods:

```bash
kubectl rollout status deployment/web
```

Verify:

```bash
POD=$(kubectl get pod -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod "$POD" | grep -A8 Liveness
```

Expected idea:

```text
Liveness: http-get http://:80/ delay=5s timeout=1s period=10s
```

Explanation:

If the HTTP GET `/` on port 80 fails repeatedly, the kubelet restarts the container.

---

## 51. Add a TCP readiness probe to Redis on port 6379

Create Redis Deployment:

```bash
kubectl create deployment redis-readiness --image=redis:7 --port=6379
```

Patch readiness probe:

```bash
kubectl patch deployment redis-readiness --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/readinessProbe",
    "value": {
      "tcpSocket": {
        "port": 6379
      },
      "initialDelaySeconds": 5,
      "periodSeconds": 10
    }
  }
]'
```

Verify:

```bash
kubectl rollout status deployment/redis-readiness
POD=$(kubectl get pod -l app=redis-readiness -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod "$POD" | grep -A8 Readiness
```

Expected idea:

```text
Readiness: tcp-socket :6379 delay=5s timeout=1s period=10s
```

Explanation:

The Pod is marked Ready only when a TCP connection to port 6379 succeeds. Services route traffic only to ready Pods.

---

## 52. Configure a startup probe for a roughly 2-minute boot

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: slow-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: slow-web
  template:
    metadata:
      labels:
        app: slow-web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          startupProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 10
            failureThreshold: 12
          livenessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 10
            failureThreshold: 3
```

Budget calculation:

```text
failureThreshold × periodSeconds = 12 × 10s = 120s
```

Justification:

- The application is expected to need about 2 minutes to boot.
- The startup probe gives it 120 seconds before Kubernetes treats startup as failed.
- While the startup probe is failing, liveness checks do not kill the container too early.
- After startup succeeds, liveness probing protects the running application.

---

## 53. Default behavior when no probes are defined

If no probes are configured:

- No liveness probe: Kubernetes does not perform app-specific health checks. The container is restarted only if the process exits, crashes, or violates other runtime conditions.
- No readiness probe: Kubernetes generally considers the container ready once it is running, so Services may send traffic to it even if the application inside is not actually ready.
- No startup probe: Liveness checks, if configured, start according to their own delays. Without startup probe, slow-starting apps can be killed too early if liveness is too aggressive.
- For production apps, probes are important because process-running does not always mean application-healthy.

---

## Useful cleanup commands

Use these only after finishing the lab:

```bash
kubectl delete deployment web web-sidecar httpd-node redis-readiness slow-web echo-a echo-b private-image-app --ignore-not-found
kubectl delete service web-svc web-clusterip web-nodeport web-lb web-nodeport-verify shared-echo redis redis-pvc --ignore-not-found
kubectl delete statefulset redis redis-pvc --ignore-not-found
kubectl delete daemonset busybox-agent --ignore-not-found
kubectl delete job calc-once calc-busybox date-manual --ignore-not-found
kubectl delete cronjob date-every-minute --ignore-not-found
kubectl delete pod sleeper shared-emptydir config-volume-pod config-subpath-pod secret-volume-pod secret-volume-tradeoff secret-envfrom selective-secret init-shared-volume readonly-volume limited-emptydir --ignore-not-found
kubectl delete configmap app-config --ignore-not-found
kubectl delete secret app-secret db-creds cert-files web-tls dockerhub-creds multi-secret --ignore-not-found
kubectl delete namespace other --ignore-not-found
```

PVC cleanup, only if you do not need the data:

```bash
kubectl delete pvc -l app=redis-pvc --ignore-not-found
kubectl get pvc
```

---

## References

Official Kubernetes documentation used for this answer:

- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Updating a Deployment](https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/)
- [Pull an Image from a Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
- [Images and imagePullSecrets](https://kubernetes.io/docs/concepts/containers/images/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [DaemonSets](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
- [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [StorageClasses](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/concepts/workloads/pods/probes/)
- [Assign Pods to Nodes](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/)
