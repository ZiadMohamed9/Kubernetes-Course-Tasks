# Task 1: Multi-container Pods (Init Container & Sidecar)

Run an nginx "hello" page on a 3-node minikube cluster. The page shows the current time and updates every 5 seconds:

```
hello in MLOPS MENA COMMUNITY : 2026-11-06 17:15:05
```

Each Pod has **three containers and no volume**, then we break the init container on purpose to see how Kubernetes reacts.

---

## Table of Contents

1. [Concepts in 2 minutes](#1-concepts-in-2-minutes)
2. [Architecture](#2-architecture)
3. [Tools installed](#3-tools-installed)
4. [Commands, step by step](#4-commands-step-by-step)
5. [The deployment file, line by line](#5-the-deployment-file-line-by-line)
6. [The failure experiment (Step 5)](#6-the-failure-experiment-step-5)
7. [Verification checklist](#7-verification-checklist)
8. [Troubleshooting I hit](#8-troubleshooting-i-hit)
9. [Clean up](#9-clean-up)
10. [Key takeaways](#10-key-takeaways)

---

## 1. Concepts in 2 minutes

| Term | Meaning |
|---|---|
| **Cluster** | A group of machines (nodes) managed by Kubernetes. |
| **Node** | One machine in the cluster. With minikube each node is a Docker container. |
| **Pod** | The smallest unit Kubernetes runs. Holds one or more containers that share network (and here, processes). |
| **Deployment** | Keeps N identical Pods running and handles updates (rolling updates). |
| **ReplicaSet** | Created by the Deployment. Each version of the Pod template gets its own ReplicaSet. |
| **Namespace** | A logical folder to group resources (here: `webapp`). |
| **Init container** | Runs **before** the main containers, must finish successfully. If it fails, the Pod never starts its main containers. |
| **Sidecar** | A helper container running next to the main one in the same Pod. |
| **Label / selector** | Key-value tags (`app: web`). The Deployment uses a selector to find the Pods it owns. |

---

## 2. Architecture

```
                     Deployment: web-deployment (3 replicas)
                                   |
        +--------------------------+--------------------------+
        |                          |                          |
      Pod 1                      Pod 2                      Pod 3
 +----------------+        +----------------+        +----------------+
 | init-done      |        |      ...       |        |      ...       |
 |  (runs first,  |        |                |        |                |
 |   exits)       |        |                |        |                |
 |       |        |        |                |        |                |
 |       v        |        |                |        |                |
 | nginx  <-------+--+     |                |        |                |
 | (serves :80)   |  |     |                |        |                |
 | time-updater --+--+     |                |        |                |
 | (writes file   |        |                |        |                |
 |  every 5 s via |        |                |        |                |
 |  /proc/PID/    |        |                |        |                |
 |  root)         |        |                |        |                |
 +----------------+        +----------------+        +----------------+
   shareProcessNamespace: true  (all containers see each other's processes)
```

**How the sidecar edits nginx's file without a shared volume:**
1. `shareProcessNamespace: true` puts all containers of the Pod in the same process namespace, so `time-updater` can see the nginx process.
2. Linux exposes every process's filesystem at `/proc/<PID>/root`.
3. The sidecar writes to `/proc/<nginx-pid>/root/usr/share/nginx/html/index.html`, which is the file *inside* the nginx container.
4. Reaching into another container's `/proc/<PID>/root` needs the `SYS_PTRACE` capability, so we add it to the sidecar.

---

## 3. Tools installed

| Tool | Purpose |
|---|---|
| **Docker** | Container engine. minikube uses it to run the cluster nodes as containers (no VM needed). |
| **minikube** | Creates a local Kubernetes cluster. |
| **kubectl** | CLI to talk to the cluster. |
| **Git** | Version control, used for submission. |

Fedora install commands:

```bash
sudo dnf install -y git

# Docker (if podman-docker is installed, remove it first: it conflicts with docker-ce)
sudo dnf remove -y podman-docker
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager addrepo --from-repofile=https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER      # then log out and back in

# minikube
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
```

| Command part | Explanation |
|---|---|
| `dnf remove -y podman-docker` | `-y` answers "yes" automatically. Removes only the `docker` -> Podman shim. |
| `systemctl enable --now docker` | `enable` starts Docker on every boot; `--now` also starts it right away. |
| `usermod -aG docker $USER` | `-a` append, `-G` to a supplementary group. Lets you run `docker` without `sudo`. |
| `curl -LO <url>` | `-L` follow redirects, `-O` save with the remote file name. |
| `install -m 0755` | Copies the binary into `/usr/local/bin` with executable permissions (`rwxr-xr-x`). |

> 📸 Screenshot: `docker --version`, `minikube version`, `kubectl version --client`

---

## 4. Commands, step by step

### Step 1: Create the cluster

```bash
minikube start --nodes 3 --driver=docker --cni=calico --cpus 2 --memory 2200
kubectl get nodes
```

| Option | Why |
|---|---|
| `--nodes 3` | 1 control-plane + 2 workers (3 nodes total), as the task requires. |
| `--driver=docker` | Run nodes as Docker containers instead of VMs. |
| `--cni=calico` | Installs the **Calico** network plugin. Later tasks need it (e.g. network policies). |
| `--cpus 2` | 2 CPUs **per node**. |
| `--memory 2200` | 2200 MB RAM **per node**. It is a limit, not a reservation. |

`kubectl get nodes` lists the nodes. Expected: `minikube`, `minikube-m02`, `minikube-m03` all `Ready`.

> 📸 Screenshot: `kubectl get nodes`

### Step 2: Create the namespace

```bash
kubectl create namespace webapp
```

Creates the `webapp` namespace. Everything in this task lives there so it is easy to clean up (`kubectl delete namespace webapp`).

### Step 3: Deploy

```bash
kubectl apply -f deployment.yaml
kubectl get pods -n webapp -w
```

| Command / option | Explanation |
|---|---|
| `kubectl apply` | **Declarative**: "make the cluster match this file". Creates the resource if new, updates it if it exists. |
| `-f deployment.yaml` | `-f` = file to read. |
| `kubectl get pods` | Lists Pods. |
| `-n webapp` | `-n` = namespace. Without it kubectl looks in `default`. |
| `-w` | `--watch`: keeps streaming changes live. Press `Ctrl+C` to stop. |

Expected: 3 Pods, each `2/2 Running`. It shows **2/2** and not 3/3 because the init container is finished and only `nginx` and `time-updater` count.

> 📸 Screenshot: `kubectl get pods -n webapp` showing 3 x `2/2 Running`

### Step 4: Open the page

```bash
kubectl port-forward -n webapp deploy/web-deployment 8080:80
```

| Part | Explanation |
|---|---|
| `port-forward` | Tunnels a port on your machine to a port inside the cluster. |
| `deploy/web-deployment` | Target: a Pod picked from this Deployment. |
| `8080:80` | `local-port:pod-port`. Your `localhost:8080` -> container port `80`. |

The command keeps running, so use a **second terminal** for other commands. Open `http://localhost:8080` and refresh after 5 seconds: the time changes.

> 📸 Screenshot: the browser showing the page, twice with different times

### Step 5: Verification commands

```bash
kubectl get nodes
kubectl get deploy web-deployment -n webapp
kubectl get pods -n webapp -l app=web
kubectl get pods -n webapp -l app=web -o jsonpath='{.items[0].spec.initContainers[*].name} {.items[0].spec.containers[*].name}'
kubectl logs deploy/web-deployment -n webapp -c init-done
kubectl exec deploy/web-deployment -n webapp -c time-updater -- ps
kubectl exec deploy/web-deployment -n webapp -c nginx -- cat /usr/share/nginx/html/index.html
kubectl get deploy web-deployment -n webapp -o jsonpath='{.spec.template.spec.volumes}'
```

| Option | Explanation |
|---|---|
| `-l app=web` | `--selector`: only Pods with label `app=web`. |
| `-o jsonpath='...'` | Print only the fields you select from the resource's JSON. `[*]` means "all items". |
| `logs ... -c init-done` | `-c` = which **container** inside the Pod (needed when a Pod has several). |
| `exec ... -c <name> -- <cmd>` | Runs a command inside a container. Everything after `--` is the command, not a kubectl option. |
| `ps` (in time-updater) | Shows `nginx: master process`, proving the sidecar sees nginx's process (`shareProcessNamespace`). |
| `cat index.html` (twice) | Run it twice, 5 seconds apart, to prove the time updates. |

Expected results:

| # | Check | Expected |
|---|---|---|
| 1 | 3 nodes | all `Ready` |
| 2 | Deployment | `READY 3/3` |
| 3 | Pods | 3 Pods, each `2/2 Running` |
| 4 | Container names | `init-done nginx time-updater` |
| 5 | Init logs | `done` |
| 6 | `ps` | `nginx: master process` in the list |
| 7 | `cat` twice | new time each time |
| 8 | Volumes | empty output |

> 📸 Screenshots: one per command above

---

## 5. The deployment file, line by line

```yaml
apiVersion: apps/v1
```
API group and version used for Deployments. Every Kubernetes object starts with this.

```yaml
kind: Deployment
```
The type of object. A Deployment manages ReplicaSets, which manage Pods.

```yaml
metadata:
  name: web-deployment
  namespace: webapp
```
`metadata` identifies the object. `name` is its name; `namespace` places it in `webapp`.

```yaml
spec:
  replicas: 3
```
`spec` is the **desired state**. `replicas: 3` means "always keep 3 Pods running". If one dies, Kubernetes creates a replacement.

```yaml
  selector:
    matchLabels:
      app: web
```
How the Deployment finds the Pods it owns: Pods labeled `app: web`. **Must match** the Pod template labels below, otherwise the Deployment is rejected.

```yaml
  template:
    metadata:
      labels:
        app: web
```
`template` is the blueprint for each Pod. `labels` tags every Pod with `app: web` so the selector above can find them.

```yaml
    spec:
      shareProcessNamespace: true
```
This `spec` is the **Pod** spec. `shareProcessNamespace: true` makes all containers share one process namespace, so the sidecar can see the nginx process. It belongs at Pod level (next to `containers`), not inside a container.

```yaml
      initContainers:
        - name: init-done
          image: busybox:1.36
          command: ["sh", "-c", "echo done"]
```
- `initContainers`: run **in order, to completion, before** the main containers.
- `name`: container name (used with `-c` in kubectl).
- `image: busybox:1.36`: a tiny Linux image with basic tools. A pinned version keeps results reproducible.
- `command`: runs `sh -c "echo done"`. `sh -c` means "run this string as a shell command". It prints `done` and exits with code 0 (success). In Step 5 this becomes `echo 'init failed'; exit 1`, which exits with code 1 (failure).

```yaml
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```
The main container. `containerPort: 80` documents that nginx listens on port 80 (informational; it doesn't open the port by itself).

```yaml
          command: ["/bin/sh", "-c"]
```
Overrides the image's default entrypoint (the nginx image normally starts via its own entrypoint script). We start a shell instead so we can run two commands.

```yaml
          args:
            - |
              echo "<h1>hello in MLOPS MENA COMMUNITY : $(date '+%Y-%m-%d %H:%M:%S')</h1>" > /usr/share/nginx/html/index.html
              nginx -g 'daemon off;'
```
`args` is the string passed to `sh -c`. The `|` is a YAML **block scalar**: it keeps the following lines as multi-line text.
- Line 1: writes the HTML with the current time (`$(date ...)`) into nginx's web root, `/usr/share/nginx/html/index.html`.
- Line 2: starts nginx. `-g 'daemon off;'` keeps nginx in the **foreground**. This is required: if nginx went to the background, the shell would end and the container would exit.

```yaml
        - name: time-updater
          image: busybox:1.36
```
The sidecar container.

```yaml
          securityContext:
            capabilities:
              add: ["SYS_PTRACE"]
```
Adds the Linux capability `SYS_PTRACE`. Without it the sidecar is not allowed to access another process's `/proc/<PID>/root` (the nginx container's filesystem).

```yaml
          command: ["/bin/sh", "-c"]
          args:
            - |
              while true; do
                PID=$(pgrep -o nginx)
                if [ -n "$PID" ]; then
                  echo "<h1>hello in MLOPS MENA COMMUNITY : $(date '+%Y-%m-%d %H:%M:%S')</h1>" \
                    > /proc/$PID/root/usr/share/nginx/html/index.html
                fi
                sleep 5
              done
```
| Line | Meaning |
|---|---|
| `while true; do ... done` | Infinite loop. |
| `PID=$(pgrep -o nginx)` | Find the **oldest** (`-o`) process named nginx, i.e. the master process. Possible only because processes are shared. |
| `if [ -n "$PID" ]; then` | Only continue if a PID was found (`-n` = non-empty). Avoids writing before nginx has started. |
| `echo "<h1>...</h1>" \ > /proc/$PID/root/...` | Writes the new HTML into nginx's filesystem through `/proc/<PID>/root`. The trailing `\` continues the command on the next line. |
| `fi` | Ends the `if`. |
| `sleep 5` | Wait 5 seconds, then loop again. |

**Note:** the file has **no `volumes:` section**. That is the point of the task: the sidecar and nginx share data through `/proc`, not through a shared volume.

---

## 6. The failure experiment (Step 5)

Change **only** the init container command:

```yaml
command: ["sh", "-c", "echo 'init failed'; exit 1"]
```

Apply it again (no delete) and observe:

```bash
kubectl apply -f deployment.yaml
kubectl get pods -n webapp -w
kubectl logs <new-pod> -n webapp -c init-done
kubectl describe pod <new-pod> -n webapp
kubectl rollout status deploy/web-deployment -n webapp
```

| Command | Why |
|---|---|
| `kubectl apply -f` again | Changing the Pod template creates a **new ReplicaSet** and starts a **rolling update**. |
| `get pods -w` | Watch the new Pod fail live. |
| `logs <pod> -c init-done` | Show what the init container printed: `init failed`. |
| `describe pod` | Detailed view: container states, exit codes, restart count, and events. |
| `rollout status` | Waits for the rollout to finish. It never finishes here (it waits up to 10 minutes), so stop it with `Ctrl+C`. |

### What I observed

- `kubectl logs` printed `init failed`.
- `describe` showed `init-done`: `State: Terminated`, `Reason: Error`, `Exit Code: 1`, `Restart Count` increasing.
- Events showed `Back-off restarting failed container init-done`.
- `nginx` and `time-updater` stayed in `Waiting` with reason `PodInitializing`.
- Conditions: `Initialized: False`, `Ready: False`.

### Answers

**Which status does the new Pod show?**
`Init:Error`, then `Init:CrashLoopBackOff`, with READY `0/2`. The init container keeps failing, and Kubernetes restarts it with increasing delays (10s, 20s, 40s, ... up to 5 minutes). `describe` shows `Status: Pending` because the Pod never got past initialization.

**Do `nginx` and `time-updater` start?**
No. Kubernetes only starts the main containers after **all** init containers succeed. They stay in `PodInitializing`.

**Is the page still reachable, and why?**
Yes. A Deployment uses a **rolling update** strategy (default `maxSurge: 25%`, `maxUnavailable: 25%`). With 3 replicas this means: create 1 new Pod first, and remove an old Pod only when the new one is **Ready**. The new Pod never becomes Ready, so the 3 old Pods (from the previous working ReplicaSet) keep running and serving traffic. The failed update is stuck, but nothing breaks for users.

> 📸 Screenshots: `get pods` (new Pod `Init:CrashLoopBackOff` + 3 old Pods `2/2`), `logs`, `describe` (Exit Code: 1), the page still loading in the browser

### Step 6: Fix it

Put the original command back (`echo done`) and apply:

```bash
kubectl apply -f deployment.yaml
kubectl get pods -n webapp -w
kubectl get deploy web-deployment -n webapp
```

The Deployment moves to a new working ReplicaSet, the stuck Pod is replaced, and the result is `READY 3/3`, `UP-TO-DATE 3`.

> 📸 Screenshot: `kubectl get deploy web-deployment -n webapp`

---

## 7. Verification checklist

| # | Check | Command | Expected | Done |
|---|---|---|---|---|
| 1 | 3 nodes | `kubectl get nodes` | 3 nodes Ready | ☐ |
| 2 | Deployment ready | `kubectl get deploy web-deployment -n webapp` | READY 3/3 | ☐ |
| 3 | Pods running | `kubectl get pods -n webapp -l app=web` | 3 x 2/2 Running | ☐ |
| 4 | Containers | jsonpath command (section 4) | `init-done nginx time-updater` | ☐ |
| 5 | Init succeeded | `kubectl logs deploy/web-deployment -n webapp -c init-done` | `done` | ☐ |
| 6 | Sidecar sees nginx | `kubectl exec ... -c time-updater -- ps` | `nginx: master process` | ☐ |
| 7 | Time updates | `kubectl exec ... -c nginx -- cat ...` (x2) | new time | ☐ |
| 8 | No volumes | jsonpath on `.spec.template.spec.volumes` | empty | ☐ |
| 9 | Failing init | `kubectl get pods -n webapp -l app=web` | new Pod `Init:CrashLoopBackOff`, 0/2; old Pods 2/2 | ☐ |
| 10 | Init error details | `kubectl describe pod <new-pod> -n webapp` | `Exit Code: 1` | ☐ |
| 11 | Fixed | `kubectl get deploy web-deployment -n webapp` | READY 3/3, UP-TO-DATE 3 | ☐ |

---

## 8. Troubleshooting I hit

| Problem | Cause | Fix |
|---|---|---|
| `podman-docker conflicts with docker-ce` | Fedora ships a shim package that fakes the `docker` command | `sudo dnf remove -y podman-docker`, then install `docker-ce` |
| `Unit docker.service does not exist` / `group 'docker' does not exist` | Docker was never installed because of the conflict above | Install Docker first, then enable the service and add the group |
| `cni: stat: stat calcio: no such file or directory` | Typo: `--cni=calcio` instead of `calico` | Use `--cni=calico` |
| `kubectl ... localhost:8080 connection refused` | No cluster was running (minikube start failed) | Start the cluster, then retry |

Tip: `minikube delete --all` wipes a broken cluster so you can start clean.

---

## 9. Clean up

Keep the cluster for Task 2. Remove only this task's resources:

```bash
kubectl delete namespace webapp
```

Deleting a namespace deletes everything inside it (Deployment, ReplicaSets, Pods). To remove the whole cluster instead: `minikube delete --all`.

---

## 10. Key takeaways

- **Init containers** gate the start of a Pod: if one fails, the main containers never run.
- **Sidecars** run next to the main container; with `shareProcessNamespace` and `SYS_PTRACE` they can reach its filesystem via `/proc/<PID>/root`, with no volume.
- **Rolling updates** protect availability: a broken new version never replaces working Pods.
- `kubectl describe` and `kubectl logs -c <container>` are the first tools for debugging a failing Pod.
