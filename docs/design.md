# Design: kubeadm + Multipass demo cluster (WordPress + RBAC + CSR users + Dashboard + Ingress)

## Goals
- Build a **reproducible 3-node Kubernetes cluster** (1 control-plane, 2 workers) using **kubeadm** on **Multipass** VMs.
- Use **containerd** as the runtime and **Calico** for cluster networking.
- Deploy a **custom Helm chart** for **WordPress + MySQL** using **persistent volumes**.
- Demonstrate **least-privilege access** using **namespace RBAC** and **CSR-based** client certificate users:
  - **Deployer**: Helm lifecycle for the app in one namespace
  - **Accessor**: read + port-forward only
- Provide a **demo-friendly access path** (Ingress NodePort + optional port-forward) and basic visibility (Dashboard).

## Non-goals
- Production HA (multi-control-plane, etcd redundancy)
- Production-grade storage (replication, backups, CSI integration)
- Full identity integration (OIDC/SAML) or centralized user lifecycle management
- Tight hardening (PSA/Gatekeeper/Kyverno, full NetworkPolicy coverage, mTLS between services)

---

## Architecture summary

### Components
- **Multipass VMs**: `cp-0`, `worker-0`, `worker-1`
- **Kubernetes**: kubeadm-managed cluster, containerd runtime
- **CNI**: Calico (overlay networking)
- **Storage**: local-path-provisioner (dynamic PVs for demo)
- **Ingress**: ingress-nginx exposed via NodePort
- **Workloads**:
  - `charts/custom-wordpress/` → WordPress + MySQL + PVCs + Secrets
  - Optional `traefik/whoami` stub API service
- **Visibility**: Kubernetes Dashboard (read-only token for demo)

### Demo access model (hostnames)
Instead of path-based routing on a single host, the demo uses **hostname-per-app**:

- `wp.<base>` → WordPress
- `api.<base>` → stub API
- `dashboard.<base>` → Kubernetes Dashboard (HTTPS backend)

The base domain defaults to `demo.test`, so the default demo hosts are:

- `wp.demo.test`
- `api.demo.test`
- `dashboard.demo.test`

You map these to the `cp-0` IP in `/etc/hosts`.

---

## Networking and cluster layout

### Networking assumptions (typical kubeadm defaults unless overridden)
- **Pod CIDR**: `10.48.0.0/16`
- **Service CIDR**: `10.49.0.0/16`
- **Overlay**: Calico **VXLAN** encapsulation for cross-node pod traffic
- **Service routing**: `kube-proxy` programs iptables/IPVS for ClusterIP/NodePort
- **Policy**: `calico-node` enforces NetworkPolicy (when defined)

### 3-node cluster networking diagram

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
````

---

## Ingress design

### Why hostname-per-app

Host-based routing simplifies the Dashboard case (no path rewrites) and makes each app’s URL “clean”:

* WordPress: `http://wp.demo.test:30080/`
* API stub: `http://api.demo.test:30080/`
* Dashboard: `https://dashboard.demo.test:30443/`

Ingress objects are created by the Taskfile (not by the WordPress Helm chart), so the Helm chart can remain “generic” and ingress-disabled by default.

### Ingress-NGINX exposure

Ingress-NGINX is installed via Helm and exposed via **NodePort**:

* HTTP NodePort: `30080`
* HTTPS NodePort: `30443`

### Host routing

Ingress rules route by **Host header**:

* `wp.<base>` → Service `custom-wordpress` in namespace `wordpress-demo`
* `api.<base>` → Service `goapi` in namespace `wordpress-demo`
* `dashboard.<base>` → the Dashboard Service in namespace `kubernetes-dashboard` (HTTPS backend)

The dashboard ingress uses:

* `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"`

No regex paths or rewrite targets are required.

---

## Security model (how access is controlled)

This repo intentionally demonstrates **minimum-privilege access** using a clear separation of duties.

### Who does what (boundaries)

* **Admin (bootstrap-only)**:

  * kubeadm init/join
  * install Calico, storage, ingress, dashboard
  * approve CSRs
* **Deployer (namespace-only)**:

  * install/upgrade/uninstall WordPress via Helm
  * manage namespaced objects needed by the release
* **Accessor (namespace-only)**:

  * read minimal resources + port-forward
  * cannot deploy/modify workloads

---

## Tradeoffs and notes

* Hostname-per-app requires either:

  * `/etc/hosts` entries (demo-friendly), or
  * real DNS / wildcard DNS (not required for this repo)
* CSR-based users are simple for demos but not scalable for real orgs without automation.
* local-path-provisioner is intentionally non-HA, suitable for demo clusters only.

