# kubeadm + Multipass demo cluster (WordPress + RBAC + CSR users + Dashboard + Ingress)

This repo builds a **reproducible local Kubernetes cluster** using **kubeadm** on **Multipass** VMs and deploys a **custom WordPress + MySQL Helm chart** with **persistent storage** and **minimum-privilege user access**.

Designed to be:
- **Easy to run** (single `task` command)
- **Reproducible** (idempotent tasks, local state in `./.state`)
- **Security-focused** (namespace RBAC + CSR-based user credentials)
- **Demo-friendly** (prints URLs, tokens, and `/etc/hosts` mapping)

---

## Deliverables

Reviewers should be able to:
- **Reproduce the cluster and app install** from scratch (`task demo:install`)
- **Validate least-privilege access** (deployer vs accessor)
- **Understand key tradeoffs** (CSR auth, local storage, bootstrap privileges)

Primary artifacts:
- `README.md` (this document)
- `Taskfile.yaml` (automation entrypoint)
- `charts/custom-wordpress/` (custom Helm chart)
- `rbac/` (namespace RBAC manifests)
- `docs/design.md` (design overview + tradeoffs)

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
  - `./.state/users/` (generated user kubeconfigs; **do not commit**)

### Storage
- **Rancher local-path-provisioner** for dynamic local persistent volumes
  - Provides a default `StorageClass` (`local-path`) suitable for demos

### Workloads
- **Custom WordPress Helm chart** (`charts/custom-wordpress`)
  - WordPress Deployment + Service
  - MySQL Deployment + Service
  - PVCs for WordPress and MySQL
  - Secret with generated credentials (preserved across upgrades)
- **Two user types** (namespace-scoped):
  - **Deployer**: install/upgrade/uninstall the WordPress release in a namespace
  - **Accessor**: read basics + port-forward to access WordPress
- **Kubernetes Dashboard** (Helm install) with a read-only `view` token for convenience
- **Ingress-NGINX (NodePort)** + host-based routing:
  - `wp.<base>` → WordPress
  - `api.<base>` → stub API service (`traefik/whoami`)
  - `dashboard.<base>` → Kubernetes Dashboard (HTTPS backend)

---

## Architecture overview

### Components and flow
1. Multipass launches 3 Ubuntu VMs using `cloud-init.tftpl`
2. `kubeadm init` runs on `cp-0`
3. Workers join using a token generated on `cp-0`
4. Calico installs cluster networking
5. local-path-provisioner installs default storage
6. Namespace + RBAC + CSR user kubeconfigs are created
7. WordPress is installed using the **deployer** kubeconfig
8. Access is performed using:
   - Ingress hostnames (recommended), and/or
   - Accessor kubeconfig + port-forward

### Conceptual networking

```mermaid
flowchart LR
  L[Laptop]
  H["/etc/hosts<br/>wp.demo.test → &lt;cp-0 IP&gt;<br/>api.demo.test → &lt;cp-0 IP&gt;<br/>dashboard.demo.test → &lt;cp-0 IP&gt;"]
  L --> H

  HTTP_WP["http://wp.demo.test:30080/<br/>Ingress-NGINX NodePort HTTP"]
  HTTP_API["http://api.demo.test:30080/<br/>Ingress-NGINX NodePort HTTP"]
  HTTPS_DASH["https://dashboard.demo.test:30443/<br/>Ingress-NGINX NodePort HTTPS"]
  L --> HTTP_WP
  L --> HTTP_API
  L --> HTTPS_DASH

  IC["Ingress-NGINX Controller"]
  HTTP_WP --> IC
  HTTP_API --> IC
  HTTPS_DASH --> IC

  WP_SVC["Service: wordpress<br/>ns: wordpress-demo"]
  API_SVC["Service: whoami<br/>ns: wordpress-demo"]
  DB_SVC["Service: dashboard<br/>ns: kubernetes-dashboard"]

  IC -->|host: wp.demo.test| WP_SVC
  IC -->|host: api.demo.test| API_SVC
  IC -->|host: dashboard.demo.test| DB_SVC

  WP_POD["Pods: wordpress"]
  API_POD["Pods: whoami"]
  DB_POD["Pods: dashboard"]

  WP_SVC --> WP_POD
  API_SVC --> API_POD
  DB_SVC --> DB_POD

  subgraph PF["Port-forward (accessor)"]
    direction LR
    K["kubectl port-forward"]
    APISERVER["Kubernetes API Server<br/>(cp-0)"]
    TARGET["Pod/Service target<br/>(namespace-scoped RBAC)"]
    L --> K --> APISERVER --> TARGET
  end
````

### 3-node cluster networking (Calico overlay)

Basic configuration (typical kubeadm defaults unless overridden):

* **Pod CIDR:** `10.48.0.0/16`
* **Service CIDR:** `10.49.0.0/16`
* **Overlay:** **Calico VXLAN** encapsulation between nodes (for pod-to-pod across nodes)
* **Service routing:** `kube-proxy` programs iptables/IPVS rules for ClusterIP/NodePort services
* **Pod routing:** `calico-node` programs routes / encapsulation and enforces NetworkPolicy

```mermaid
%%{init: {"flowchart": {"rankSpacing": 90, "nodeSpacing": 40, "curve": "basis"}}}%%
flowchart TB

  %% ===== Cluster Nodes =====
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
    W0_PODS["Pods<br/>Pod CIDR: 10.48.0.0/16"]
  end

  subgraph W1["worker-1"]
    direction TB
    W1_KUBELET["kubelet"]
    W1_PROXY["kube-proxy"]
    W1_CALICO["calico-node<br/>(VXLAN + policy)"]
    W1_PODS["Pods<br/>Pod CIDR: 10.48.0.0/16"]
  end

  %% ===== Core control-plane wiring =====
  CP_APISERVER --> CP_ETCD
  CP_CM --> CP_APISERVER
  CP_SCHED --> CP_APISERVER
  CP_KUBELET --> CP_APISERVER

  W0_KUBELET --> CP_APISERVER
  W1_KUBELET --> CP_APISERVER
  W0_CALICO --> CP_APISERVER
  W1_CALICO --> CP_APISERVER
  CP_CALICO --> CP_APISERVER
  W0_PROXY --> CP_APISERVER
  W1_PROXY --> CP_APISERVER
  CP_PROXY --> CP_APISERVER

  %% ===== Networking “concept” boxes =====
  OVERLAY["Calico overlay<br/>VXLAN tunneling between nodes<br/>(pod-to-pod across nodes)"]
  SERVICES["Service networking<br/>Service CIDR: 10.49.0.0/16<br/>(ClusterIP / NodePort via kube-proxy iptables/IPVS)"]
  POLICY["NetworkPolicy enforcement<br/>(calico-node)"]

  %% Hook node components to concepts
  W0_CALICO --> OVERLAY
  W1_CALICO --> OVERLAY
  CP_CALICO --> OVERLAY

  W0_PROXY --> SERVICES
  W1_PROXY --> SERVICES
  CP_PROXY --> SERVICES

  W0_CALICO --> POLICY
  W1_CALICO --> POLICY
  CP_CALICO --> POLICY
```

---

## Security model (how access is controlled)

This repo intentionally demonstrates **minimum-privilege access** using a clear separation of duties.

### Who does what (boundaries)

* **Admin (bootstrap only)**

  * kubeadm init/join, CNI install, storage class install, ingress controller, dashboard install
  * approves CSRs
* **Deployer (namespace only)**

  * installs/upgrades/uninstalls WordPress via Helm
  * manages namespaced resources needed for the release
* **Accessor (namespace only)**

  * reads basic app resources and uses **port-forward**
  * cannot deploy or modify workloads

### Authentication (CSR)

User credentials are created via **Kubernetes CertificateSigningRequest (CSR)**:

* A private key + CSR is generated locally
* A CSR object is submitted to the cluster
* The CSR is approved (admin step)
* A kubeconfig is generated embedding the user cert and cluster CA

Generated kubeconfigs:

* `./.state/users/<namespace>/deployer.kubeconfig`
* `./.state/users/<namespace>/accessor.kubeconfig`

### Authorization (RBAC)

RBAC is namespace-scoped in `./rbac`:

* `wp-deployer` Role:

  * create/update/delete common namespaced objects needed for Helm
  * manage Deployments/ReplicaSets/Jobs/Ingresses/PVCs/Secrets/etc.
* `wp-accessor` Role:

  * read-only discovery (`get/list/watch`) for Pods/Services/Endpoints
  * `create` on `pods/portforward` (required for port-forward)
  * optional `get` on `pods/log` for troubleshooting

#### Quick “can / cannot” summary

* **Deployer can:** manage Deployments/Services/Ingress/PVC/Secrets in the namespace; Helm lifecycle
* **Deployer cannot:** list nodes; install cluster-scoped components; modify cluster-wide resources
* **Accessor can:** `get/list/watch` pods/svc/endpoints; `port-forward`; optionally view logs
* **Accessor cannot:** create deployments; edit secrets; change ingress; manage PVCs

> Note: Cluster-scoped components are installed with the admin kubeconfig only.

---

## Repository layout

* `Taskfile.yaml` — automation entrypoint (`task demo:install`)
* `cloud-init.tftpl` — provisions VMs (containerd + kubelet/kubeadm/kubectl)
* `charts/custom-wordpress/` — custom Helm chart
* `rbac/` — namespaced RBAC for deployer/accessor users
* `docs/` — design notes
* `./.state/` — generated state (kubeconfigs, rendered cloud-init, stamps)

  * Safe to delete (regenerates on install)
  * **Do not commit** (contains kubeconfigs, certs, tokens)

---

## Prerequisites

You need these on your laptop:

* [multipass](https://canonical.com/multipass/install)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)
* [helm](https://helm.sh/docs/intro/install/)
* [task](https://taskfile.dev/docs/installation) (Taskfile runner)
* `envsubst` (from gettext; used to render cloud-init)

---

## Quickstart

### 1) Install everything (cluster + apps)

```bash
task demo:install
```

This will:

* create VMs
* init kubeadm
* install Calico
* install local-path-provisioner
* install Dashboard
* create namespace + RBAC + CSR users
* install WordPress (using deployer credentials)
* install ingress-nginx + host-based rules
* print URLs + dashboard token + `/etc/hosts` mapping

### 2) Add the host mappings

Run:

```bash
task demo:info
```

It prints three lines like:

```
192.168.x.y wp.demo.test
192.168.x.y api.demo.test
192.168.x.y dashboard.demo.test
```

Add those to `/etc/hosts` on your laptop.

### 3) Browse

* WordPress:  `http://wp.demo.test:30080/`
* Go API:     `http://api.demo.test:30080/`
* Dashboard:  `https://dashboard.demo.test:30443/`

The Dashboard login token is printed by `task demo:info`.

---

## Smoke test (30 seconds)

```bash
export KUBECONFIG=./.state/kubeconfig
kubectl get nodes -o wide
kubectl -n wordpress-demo get pods,svc,pvc,ingress
curl -I http://wp.demo.test:30080/ | head -n 1
curl -I http://api.demo.test:30080/ | head -n 1
```

---

## Demo flow (copy/paste)

### 1) Show cluster health (admin)

```bash
export KUBECONFIG=./.state/kubeconfig
kubectl get nodes -o wide
kubectl get pods -A
```

### 2) Show deployer permissions (namespace lifecycle)

```bash
export KUBECONFIG=./.state/users/wordpress-demo/deployer.kubeconfig

kubectl -n wordpress-demo get deploy,svc,pvc,ingress

kubectl auth can-i create deployments -n wordpress-demo
kubectl auth can-i create secrets -n wordpress-demo
kubectl auth can-i list nodes
```

### 3) Show accessor permissions (read + port-forward)

```bash
export KUBECONFIG=./.state/users/wordpress-demo/accessor.kubeconfig

kubectl -n wordpress-demo get svc,pods

kubectl auth can-i create pods/portforward -n wordpress-demo
kubectl auth can-i create deployments -n wordpress-demo
```

### 4) Access WordPress via port-forward (accessor)

```bash
kubectl -n wordpress-demo port-forward svc/custom-wordpress 8080:80
# open http://127.0.0.1:8080
```

### 5) Optional: show a forbidden action (accessor)

```bash
kubectl -n wordpress-demo create deployment nope --image=nginx
```

---

## Manual: Create an unprivileged user kubeconfig via CSR (no Taskfile)

This project’s “deployer” and “accessor” users authenticate using **client certificates** issued via the Kubernetes **CertificateSigningRequest (CSR)** API. Authorization is provided by **namespace-scoped RBAC** in `./rbac`.

You’ll end up with a kubeconfig that:

* uses a client cert for authn
* defaults to a namespace context
* has only the RBAC permissions you bind (no cluster-admin)

### Prereqs

* Working **admin kubeconfig** (example: `./.state/kubeconfig`)
* Tools: `kubectl`, `openssl`, `base64`
* Target namespace (example: `wordpress-demo`)

> CSR approval requires cluster permissions. Run CSR creation/approval using an admin kubeconfig.

### Step 1) Set admin kubeconfig and namespace

```bash
export KUBECONFIG=./.state/kubeconfig
export NS=wordpress-demo
```

Ensure the namespace exists:

```bash
kubectl get ns "$NS" >/dev/null 2>&1 || kubectl create ns "$NS"
```

### Step 2) Apply RBAC (Role + RoleBinding)

This repo ships two patterns:

* `wp-deployer` Role bound to group `wp-deployers`
* `wp-accessor` Role bound to group `wp-accessors`

Apply both:

```bash
kubectl -n "$NS" apply -f rbac/10-role-wp-deployer.yaml \
  -f rbac/11-rolebinding-wp-deployer.yaml \
  -f rbac/20-role-wp-accessor.yaml \
  -f rbac/21-rolebinding-wp-accessor.yaml
```

If you want user-based binding instead of groups, edit the RoleBinding manifests and use `kind: User` subjects.

### Step 3) Create a kubeconfig for a user (CSR flow)

#### 3a) Choose identity + output file

```bash
export USERNAME=wp-accessor
export GROUP=wp-accessors
export OUT=./.state/users/${NS}/${USERNAME}.kubeconfig
mkdir -p "$(dirname "$OUT")"
```

#### 3b) Generate key + CSR locally

```bash
WORKDIR="$(mktemp -d)"
trap 'rm -rf "$WORKDIR"' EXIT

openssl genrsa -out "$WORKDIR/client.key" 2048
openssl req -new \
  -key "$WORKDIR/client.key" \
  -out "$WORKDIR/client.csr" \
  -subj "/CN=${USERNAME}/O=${GROUP}"
```

#### 3c) Submit the CSR object to Kubernetes

```bash
CSR_B64="$(base64 < "$WORKDIR/client.csr" | tr -d '\n')"
CSR_NAME="csr-${USERNAME}-${NS}"

kubectl delete csr "$CSR_NAME" >/dev/null 2>&1 || true

kubectl apply -f - <<EOF
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: ${CSR_NAME}
spec:
  request: ${CSR_B64}
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF
```

#### 3d) Approve it (admin step)

```bash
kubectl certificate approve "$CSR_NAME"
kubectl get csr "$CSR_NAME"
```

#### 3e) Retrieve the signed certificate

```bash
CERT_B64=""
for i in $(seq 1 30); do
  CERT_B64="$(kubectl get csr "$CSR_NAME" -o jsonpath='{.status.certificate}' 2>/dev/null || true)"
  if [ -n "$CERT_B64" ]; then break; fi
  sleep 1
done

test -n "$CERT_B64" || { echo "CSR did not receive a certificate: $CSR_NAME" >&2; exit 1; }
echo "$CERT_B64" | base64 -d > "$WORKDIR/client.crt"
```

#### 3f) Build a kubeconfig (embedded certs)

```bash
CLUSTER_NAME="$(kubectl config view -o jsonpath='{.clusters[0].name}')"
SERVER="$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}')"
CA_DATA="$(kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')"

echo "$CA_DATA" | base64 -d > "$WORKDIR/ca.crt"

kubectl config --kubeconfig="$OUT" set-cluster "$CLUSTER_NAME" \
  --server="$SERVER" \
  --certificate-authority="$WORKDIR/ca.crt" \
  --embed-certs=true

kubectl config --kubeconfig="$OUT" set-credentials "$USERNAME" \
  --client-certificate="$WORKDIR/client.crt" \
  --client-key="$WORKDIR/client.key" \
  --embed-certs=true

kubectl config --kubeconfig="$OUT" set-context "${USERNAME}@${CLUSTER_NAME}" \
  --cluster="$CLUSTER_NAME" \
  --user="$USERNAME" \
  --namespace="$NS"

kubectl config --kubeconfig="$OUT" use-context "${USERNAME}@${CLUSTER_NAME}"
chmod 0600 "$OUT"

echo "Wrote kubeconfig: $OUT"
```

### Step 4) Test the kubeconfig

```bash
export KUBECONFIG="$OUT"

kubectl -n "$NS" get svc
kubectl -n "$NS" get pods
kubectl auth can-i create pods/portforward -n "$NS"

# should fail (accessor should NOT create deployments)
kubectl -n "$NS" create deployment nope --image=nginx
```

---

## Common operations

```bash
task                 # list tasks
task cluster:status  # check cluster status
task demo:info       # print URLs + token + /etc/hosts lines
task demo:uninstall  # uninstall demo apps (keep cluster)
task cluster:down    # delete everything (VMs + local state)
```

---

## Troubleshooting

### Hostnames do not resolve

Run:

```bash
task demo:info
```

Add the printed `cp-0 IP -> hostnames` mappings to `/etc/hosts`.

### CSR is approved but no certificate appears

```bash
export KUBECONFIG=./.state/kubeconfig
kubectl get csr
kubectl describe csr <name>
```

### PVCs stuck in Pending

```bash
export KUBECONFIG=./.state/kubeconfig
kubectl get sc
kubectl -n wordpress-demo get pvc
```

Ensure `local-path` is installed and set as the default StorageClass.

---

## Tradeoffs and next steps

### Tradeoffs in this approach

* **CSR-based users** are easy to demonstrate but operationally heavy at scale:

  * manual approvals (or additional automation)
  * certificate lifecycle/rotation and revocation complexity
* **Local-path storage** is perfect for demos but not HA and not suitable for production
* Cluster-scoped installs (CNI, ingress controller, dashboard) still require admin-level bootstrapping

### Recommended next steps (if hardening beyond a demo)

* Replace CSR users with **OIDC-based auth** (centralized identity, group claims)
* Add automated policy enforcement (Pod Security admission, network policies)
* Replace local-path storage with a proper CSI backend if moving beyond a single-machine demo
* Replace the stub API service with a real service and least-privilege DB access

---

## License / usage

This repository is intended as a learning/demo environment. Use it freely and adapt as needed.

