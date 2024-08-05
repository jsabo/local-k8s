# local-k8s

## Deploy Local K8s cluster

#### Deploy local k8s with `Kind`

```
cat <<EOF | kind create cluster --name blue --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
networking:
  podSubnet: "10.5.0.0/24"

EOF
cat <<EOF | kind create cluster --name blue --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
networking:
  podSubnet: "10.6.0.0/24"
EOF
```

```
kubectl config use-context blue
kubectl config use-context green
```

OR 

#### Deploy local k8s with `Talos`

```
talosctl cluster create --name blue --cidr 10.5.0.0/24
talosctl cluster create --name green --cidr 10.6.0.0/24
```

```
kubectl config use-context admin@blue
kubectl config use-context admin@green
```

