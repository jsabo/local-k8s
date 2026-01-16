# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Kubernetes demo cluster provisioning project that creates a 3-node kubeadm cluster on Multipass VMs with a full demo stack (Calico CNI, local-path storage, Headlamp UI, Calico Whisker UI, NGINX Gateway Fabric).

## Commands

All automation uses `task` (Taskfile runner):

```bash
# Main workflows
task demo:install      # Full one-command setup: cluster + storage + UIs + Gateway API
task cluster:down      # Delete VMs and local state
task cluster:purge     # Delete everything including Multipass cache
task demo:info         # Print URLs, tokens, /etc/hosts entries

# Cluster lifecycle
task cluster:up        # Create cluster only (no demo apps)
task cluster:status    # Show node status
task demo:uninstall    # Remove demo apps, keep cluster

# List all available tasks
task --list
```

Required dependencies: `multipass`, `kubectl`, `helm`, `task`, `envsubst`

## Architecture

**Cluster Topology:**
- 3-node kubeadm cluster: `cp-0` (control-plane), `worker-0`, `worker-1`
- Calico VXLAN overlay networking with NetworkPolicy support
- Rancher local-path-provisioner for dynamic local PVs

**Demo Stack:**
- **Headlamp**: Kubernetes web UI at `headlamp.local.k8s:30080`
- **Whisker**: Calico observability UI at `whisker.local.k8s:30080`
- **NGINX Gateway Fabric**: Gateway API implementation routing via NodePort

**Access Flow:**
```
Browser → /etc/hosts (*.local.k8s → VM IP) → NGF NodePort (30080)
       → Gateway + HTTPRoute → Headlamp/Whisker Service
```

## Code Structure

**Template-based configuration:**
- `.tftpl` files are Terraform templates rendered via `envsubst` to `.state/`
- `cloud-init.tftpl` → VM bootstrap configuration
- `kubeadm-config.tftpl` → kubeadm init configuration
- `gateway.tftpl` → Gateway API resources (Gateway + HTTPRoutes)

**State management:**
- `.state/` directory contains all rendered configs and state (git-ignored)
- Stamp files (`.state/.<component>.installed`) provide idempotency

**Taskfile.yaml:**
- Primary automation engine (~820 lines)
- Environment variables for all configurable values (versions, CIDR ranges, resource sizing)
- Internal tasks prefixed appropriately (`cluster:`, `storage:`, `apps:`, `gateway:`)

## Configuration

Override defaults via environment variables:

```bash
# Examples
K8S_VERSION_FULL=1.33.1 task demo:install
HEADLAMP_CLUSTERROLE=view HEADLAMP_TOKEN_DURATION=15m task demo:install
MP_CPUS=4 MP_MEM=8G task demo:install
```

Key variables: `K8S_VERSION_FULL`, `CALICO_VERSION`, `POD_CIDR`, `SERVICE_CIDR`, `EDGE_HOST`, `GATEWAY_NODEPORT_HTTP`
