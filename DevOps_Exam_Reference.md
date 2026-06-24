# Intro to DevOps — Exam Reference Guide

*A complete reference of the tools, commands, configuration, and concepts used in the TempConverter containerization & deployment project. Built for open-book use — scan the Table of Contents, jump to the section, read the explanation, copy the command.*

---

## Table of Contents

1. [Core concepts & definitions](#1-core-concepts--definitions)
2. [The application (TempConverter)](#2-the-application-tempconverter)
3. [The tools — what each one is](#3-the-tools--what-each-one-is)
4. [Dockerfile reference](#4-dockerfile-reference)
5. [Command reference by tool](#5-command-reference-by-tool)
6. [Docker Swarm stack file — fields explained](#6-docker-swarm-stack-file--fields-explained)
7. [Kubernetes manifests — fields explained](#7-kubernetes-manifests--fields-explained)
8. [CI pipeline (GitHub Actions) — explained](#8-ci-pipeline-github-actions--explained)
9. [Container vs Virtual Machine](#9-container-vs-virtual-machine)
10. [Docker Swarm vs Kubernetes](#10-docker-swarm-vs-kubernetes)
11. [Troubleshooting reference](#11-troubleshooting-reference)
12. [Likely exam questions & short answers](#12-likely-exam-questions--short-answers)

---

## 1. Core concepts & definitions

**Container** — A lightweight, isolated unit that packages an application together with its dependencies (libraries, runtime) so it runs the same on any host. A container *shares the host operating-system kernel*; it is not a full computer. Implemented on Linux using two kernel features:
- **Namespaces** — provide *isolation*: each container sees its own process tree (PID), network, mounts, users, hostname, etc.
- **cgroups (control groups)** — provide *resource limiting and accounting*: how much CPU, memory, etc. a container may use.

**Image** — A read-only template (a stack of filesystem layers + metadata) from which containers are created. You *build* an image (e.g. from a Dockerfile) and *run* containers from it. One image → many identical containers.

**Layer** — Each instruction in a Dockerfile adds a layer. Layers are cached and shared between images, which is why rebuilding after a small change is fast (only changed layers and everything after them rebuild).

**Registry** — A server that stores and distributes images (e.g. **Docker Hub**, GitHub Container Registry `ghcr.io`, AWS ECR). You `push` images to it and `pull` them down. Lets every cluster node fetch the *identical* image instead of rebuilding.

**Tag** — A label on an image, written `repository:tag` (e.g. `tempconverter:latest`, `tempconverter:dev`). `latest` is just a conventional default tag, not "the newest" automatically. Tags are how you version images.

**Microservices** — An architecture where an application is split into small, independent services that each do one job and communicate over the network (vs a single large "monolith"). Containers are the common way to package microservices.

**Orchestration / container orchestration system** — Software that automates running containers across many machines: scheduling them onto nodes, keeping the desired number running, restarting failed ones, load-balancing, scaling, rolling updates. Examples: **Docker Swarm** (simple), **Kubernetes** (complex).

**Node** — A single machine (physical or virtual) that is part of a cluster and runs containers.

**Cluster** — A group of nodes managed together by an orchestrator. Usually has *manager/control-plane* nodes (make scheduling decisions) and *worker* nodes (run the workload).

**Replica** — One running copy of an application. Running multiple replicas gives high availability and lets load be shared.

**Load balancing** — Distributing incoming requests across multiple replicas so no single one is overloaded and so the service keeps working if one replica dies.

**Self-healing** — The orchestrator automatically restarts or replaces containers that crash or fail health checks, returning the system to the desired state without human action.

**Service discovery** — How containers find each other by *name* instead of IP. The orchestrator runs internal DNS so a service named `db` is reachable at the hostname `db`.

**Overlay network** — A virtual network that spans all nodes in a cluster, letting containers on *different* physical machines communicate as if on one LAN. Used by Swarm and Kubernetes.

**Anti-affinity (placement rule)** — A scheduling rule that prevents certain containers from running on the same node, so a single node failure can't take out all replicas. (Swarm: `max_replicas_per_node`; Kubernetes: `podAntiAffinity`.)

**Affinity / nodeSelector / constraint / toleration** — Rules that *attract* or *restrict* where containers run (e.g. "database only on the manager/control-plane node").

**Secret** — A dedicated object for storing sensitive values (passwords, tokens) separately from the rest of the config, so they aren't hard-coded in plain manifests.

**Environment variable** — A key=value pair passed into a container at start-up to configure it (e.g. `DB_HOST`, `DB_PASS`). The 12-factor way to configure containers.

**Non-root user** — Running as a normal (unprivileged) user instead of `root`, to limit damage if compromised. In this project two senses:
- *Container OS user* — the Dockerfile creates `appuser` so the process inside the container isn't root.
- *Database user* — the app connects to MySQL as `appuser`, not the MySQL `root` user.

**Volume / persistent storage** — Storage that lives *outside* the container's writable layer so data survives container restarts/removal (e.g. the MySQL data directory mounted on a named volume).

**CI/CD (Continuous Integration / Continuous Delivery)** — Automatically testing and building code on every change. Here: a GitHub Actions pipeline runs tests and builds the image on every `git push`.

**Unit test vs Integration test**
- *Unit test* — tests one small piece in isolation (the pure conversion function), fast, no database or web server.
- *Integration test* — tests multiple parts working together (the Flask app + the real MySQL database), confirming a conversion is computed, stored, and read back.

---

## 2. The application (TempConverter)

- A **Python Flask** web app that converts Celsius → Fahrenheit and logs each conversion to **MySQL 8**.
- Listens on **0.0.0.0:5000**, started with `python app.py`.
- Configured entirely by **environment variables**:
  - `DB_USER`, `DB_PASS`, `DB_HOST`, `DB_NAME` → database connection (`mysql+pymysql://USER:PASS@HOST/DBNAME`).
  - `STUDENT`, `COLLEGE` → name and college shown on the page.
- **Key behaviour / the "race condition":** on start-up the app calls `db.create_all()`, so it tries to reach MySQL *the instant it boots*. If the database isn't ready yet, the app crashes. This drove many design decisions:
  - Locally: start DB first, wait until it answers, then start the app.
  - On orchestrators: solved automatically by **self-healing** (Swarm `restart_policy`, Kubernetes default restarts).
- Dependencies (in `requirements.txt`): Flask, Flask-WTF, Flask-SQLAlchemy, **pymysql**, **cryptography** (the last is needed for MySQL 8's `caching_sha2_password` auth).

---

## 3. The tools — what each one is

| Tool | What it is | Role in the project |
|---|---|---|
| **WSL2** (Windows Subsystem for Linux 2) | Runs a real Linux kernel inside a lightweight VM on Windows | The Linux environment where podman ran |
| **podman** | Daemonless, rootless container engine; CLI compatible with Docker | Built, ran, pushed images; local deployment |
| **Docker / Docker Engine** | The original container platform (with a background daemon) | Used *inside* the Swarm VMs (Swarm is a Docker feature) |
| **Docker Hub** | Public container registry | Stored the image so cluster nodes could pull it |
| **MySQL 8** | Relational database | Stores conversion records |
| **Flask** | Python web framework | The application itself |
| **Git** | Version control system | Tracks all code |
| **GitHub** | Hosting for Git repos + CI | Remote repo + Actions pipeline |
| **GitHub Actions** | CI/CD service built into GitHub | Runs tests + builds image on each push |
| **pytest** | Python testing framework | Runs unit & integration tests |
| **multipass** | Canonical tool to launch lightweight Ubuntu VMs | Created the multi-node Swarm and k8s clusters, and the comparison VM |
| **Docker Swarm** | Simple orchestrator built into Docker | The "simpler" orchestration system |
| **Kubernetes** | Industry-standard orchestrator | The "complex" orchestration system |
| **k3s** | Lightweight, fully-conformant Kubernetes distribution | Ran Kubernetes on the laptop |
| **kubectl** | The Kubernetes command-line client | Controls the k8s cluster |
| **systemd** | Linux init system / service manager | Manages services (Docker, k3s) inside VMs; enabled in WSL for cgroups v2 |

---

## 4. Dockerfile reference

A **Dockerfile** is a text recipe for building an image. Each instruction = one layer.

```dockerfile
FROM python:3.12-slim                 # base image to build on
RUN apt-get update && apt-get upgrade -y && \   # (a) update OS packages
    rm -rf /var/lib/apt/lists/*       # clean cache to keep image small
WORKDIR /app                          # set working dir for later instructions
COPY requirements.txt .               # copy deps list first (better layer caching)
RUN pip install --no-cache-dir -r requirements.txt   # (c) install requirements
COPY . .                              # copy the rest of the app
RUN useradd --create-home appuser     # create a non-root user
USER appuser                          # run as that user from here on
EXPOSE 5000/tcp                       # (b) document the port the app listens on
CMD ["python", "app.py"]              # (d) default command to start the app
```

**Instruction meanings**

| Instruction | Meaning |
|---|---|
| `FROM` | Base image to start from |
| `RUN` | Execute a command at *build* time (creates a layer) |
| `WORKDIR` | Set the current directory for following instructions |
| `COPY` | Copy files from build context into the image |
| `USER` | Switch the user that following instructions / the container run as |
| `EXPOSE` | Declare the port the app uses (documentation; does *not* publish it) |
| `CMD` | Default command run when a container starts (can be overridden) |
| `ENTRYPOINT` | Like CMD but harder to override (fixed executable) |
| `ENV` | Set an environment variable baked into the image |
| `ARG` | Build-time variable |

**Build-context note:** `.dockerignore` lists files to *exclude* from the image (e.g. `.git`, `screenshots/`), keeping it lean.

---

## 5. Command reference by tool

> Convention used in the project: inside the Swarm/k8s VMs, every Docker/kubectl command was run via `multipass exec <vm> -- sudo <command>`. Below the bare command is shown; prefix as needed.

### 5.1 podman (and Docker — same syntax)

```bash
podman build -t tempconverter:latest .          # build image from Dockerfile in current dir; -t = name:tag
podman images                                   # list local images (and sizes)
podman system df                                # disk used by images/containers/volumes
podman tag SRC docker.io/lpetrec/tempconverter:latest   # add a registry-qualified tag
podman login docker.io                          # authenticate to a registry (use a token as password)
podman push docker.io/lpetrec/tempconverter:latest      # upload image to registry
podman pull docker.io/lpetrec/tempconverter:latest      # download image from registry

podman network create tempconverter-net         # create a user-defined network (gives DNS by name)

podman run -d --name app \                       # run a container:
  --network tempconverter-net \                  #   -d  detached (background)
  -p 5000:5000 \                                 #   --name  fixed name
  -e DB_HOST=tempconverter-db \                  #   -p HOST:CONTAINER  publish port
  -e STUDENT="Luka Petrecija" \                  #   -e  set environment variable
  localhost/tempconverter:dev                    #   image to run
# extra useful run flags:
#   --replace   replace an existing container of the same name
#   --rm        auto-remove container when it stops
#   -v VOL:/path  mount a volume for persistent data

podman ps                                        # list running containers (-a for all)
podman exec CONTAINER CMD                         # run a command inside a running container
podman stats --no-stream                          # one-shot CPU/memory usage per container
podman logs CONTAINER                             # view a container's output
podman stop / start / rm CONTAINER                # stop / start / remove a container
podman volume rm VOL                              # remove a volume
```

Example proving the **non-root DB user**:
```bash
podman exec tempconverter-db mysql -uappuser -papppass \
  -e "SELECT CURRENT_USER();" tempconverter      # returns appuser@%  (not root)
```

### 5.2 Git / GitHub

```bash
git clone https://github.com/jstanesic/tempconverter.git   # copy a remote repo locally
git config user.name "Luka Petrecija"            # set commit author name
git config user.email "lpetrec@algebra.hr"
git add .                                         # stage all changes
git status                                        # show staged/unstaged/untracked
git commit -m "message"                           # record a snapshot
git remote set-url origin https://github.com/USER/repo.git   # point at your repo
git push -u origin HEAD                            # upload commits (first push sets upstream)
git ls-files                                       # list files Git is tracking (catches "not committed")
git log --oneline -5                               # recent commits, compact
git branch --show-current                          # current branch
```

**Personal Access Token (PAT) scopes** — GitHub needs a token (not your password) to push:
- `repo` — push/pull code.
- `workflow` — required *additionally* to create/update files under `.github/workflows/`.

### 5.3 pytest

```bash
pip install pytest --break-system-packages        # install (flag needed on Debian/Ubuntu PEP-668)
pytest tests/test_unit.py -v                       # run a test file, verbose (shows each PASSED)
pytest                                             # run all tests it can discover
```
- `conftest.py` — an (often empty) file at the repo root that puts the project root on the import path, so tests can `import app` / `import converter`. Fixes `ModuleNotFoundError`.

### 5.4 WSL (run in **Windows PowerShell**, not inside Ubuntu)

```powershell
wsl -l -v                  # list distros and their WSL version (1 or 2)
wsl --status               # default distro & version
wsl --install -d Ubuntu    # install a real Ubuntu distro
wsl --set-default Ubuntu   # make Ubuntu the default
wsl --update               # update the WSL engine (needed for systemd support)
wsl --shutdown             # fully stop WSL (needed to reload /etc/wsl.conf)
```
Enable **systemd** (needed for **cgroups v2**, which `podman stats` requires) — inside Ubuntu:
```bash
sudo tee /etc/wsl.conf > /dev/null << 'EOF'
[boot]
systemd=true
EOF
# then: wsl --shutdown (from PowerShell), reopen Ubuntu
stat -fc %T /sys/fs/cgroup/    # want: cgroup2fs   (tmpfs = not active yet)
```

### 5.5 multipass (run in **Windows PowerShell**)

```powershell
multipass launch --name swarm1 --memory 2G --disk 6G --cpus 1   # create+boot an Ubuntu VM
multipass list                                   # list VMs and their IPv4 addresses
multipass exec swarm1 -- <command>               # run a command INSIDE the VM (after --)
multipass info swarm1                             # VM details: memory, disk, CPU usage
multipass transfer SRC swarm1:/path/dest         # copy a file from host into the VM
multipass stop / start swarm1                     # stop / start a VM (stopped = frees RAM)
multipass delete swarm1; multipass purge          # delete a VM and reclaim its disk
```
> Reaching a WSL file from a Windows command: `\\wsl$\Ubuntu\home\student\file` — used as the SRC for `multipass transfer`.

### 5.6 Docker Swarm (the simpler orchestrator)

```bash
docker swarm init --advertise-addr <MANAGER_IP>   # turn this node into the Swarm manager
docker swarm join-token worker                     # print the command/token to add a worker
docker swarm join --token <SWMTKN-...> <MANAGER_IP>:2377   # run on a worker to join
docker node ls                                     # list cluster nodes (manager shows "Leader")
docker node rm <ID> --force                        # remove a (e.g. stale "Down") node

docker stack deploy -c tempconverter-stack.yml tempconverter   # deploy a stack from a file
docker stack services tempconverter                # services + REPLICAS (e.g. app 2/2)
docker stack ps tempconverter                      # tasks + which NODE each runs on
docker service scale tempconverter_app=3           # scale a service to N replicas
docker service ps tempconverter_app --no-trunc     # tasks of one service, full error text
docker stack rm tempconverter                      # remove the whole stack
docker swarm leave --force                          # make a node leave the swarm
```
Key terms: a **stack** = a set of services defined in one compose-style file; a **service** = a definition that runs N replica **tasks**; a **task** = one container. Port **2377** = Swarm management. The **routing mesh** publishes a port on *every* node and load-balances to the service.

### 5.7 Kubernetes via k3s (the complex orchestrator)

Install **server (control-plane)** on the first VM:
```bash
curl -sfL https://get.k3s.io | sudo sh -s - --disable traefik --write-kubeconfig-mode 644
#   --disable traefik           frees port 80 for our own LoadBalancer
#   --write-kubeconfig-mode 644 lets kubectl run without extra sudo
```
Get the **join token**, then join **workers**:
```bash
sudo cat /var/lib/rancher/k3s/server/node-token    # the FULL token (incl. the part after ::server:)
curl -sfL https://get.k3s.io | sudo K3S_URL=https://<SERVER_IP>:6443 K3S_TOKEN=<FULL_TOKEN> sh -
```
Control the cluster with **kubectl** (here `k3s kubectl`):
```bash
kubectl get nodes -o wide                          # list nodes (control-plane + workers)
kubectl apply -f tempconverter-k8s.yml             # create/update all objects in a manifest
kubectl get pods -o wide                           # pods + their NODE + RESTARTS
kubectl get svc app                                # services + EXTERNAL-IP + PORT(S)
kubectl scale deployment app --replicas=3          # scale a Deployment
kubectl describe pod -l app=tempconverter          # detailed status/events (scheduling errors)
```
Port **6443** = Kubernetes API server. `-o wide` adds the node/IP columns. `-l` selects by label.

---

## 6. Docker Swarm stack file — fields explained

```yaml
version: "3.8"                  # compose file format version
services:
  db:
    image: mysql:8             # which image to run
    environment:               # env vars passed to the container
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: tempconverter
      MYSQL_USER: appuser       # auto-creates a NON-ROOT user with rights on the DB
      MYSQL_PASSWORD: apppass
    volumes:
      - dbdata:/var/lib/mysql   # persistent storage for the database
    networks: [tcnet]           # attach to the overlay network
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]   # pin DB to the manager node
  app:
    image: docker.io/lpetrec/tempconverter:dev
    environment:
      DB_HOST: db               # reach the DB by SERVICE NAME (service discovery)
      STUDENT: "Luka Petrecija"
      COLLEGE: "Algebra Bernays University"
    ports: ["80:5000"]          # publish port 80 (host) -> 5000 (container) via routing mesh
    networks: [tcnet]
    deploy:
      replicas: 2               # run TWO app replicas
      placement:
        max_replicas_per_node: 1   # ANTI-AFFINITY: at most 1 app replica per node
      restart_policy:
        condition: on-failure   # SELF-HEALING: restart a replica if it exits with an error
networks:
  tcnet:
    driver: overlay             # cluster-wide virtual network
volumes:
  dbdata:                       # named volume declaration
```

---

## 7. Kubernetes manifests — fields explained

Kubernetes config = one or more **objects** separated by `---`. Core objects used:

- **Secret** — stores the DB credentials separately from the rest of the config.
- **Deployment** — declares the desired number of identical **pods** and keeps them running (self-heals).
- **Pod** — the smallest unit; wraps one (or more) containers. Created by the Deployment.
- **Service** — a stable name + virtual IP that load-balances to a set of pods.

```yaml
apiVersion: v1
kind: Secret                    # holds sensitive values
metadata: { name: tc-secret }
type: Opaque
stringData:
  DB_USER: appuser
  DB_PASS: apppass
  DB_NAME: tempconverter
  MYSQL_ROOT_PASSWORD: rootpass
---
apiVersion: apps/v1
kind: Deployment                # MySQL deployment
metadata: { name: db }
spec:
  replicas: 1
  selector: { matchLabels: { app: db } }   # which pods this Deployment owns
  template:                                 # the pod template
    metadata: { labels: { app: db } }
    spec:
      nodeSelector:                          # only schedule on the control-plane node
        node-role.kubernetes.io/control-plane: "true"
      tolerations:                           # allow running on the (tainted) control-plane
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      containers:
        - name: mysql
          image: mysql:8
          env:
            - { name: MYSQL_USER, valueFrom: { secretKeyRef: { name: tc-secret, key: DB_USER } } }
          ports: [ { containerPort: 3306 } ]
---
apiVersion: v1
kind: Service                   # gives MySQL a stable in-cluster name "db"
metadata: { name: db }
spec:
  selector: { app: db }
  ports: [ { port: 3306, targetPort: 3306 } ]
---
apiVersion: apps/v1
kind: Deployment                # the application
metadata: { name: app }
spec:
  replicas: 2                   # two app pods
  selector: { matchLabels: { app: tempconverter } }
  template:
    metadata: { labels: { app: tempconverter } }
    spec:
      affinity:
        podAntiAffinity:        # ANTI-AFFINITY: keep app pods on DIFFERENT nodes
          requiredDuringSchedulingIgnoredDuringExecution:   # HARD rule
            - labelSelector: { matchLabels: { app: tempconverter } }
              topologyKey: kubernetes.io/hostname           # "different node" = different hostname
      containers:
        - name: app
          image: docker.io/lpetrec/tempconverter:dev
          ports: [ { containerPort: 5000 } ]
          env:
            - { name: DB_HOST, value: db }   # reach DB by service name
            - { name: STUDENT, value: "Luka Petrecija" }
---
apiVersion: v1
kind: Service
metadata: { name: app }
spec:
  type: LoadBalancer            # expose the app OUTSIDE the cluster on port 80
  selector: { app: tempconverter }
  ports: [ { port: 80, targetPort: 5000 } ]   # 80 external -> 5000 in the container
```

**Service types (exam favourite):**
- `ClusterIP` (default) — reachable only inside the cluster.
- `NodePort` — opens a high port on every node.
- `LoadBalancer` — external access on a normal port (k3s provides this with its built-in load balancer).

**Anti-affinity scheduling modes:**
- `requiredDuringScheduling...` — **hard** rule; if it can't be satisfied the pod stays **Pending**.
- `preferredDuringScheduling...` — **soft** preference; placed anyway if needed.

---

## 8. CI pipeline (GitHub Actions) — explained

A workflow file lives at `.github/workflows/ci.yml`. Triggered on every `push`.

```yaml
name: CI
on: [push, workflow_dispatch]      # run on push, or manually
jobs:
  test-and-build:
    runs-on: ubuntu-latest         # the runner OS
    services:                      # side-car containers for the job
      mysql:
        image: mysql:8             # a real MySQL for integration tests
        env: { MYSQL_USER: appuser, MYSQL_PASSWORD: apppass, ... }
        options: >-                # health-check so the job can wait for it
          --health-cmd="mysqladmin ping --silent" ...
    env: { DB_HOST: 127.0.0.1, ... }   # env for the app/tests
    steps:
      - uses: actions/checkout@v4          # check out the repo
      - uses: actions/setup-python@v5      # install Python
      - run: pip install -r requirements.txt pytest   # install deps
      - run: pytest tests/test_unit.py -v             # unit tests
      - run: <wait loop until MySQL answers>          # handle the DB race
      - run: pytest tests/test_integration.py -v      # integration tests
      - run: docker build -t tempconverter:ci .       # build the image
```
**Concepts:** *workflow* (the whole file) → *job* (`test-and-build`) → *steps* (run in order). A *service container* (the MySQL side-car) lets integration tests hit a real database. The pipeline also caught a real bug: a file present locally but **not committed** fails in CI — proving CI validates the *repository*, not your laptop.

---

## 9. Container vs Virtual Machine

**VM** = virtualizes *hardware*; each VM runs a full **guest OS with its own kernel** on top of a hypervisor. **Container** = virtualizes the *operating system*; shares the **host kernel**, packaging only the app + deps.

Measured in the project:

| Metric | Container (app running) | VM (idle Ubuntu) |
|---|---|---|
| Memory in use | 96.12 MB | 350.9 MiB |
| Disk / image | 181 MB | 2.2 GiB |
| Start / boot time | ~0.44 s | ~19.18 s |
| Own OS kernel? | No (shares host) | Yes (full guest OS) |

**Why containers win on resources:** no second kernel to boot or keep in RAM, and only the app's files on disk. **Trade-off:** VMs give *stronger isolation* (separate kernels) — useful for untrusted workloads or different OSes; containers are lighter and faster but share the kernel.

How they were measured:
- Container: `podman stats --no-stream`, `podman images`, `time podman start`.
- VM: `multipass info <vm>` (memory/disk), `multipass exec <vm> -- systemd-analyze` (boot time).

---

## 10. Docker Swarm vs Kubernetes

Same concept, expressed two ways:

| Aspect | Docker Swarm | Kubernetes (k3s) |
|---|---|---|
| Setup | `docker swarm init` + join token; very quick | server + agents + token; more steps |
| Config format | one stack file (compose style) | several objects: Secret, Deployment, Service |
| Deploy | `docker stack deploy` | `kubectl apply -f` |
| One replica per node | `placement: max_replicas_per_node: 1` | `podAntiAffinity` (required) |
| External access on port 80 | published port + **routing mesh** | Service `type: LoadBalancer` |
| Self-healing | needs explicit `restart_policy` | **automatic by default** |
| Secrets | inline env vars | dedicated **Secret** object |
| Service discovery | service name on overlay net | service name via cluster DNS |
| Scaling | `docker service scale` | `kubectl scale deployment` |
| Learning curve | gentle | steeper, but far more control |

**Scaling behaviour observed:** with one replica per node and 3 nodes, both reach 3 replicas (one per node). A replica only goes **Pending** when the requested count **exceeds the number of eligible nodes** (e.g. 4 replicas on 3 nodes). To go higher: add a node, or relax the rule to a soft preference.

**One-line verdict:** *Both enforce the same guarantees; Swarm minimizes effort, Kubernetes maximizes control.* → **Swarm for simpler/smaller environments, Kubernetes for complex/production.**

---

## 11. Troubleshooting reference

| Problem | Symptom | Cause | Fix |
|---|---|---|---|
| App crashes on start | `Can't connect to MySQL ... 'CHANGEME'` or exits at boot | App calls `db.create_all()` before DB is ready | Start DB first & wait; on orchestrators rely on self-healing |
| `podman stats` fails | "stats not supported in rootless mode without cgroups v2" | cgroups v2 not active in WSL | Enable systemd in `/etc/wsl.conf`, `wsl --shutdown`, reopen → `cgroup2fs` |
| WSL won't reload config | still `tmpfs` after restart | Docker Desktop kept the WSL VM alive | Quit Docker Desktop fully, then `wsl --shutdown` |
| `wsl` "command not found" | error inside Ubuntu | `wsl` is a *Windows* command | Run it in PowerShell, not the Linux shell |
| pytest import error | `ModuleNotFoundError: converter` | repo root not on import path | add empty `conftest.py` at repo root |
| Can't push workflow file | "refusing... without `workflow` scope" | PAT lacks `workflow` scope | new token with `repo` **and** `workflow` |
| CI build fails | "no such file or directory: Dockerfile" | Dockerfile not committed to Git | `git add Dockerfile` + push |
| k3s workers won't join | stay absent from `get nodes` | only half the node-token used | use the FULL token (`K10...::server:...`) |
| Swarm workers `Down` after restart | "no suitable node (… not available)" | VM IPs changed; workers point at old manager IP | each worker `leave --force` then `join` at the manager's CURRENT IP; `node rm` stale entries |
| 3rd/4th replica `Pending` | scheduling stops short | one-replica-per-node rule + not enough nodes | add a node or relax the placement rule |

---

## 12. Likely exam questions & short answers

**Q: What is the difference between a container and a VM?**
A container shares the host OS kernel and packages just the app + dependencies (small, fast); a VM runs a full guest OS with its own kernel on a hypervisor (heavier, stronger isolation).

**Q: What two Linux kernel features make containers possible?**
Namespaces (isolation) and cgroups (resource limiting/accounting).

**Q: What is the difference between `EXPOSE` and `-p`/`ports`?**
`EXPOSE` only documents the port; publishing with `-p HOST:CONTAINER` (or `ports:`) actually maps it so traffic can reach the container.

**Q: How does an app reach the database by the name `db`?**
Service discovery — the orchestrator runs internal DNS, and on a shared (overlay) network a service's name resolves to it.

**Q: How are two replicas kept on different nodes?**
Swarm: `placement: max_replicas_per_node: 1`. Kubernetes: `podAntiAffinity` with `topologyKey: kubernetes.io/hostname` (required = hard rule).

**Q: How is the app exposed on port 80?**
Swarm: publish `80:5000`; the routing mesh load-balances across nodes. Kubernetes: a `Service` of `type: LoadBalancer` mapping `port: 80` → `targetPort: 5000`.

**Q: What is self-healing and how does each system do it?**
Automatically restarting/replacing failed containers. Swarm needs `restart_policy: on-failure`; Kubernetes restarts pods by default.

**Q: Why must the app connect as a non-root DB user?**
Least privilege — limits the damage if the app is compromised; the MySQL image auto-creates `appuser` via `MYSQL_USER`/`MYSQL_PASSWORD`.

**Q: Unit vs integration test?**
Unit = one function in isolation (no DB/server); integration = the whole app against a real database.

**Q: What does the CI pipeline do?**
On every push: runs unit tests, waits for a MySQL service, runs integration tests against it, builds the image. It validates the repository, not just the local machine.

**Q: Which orchestrator for which environment?**
Swarm for simpler/smaller setups (fast, one file); Kubernetes for complex/production (more control, default self-healing, huge ecosystem, portability).

**Q: When does a replica go `Pending`/unscheduled?**
When the requested replica count exceeds the number of nodes allowed by the placement/anti-affinity rule.

---

*End of reference. Tip during the exam: use Ctrl+F on the Table of Contents headings to jump straight to a tool or concept.*
