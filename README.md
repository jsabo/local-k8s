# kubeadm + Multipass demo cluster (Calico + local-path + Headlamp + Ingress + Whisker)

This repo builds a **reproducible local Kubernetes cluster** using **kubeadm** on **Multipass** VMs, then installs a small “demo stack”:

- **Calico** (via Tigera operator) for CNI + NetworkPolicy (and Whisker UI)
- **Rancher local-path-provisioner** for dynamic local PVs (demo-friendly default `StorageClass`)
- **Headlamp** (Kubernetes UI) exposed through **ingress-nginx (NodePort)**
- **Calico Whisker** exposed through **ingress-nginx (NodePort)**

Designed to be:
- **Easy to run** (single `task` command)
- **Reproducible** (idempotent-ish tasks; state in `./.state`)
- **Demo-friendly** (prints URLs, tokens, and `/etc/hosts` mapping)
- **Composable** (cluster lifecycle separated from app installs)

---

## Deliverables

Reviewers should be able to:

- **Reproduce the cluster + demo stack** from scratch: `task demo:install`
- **Inspect rendered artifacts** in `./.state/` (cloud-init, kubeadm config, audit policy, etc.)
- **Access UIs** via host-based routing:
  - Headlamp (token auth)
  - Whisker (Calico UI)

Primary artifacts:
- `README.md` (this document)
- `Taskfile.yaml` (automation entrypoint)
- `cloud-init.tftpl` (VM bootstrap)
- `kubeadm-config.tftpl` (kubeadm config)
- `audit-policy.tftpl` (apiserver audit policy)
- `calico-custom-resources.tftpl` (operator CRs: Installation/APIServer/Goldmane/Whisker)
- `headlamp.tftpl` (ClusterRoleBinding template)
- `ingress.tftpl` (Ingress + NetworkPolicy template)

---

## What this project includes

### Cluster
- 3-node Kubernetes cluster:
  - 1 control-plane (`cp-0`)
  - 2 workers (`worker-0`, `worker-1`)
- Built with:
  - **kubeadm**
  - **containerd**
  - **Calico** (via operator)
- Local state output:
  - `./.state/kubeconfig` (admin kubeconfig for cluster operations)
  - `./.state/env` (helper to export `KUBECONFIG`)
  - `./.state/*.yaml` (rendered cloud-init + config/manifests)
  - `./.state/.<component>*.installed` (stamp files for idempotence)

### Storage
- **Rancher local-path-provisioner**
  - Provides a default `StorageClass` (`local-path`) suitable for demos

### UIs / Demo Apps
- **Headlamp** (Helm install)
  - Token auth using a ServiceAccount (default is cluster-admin for demo convenience; configurable)
- **Calico Whisker** (enabled via Calico operator CR)
- **Ingress-NGINX (NodePort)** + host-based routing:
  - `headlamp.<base>` → Headlamp UI (HTTP NodePort)
  - `whisker.<base>` → Whisker UI (HTTP NodePort)

---

## Architecture overview

### Components and flow

1. Task renders templates into `./.state/` (audit policy, kubeadm config, cloud-init, Calico CRs, ingress rules)
2. Multipass launches 3 Ubuntu VMs using the rendered `cloud-init.yaml`
3. `kubeadm init` runs on `cp-0`
4. Workers join using a token generated on `cp-0`
5. Calico installs cluster networking (operator + custom resources, including Whisker)
6. local-path-provisioner installs default storage
7. Headlamp installs via Helm + ClusterRoleBinding
8. ingress-nginx installs via Helm, then applies ingress rules for Headlamp + Whisker
9. `task demo:info` prints URLs, Headlamp token, and `/etc/hosts` mappings

### Conceptual access path (Ingress via NodePort)

```mermaid
flowchart LR
  L[Laptop]
  H["/etc/hosts<br/>headlamp.&lt;base&gt; → &lt;cp-0 IP&gt;<br/>whisker.&lt;base&gt; → &lt;cp-0 IP&gt;"]
  L --> H

  HTTP_H["http://headlamp.&lt;base&gt;:&lt;nodeport-http&gt;/"]
  HTTP_W["http://whisker.&lt;base&gt;:&lt;nodeport-http&gt;/"]
  L --> HTTP_H
  L --> HTTP_W

  IC["ingress-nginx Controller<br/>(Service: NodePort)"]
  HTTP_H --> IC
  HTTP_W --> IC

  HL_SVC["Service: Headlamp<br/>ns: kube-system"]
  WK_SVC["Service: whisker<br/>ns: calico-system"]

  IC -->|host: headlamp.&lt;base&gt;| HL_SVC
  IC -->|host: whisker.&lt;base&gt;| WK_SVC
````

### 3-node cluster networking (Calico overlay)

```mermaid
%%{init: {"flowchart": {"rankSpacing": 80, "nodeSpacing": 30, "curve": "basis"}}}%%
flowchart TB

  subgraph CP["cp-0 (control-plane)"]
    direction TB
    CP_APISERVER["kube-apiserver"]
    CP_ETCD["etcd"]
    CP_CM["kube-controller-manager"]
    CP_SCHED["kube-scheduler"]
    CP_KUBELET["kubelet"]
    CP_PROXY["kube-proxy"]
    CP_CALICO["calico-node<br/>(VXLAN + policy)"]
  end

  subgraph W0["worker-0"]
    direction TB
    W0_KUBELET["kubelet"]
    W0_PROXY["kube-proxy"]
    W0_CALICO["calico-node<br/>(VXLAN + policy)"]
  end

  subgraph W1["worker-1"]
    direction TB
    W1_KUBELET["kubelet"]
    W1_PROXY["kube-proxy"]
    W1_CALICO["calico-node<br/>(VXLAN + policy)"]
  end

  subgraph OP["Operator / Calico system namespaces"]
    direction TB
    OP_TIG["tigera-operator<br/>(ns: tigera-operator)"]
    OP_CALICO["calico-kube-controllers<br/>(ns: calico-system)"]
    OP_GOLD["goldmane<br/>(ns: calico-system)"]
    OP_WHISK["whisker<br/>(ns: calico-system)"]
  end

  CP_APISERVER --> CP_ETCD
  CP_CM --> CP_APISERVER
  CP_SCHED --> CP_APISERVER
  CP_KUBELET --> CP_APISERVER

  W0_KUBELET --> CP_APISERVER
  W1_KUBELET --> CP_APISERVER

  CP_PROXY --> CP_APISERVER
  W0_PROXY --> CP_APISERVER
  W1_PROXY --> CP_APISERVER

  CP_CALICO --> CP_APISERVER
  W0_CALICO --> CP_APISERVER
  W1_CALICO --> CP_APISERVER

  OP_TIG --> CP_APISERVER
  OP_CALICO --> CP_APISERVER
  OP_GOLD --> CP_APISERVER
  OP_WHISK --> CP_APISERVER

  OVERLAY["Calico overlay<br/>(VXLAN between nodes)"]
  POLICY["NetworkPolicy enforcement<br/>(calico-node)"]
  SERVICES["Service networking<br/>(kube-proxy iptables/IPVS)"]

  CP_CALICO --> OVERLAY
  W0_CALICO --> OVERLAY
  W1_CALICO --> OVERLAY

  CP_CALICO --> POLICY
  W0_CALICO --> POLICY
  W1_CALICO --> POLICY

  CP_PROXY --> SERVICES
  W0_PROXY --> SERVICES
  W1_PROXY --> SERVICES
```

---

## Security model (demo defaults)

Headlamp is installed with a ServiceAccount you can use for token login:

* `HEADLAMP_SA` default: `headlamp-admin`
* `HEADLAMP_CLUSTERROLE` default: `cluster-admin` (convenient for demos)
* `HEADLAMP_TOKEN_DURATION` default: `1h`

You can tighten this by overriding variables when running tasks, for example:

```bash
HEADLAMP_CLUSTERROLE=view HEADLAMP_TOKEN_DURATION=15m task demo:install
```

> Note: Headlamp permissions are determined entirely by the ClusterRoleBinding rendered from `headlamp.tftpl`.

---

## Repository layout

* `Taskfile.yaml` — automation entrypoint (`task demo:install`)
* `cloud-init.tftpl` — provisions VMs (containerd + kubelet/kubeadm/kubectl + audit policy + kubeadm config)
* `audit-policy.tftpl` — API server audit policy (tokens/secrets protected; noisy reads reduced)
* `kubeadm-config.tftpl` — kubeadm init configuration (service/pod CIDRs + audit config)
* `calico-custom-resources.tftpl` — Calico operator CRs (Installation/APIServer/Goldmane/Whisker)
* `headlamp.tftpl` — ClusterRoleBinding for the Headlamp ServiceAccount
* `ingress.tftpl` — Ingress objects for Headlamp + Whisker and a NetworkPolicy for Whisker
* `./.state/` — generated state (kubeconfig, rendered YAML, stamps)

  * Safe to delete (regenerates on install)
  * **Do not commit** (contains kubeconfig and other local artifacts)

---

## Prerequisites

You need these on your laptop:

* [multipass](https://canonical.com/multipass/install)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)
* [helm](https://helm.sh/docs/intro/install/)
* [task](https://taskfile.dev/docs/installation) (Taskfile runner)
* `envsubst` (from gettext; used to render templates)

---

## Quickstart

### 1) Install everything (cluster + demo stack)

```bash
task demo:install
```

This will:

* create VMs
* init kubeadm
* install Calico
* install local-path-provisioner
* install Headlamp
* install ingress-nginx + host-based rules
* print URLs + a Headlamp token + `/etc/hosts` mappings

### 2) Add the host mappings

Run:

```bash
task demo:info
```

It prints lines like:

```
192.168.x.y headlamp.local.k8s
192.168.x.y whisker.local.k8s
```

Add them to `/etc/hosts` on your laptop.

### 3) Browse

* Headlamp: `http://headlamp.<base>:<nodeport-http>/`
* Whisker:  `http://whisker.<base>:<nodeport-http>/`

Defaults:

* base domain: `local.k8s`
* NodePort HTTP: `30080`

So by default:

* Headlamp: `http://headlamp.local.k8s:30080/`
* Whisker:  `http://whisker.local.k8s:30080/`

---

## Smoke test (30 seconds)

```bash
export KUBECONFIG=./.state/kubeconfig

kubectl get nodes -o wide
kubectl get pods -A | head

# confirm ingress service is NodePort and has the expected ports
kubectl -n ingress-nginx get svc ingress-nginx-controller -o wide

# confirm Headlamp + Whisker ingress objects exist
kubectl -n kube-system get ingress headlamp
kubectl -n calico-system get ingress whisker
```

---

## Demo flow (copy/paste)

### 1) Show cluster health

```bash
export KUBECONFIG=./.state/kubeconfig
kubectl get nodes -o wide
kubectl get pods -A
```

### 2) Show the rendered config artifacts (local)

```bash
ls -la ./.state/
sed -n '1,80p' ./.state/kubeadm-config.yaml
sed -n '1,120p' ./.state/audit-policy.yaml
```

### 3) Print URLs + Headlamp token

```bash
task demo:info
```

---

## Common operations

```bash
task                 # list tasks
task cluster:status  # check cluster status (kubectl get nodes)
task demo:info       # print URLs + token + /etc/hosts lines
task demo:uninstall  # uninstall demo apps (keeps cluster running)
task cluster:down    # delete everything (VMs + local state)
task cluster:purge   # delete everything + multipass purge
```

---

## Idempotence and stamp files

The repo uses “stamp files” under `./.state/` to avoid repeating slow/noisy installs:

* `.calico-<version>.installed`
* `.local-path-<version>.installed`
* `.ingress-nginx-<ports>.installed`
* `.headlamp.installed`

This makes repeat runs faster and less chatty, but it also means:

* if you manually delete a component from the cluster, you may want to delete the corresponding stamp file (or `rm -rf ./.state/`) so the install task reruns cleanly.

---

## Troubleshooting

### Hostnames do not resolve

Run:

```bash
task demo:info
```

Add the printed mappings to `/etc/hosts`.

### Ingress works but Whisker doesn’t load

Check that the Whisker pods/services exist and the NetworkPolicy isn’t blocking ingress-nginx:

```bash
export KUBECONFIG=./.state/kubeconfig
kubectl -n calico-system get pods,svc
kubectl -n calico-system get networkpolicy whisker-ingress-nginx -o yaml
kubectl -n calico-system describe ingress whisker
```

### Headlamp token login fails

Recreate a token and try again:

```bash
export KUBECONFIG=./.state/kubeconfig
kubectl -n kube-system create token headlamp-admin --duration=1h
```

If you changed `HEADLAMP_NS` / `HEADLAMP_SA`, use those values.

### Want a clean rebuild

```bash
task cluster:down
task demo:install
```

---

## License / usage

This repository is intended as a learning/demo environment. Use it freely and adapt as needed.

